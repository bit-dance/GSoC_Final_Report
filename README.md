# **GSoC_Final_Report**
This repo is for saving my work during GSoC and make records for some undone works.
First of all, I want to thank my mentor **@lxc** . She gave me lots of help and advices.
And I also want to thank Curve community for their attention on open source and help fresh men.

## **Introduction**
Curve has a NameServer which manages namespace metadata information and includes FileInfo,PageFileSegment,PageFileChunkInfo components. And we could find it is easy to show the file information between directories and files in KV storage and just like this:

**![](https://lh5.googleusercontent.com/wxObQbxHSjPuRPzuE1qGzctCCbUpjZnyWFYKGA1aauFjihrNHOFEYkwlQ3ZuOlDBczKDkJCv7BTaTBx3qJz0N3fr_TCNRZEdxEG-JRn3ceqq3THcoMa0rQ5Uyj9G_d1m_01y_D2W6xMTaTGQvE5H6-U)**

It can be seen that metadata storage is very important for Curve. Only by ensuring the accuracy and high availability of metadata, and supporting rapid search and update, can users be assured of deployment. With the application of large-scale machine learning, big data analysis, and enterprise level data lakes, the data scale of distributed file systems has changed from PB level to EB level. Currently, most distributed file systems (such as curve) are facing the challenge of metadata scalability. And thus due to the storage engine etcd backend of curve has limited scalability, and the amount of metadata that can be stored is limited. In order to give more options to support larger clusters, it is hoped that the metadata can add a sql database like MySQL, MariaDB, PostgreSQL, etc. as one of the storage engines which could be selected by users.

Therefore, my goal is to add a new storage engine MySQL for Curve, and support the functions such as leader election and id_generator that were originally responsible for by etcd, and keep the high performance of file lookup or directory rename functions in MySQL storage engine so that metadata can be smoothly migrated from etcd to MySQL database.

## **What I have done**
### **1. Add MySQL storage engine for Curve**
This part i have add a new_local_repository in workspace which used for loading MySQL client from loacl and avoid open source license problem. And then i add some configuration items in conf/mds.conf to support MySQL storage engine. 
And to ensure consistency with the original KV storage interface,i abstract a storage class in include/etcdclient/storageclient.h and later all the storage engine could inherit from it. And then i add a mysqlclient class in src/mds/storageclient/mysqlclient.h which is used for connecting MySQL database and implement the interface in storageclient.h. 
So let's see the interface in storageclient.h:
```cpp
virtual int Put(const std::string &key, const std::string &value) = 0;
virtual int Get(const std::string &key, std::string *out) = 0;
virtual int List(const std::string &startKey, const std::string &endKey,
        std::vector<std::string> *values) = 0;
virtual int Delete(const std::string &key) = 0;
```
These are some basic required functions and i just use the original parameter and restore them directly in MySQL database. Of course, we need create a table for storing metadata in MySQL database. And the table structure is like this:
```sql
 "CREATE TABLE IF NOT EXISTS " + tableName +
                          "("
                          "storekey VARCHAR(255) NOT NULL,"
                          "storevalue LONGBLOB NOT NULL,"
                          "revision BIGINT NOT NULL"
                          ")";
```
And then just config your ip, port, username, password, database name and table name in mds.conf. And then you can use these basic functions in Curve.Here i connect MySQL like this :
```cpp
int MysqlClientImp::Init(MysqlConf conf, int timeout, int retryTiems) {
    try {
        sql::Driver *driver = sql::MySQL::get_driver_instance();
        conn_ = std::shared_ptr<sql::Connection>(driver->connect(conf.host_, conf.user_, conf.passwd_));
        is_connected_.store(true);
        conn_->setSchema(conf.db_);
        stmt_ = conn_->createStatement();
        conf_ = conf;//deep copy
        driver_ = driver;

    } catch (sql::SQLException &e) {
        LOG(ERROR) << "MysqlClientImp Init failed, error: " << e.what();
        return -1;
    }
    return 0;
}
```
We would better use a try-catch block to catch the exception to avoid the connection lost.
And when we close the client we should close the connection and statement to avoid memory leak.
And for supporting TXN here i refer to ETCD's support for txn and use the Revision field to simulate multi version control.And some core functions here:
https://github.com/opencurve/curve/pull/2527/files#diff-6efcb07f8af51c8ffd06bddb68327357f77d9b78e17a37b2b6a6f554bc50eb06R99 

And then i use MySQL auto_commit to ensure the atomicity of the transaction. And here is the core code:
```cpp
conn_->setAutoCommit(false);

some operations...

if(some operations failed){
    conn_->rollback();
}
else{
    conn_->commit();
}

conn_->setAutoCommit(true);
```
And then i add some unit tests for MySQL storage engine in test/mysqlstorageclient/mysqlclient_test.cpp.

### **2. Support leader election**

In order to ensure the high availability of metadata, we need to support leader election. And here i choose to create a table in MySQL database to store the leader information. And the table structure is like this:
```sql
"CREATE TABLE IF NOT EXISTS leader_election ("
              "elect_key varchar(255) NOT NULL ,"
              "leader_name varchar(255) NOT NULL ,"
              "create_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ,"
              "update_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP , "
              "leader_id bigint AUTO_INCREMENT,"
              "PRIMARY KEY (leader_id),"
              "UNIQUE KEY (elect_key)"
              ")";
```
And you can see my solution is composed of a single table “leader_election” and CAS algorithm implemented using MySQL version number. 

Assuming that there is a version field in a table, and each update requires adding one to the version field, each service will obtain its own version number before the update. If everyone is the same, then all services want to add one to the version number during the update, but MySQL ensures that only one service can update the version number at the same time, that is, obtain a larger version number, and then the remaining services can determine their leader.

Based on this idea,we could define some rules for election of this method:

1.The initial version number is 0. During service instances initialization, all want to update the election version number, but only one will get success. The winner becomes the leader.

2.The leader regularly sends a heartbeat to MySQL and show it is alive.

3.The follower periodically detects the last heartbeat time of the leader. If it exceeds the time limit (such as 5s), it is considered that the leader is down.

4.The follower detected that the leader's heartbeat was interrupted and think the leader is down.

5.The followers will get the last version before they think the leader is down and start a new round for election just like above.

And some detailed code here:
https://github.com/opencurve/curve/pull/2527/files#diff-6efcb07f8af51c8ffd06bddb68327357f77d9b78e17a37b2b6a6f554bc50eb06R429

When i choose a leader , there will only one mds node succeed and others will fail. And then it should send hearbeat to other followers to show it is alive. And when the leader is down, the followers will start a new round for election. And here i use a thread to detect the leader's heartbeat and if it is down, the follower will start a new round for election. And here is the core code:
```cpp
Thread t([this](){
            std::shared_ptr<sql::Connection> conn3_;
            // sql::Driver *driver = sql::mysql::get_driver_instance();
            conn3_ = std::shared_ptr<sql::Connection>(
                this->driver_->connect(this->conf_.host_, this->conf_.user_, this->conf_.passwd_));
            conn3_->setSchema(this->conf_.db_);
            while(1){
            this->conn_mutex_.lock();
            if(this->conn_==nullptr)
            {
                LOG(ERROR) << "conn_ is nullptr";
                conn3_->close();
                conn3_=nullptr;
            }
            this->conn_mutex_.unlock();
            if (conn3_ == nullptr) {
                    LOG(ERROR) << "conn3_ is nullptr";
                    return;
            }
            try {
                std::string sql =
                    "UPDATE leader_election SET update_time = NOW() "
                    "WHERE leader_id = " +
                    std::to_string(leaderOid_) + ";";

                sql::Statement * stmt_ = conn3_->createStatement();
                stmt_->executeUpdate(sql);
                LOG(INFO) << "update leader_election success";
            } catch (sql::SQLException &e) {
                LOG(ERROR) << "update leader_election failed, error: "
                           << e.what();
                return;
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(300));
            }
        });
        t.detach();
```
You can see we must use a new connection to avoid the connection lost. And then we use a lock to sync conn state and use a while loop to send heartbeat to MySQL database. And here i use a sleep function to control the frequency of heartbeat. And then we use a try-catch block to catch the exception to avoid the connection lost. 

As for leader observe we just need to detect the leader timestamp and this is achieved by the upper functions so we just check the timestamp.

And for the leader resign we could delete the original key and add a new key to start a new round for election. And here is the core code:
```cpp
try {
        std::string sql = "DELETE FROM leader_election WHERE leader_id = " + std::to_string(leaderOid) + ";";
        stmt_->execute(sql);
    } catch (sql::SQLException &e) {
        LOG(ERROR) << "MysqlClientImp LeaderResign failed, error: " << e.what();
        return -1;
    }

    return EtcdErrCode::EtcdLeaderResiginSuccess;
```
And also i add some unit tests for leader election in test/mysqlstorageclient/mysqlclient_test.cpp.

## **Some future tasks**
### **1. Support id_generator**
In order to ensure the uniqueness of the file id, we need to support id_generator. And here just replace some interface and we can use Compare And Swap algorithm to ensure the uniqueness of the file id. 

### **2.Test leader election**
We must test the leader election in a large cluster and ensure the correctness of the algorithm. And we should also test the performance of the leader election and ensure the high availability of metadata.

### **3. Build a cluster to test storage**
Here i made lots of affort to build a cluster to test the storage engine but i failed. Most of the time i spent on building the cluster and i have no time to test the storage engine. So i think it is very important to build a cluster to test the storage engine and ensure the correctness of the storage engine.

## **Summary**
In this GSoC,I have learned a lot, especially in building the environment. I have learned to read scripts myself, push images, compile using bazel, and run individual tests using bazel. I have also learned how to collaborate among communities and strengthened my confidence in participating in open source. I will continue to complete tasks and participate in the curve community.





