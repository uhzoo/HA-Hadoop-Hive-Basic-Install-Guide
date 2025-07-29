# HA-Hadoop-Hive-Basic-Install
Step-by-step guide for installing a basic High Availability (HA) Hadoop cluster with Hive.

| Service  | Version |
| ------------- | ------------- |
|OS|Rocky Linux 9.4 Minimal, 2 vCPUs, 4 GB RAM|
|Hadoop|3.4.1|
|ZooKeeper|3.8.4|
|Hive|4.0.1|
|Hive MetaStore|PostgreSQL 16|

The following guide uses `test1.hadoop.com`, `test2.hadoop.com`, `test3.hadoop.com` in example URLs.  
To use them as-is, update your hosts file to map each hostname to the corresponding server IP.
- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Mac/Linux: `/etc/hosts`

Alternatively, you can access services directly using `YOUR_IP:PORT`.

The following table shows how services are distributed across the three servers.

|test1.hadoop.com|test2.hadoop.com|test3.hadoop.com|
| ------------- | ------------- | ------------- |
|ZooKeeper|ZooKeeper|ZooKeeper|
|JournalNode|JournalNode|JournalNode|
|DataNode|NameNode|NameNode|
|NodeManager|ResourceManager|ResourceManager|
|JobHistoryServer|DataNode|DataNode|
|Hive|NodeManager|NodeManager|

## Install HA Hadoop
Before installing Hadoop, disable the firewall on all servers for convenience.
```bash
# Run on all server as root
systemctl disable firewalld --now;
```
Hadoop requires Java (JDK 8), so install it on all servers first.
```bash
# Run on all server as root
dnf install java-1.8.0-*;
```
To download and extract the installation files, install wget and tar.
```bash
# Run on test1.hadoop.com as root
dnf install wget tar;
```
Configure `/etc/hosts` and set the hostname on each server to enable Hadoop to use fully qualified domain names (FQDN).
```bash
# Run on test1.hadoop.com as root
echo YOUR_TEST1_IP test1.hadoop.com >> /etc/hosts;
echo YOUR_TEST2_IP test2.hadoop.com >> /etc/hosts;
echo YOUR_TEST3_IP test3.hadoop.com >> /etc/hosts;
```
```bash
# Run on test1.hadoop.com as root
hostnamectl set-hostname test1.hadoop.com
```
```bash
# Run on test2.hadoop.com as root
hostnamectl set-hostname test2.hadoop.com
```
```bash
# Run on test3.hadoop.com as root
hostnamectl set-hostname test3.hadoop.com
```
Add a user named hadoop and set its password (in this example, the password is also hadoop).
```bash
# Run on all server as root
useradd hadoop;
passwd hadoop;
```
We will store all Hadoop-related data under `/data`, so let's create the directory and give ownership to the hadoop user.
```bash
# Run on test1.hadoop.com as root
mkdir -p /data;
chown -R hadoop. /data;
```
### Install ZooKeeper.
Download Zookeeper, extract it to `/opt`, and change the ownership to the hadoop user.
```bash
# Run on test1.hadoop.com as root
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.4/apache-zookeeper-3.8.4-bin.tar.gz;
tar -zxf apache-zookeeper-3.8.4-bin.tar.gz -C /opt;
chown -R hadoop. /opt/apache-zookeeper-3.8.4-bin;
```
Copy the default configuration file and modify it to set the data directory and server list for your cluster.
```bash
# Run on test1.hadoop.com as hadoop
cp /opt/apache-zookeeper-3.8.4-bin/conf/zoo_sample.cfg /opt/apache-zookeeper-3.8.4-bin/conf/zoo.cfg;
```
```bash
# Run on test1.hadoop.com as hadoop
vi /opt/apache-zookeeper-3.8.4-bin/conf/zoo.cfg;
```
```bash
#dataDir=/tmp/zookeeper
dataDir=/data/zookeeper
server.1=test1.hadoop.com:2888:3888
server.2=test2.hadoop.com:2888:3888
server.3=test3.hadoop.com:2888:3888
admin.enableServer=true
admin.serverPort=8080
admin.serverAddress=0.0.0.0
```
Create the ZooKeeper data directory.
```bash
# Run on test1.hadoop.com as hadoop
mkdir /data/zookeeper;
```
Create the myid file inside the data directory.
```bash
# Run on test1.hadoop.com as hadoop
touch /data/zookeeper/myid;
```
Send `/etc/hosts`, the ZooKeeper installation directory, and the ZooKeeper data directory to the other servers.
You can also manually configure `/etc/hosts` on each server instead of using scp.
```bash
# Run on test1.hadoop.com as root
scp /etc/hosts test2.hadoop.com:/etc;
scp -r /opt/apache-zookeeper-3.8.4-bin test2.hadoop.com:/opt;
echo 2 > /data/zookeeper/myid;
scp -r /data test2.hadoop.com:/;
scp /etc/hosts test3.hadoop.com:/etc;
scp -r /opt/apache-zookeeper-3.8.4-bin test3.hadoop.com:/opt;
echo 3 > /data/zookeeper/myid;
scp -r /data test3.hadoop.com:/;
echo 1 > /data/zookeeper/myid;
```
Change the ownership of `/opt/apache-zookeeper-3.8.4-bin` and `/data` to the hadoop user.
```bash
# Run on test2.hadoop.com, test3.hadoop.com as root
chown -R hadoop. /opt/apache-zookeeper-3.8.4-bin;
chown -R hadoop. /data;
```
Start ZooKeeper on all servers.
```bash
# Run on all server as hadoop
/opt/apache-zookeeper-3.8.4-bin/bin/zkServer.sh start
```
If you see the following output from `jps -m` on all server, it indicates that ZooKeeper service is running correctly.
```bash
# Run on all server as hadoop
jps -m;
```
```text
29150 QuorumPeerMain /opt/apache-zookeeper-3.8.4-bin/bin/../conf/zoo.cfg
```
You can access the Web UI of each ZooKeeper service using the following URLs.

| Service  | URL |
| ------------- | ------------- |
|ZooKeeper|http://test1.hadoop.com:8080/commands|
|ZooKeeper|http://test2.hadoop.com:8080/commands|
|ZooKeeper|http://test3.hadoop.com:8080/commands|

### Install Hadoop.
Download Hadoop, extract it to `/opt`, and change the ownership to the hadoop user.
```bash
# Run on test1.hadoop.com as root
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz;
tar -zxf hadoop-3.4.1.tar.gz -C /opt;
chown -R hadoop. /opt/hadoop-3.4.1;
```
In order to use commands like start-all.sh and enable communication between nodes, passwordless SSH access is required.
Generate an SSH key and copy it to all servers using `ssh-copy-id`.
When prompted during `ssh-keygen`, simply press Enter to accept the default settings.
```bash
# Run on test1.hadoop.com as hadoop
ssh-keygen -t rsa -m PEM;
ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub test1.hadoop.com;
ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub test2.hadoop.com;
ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub test3.hadoop.com;
```
Now configure the `HADOOP_HOME` and `JAVA_HOME` environment variables in `/opt/hadoop-3.4.1/etc/hadoop/hadoop-env.sh`.
You can determine your `JAVA_HOME` path with the following command: `readlink -f /usr/bin/java`
```bash
# Run on test1.hadoop.com as hadoop
vi /opt/hadoop-3.4.1/etc/hadoop/hadoop-env.sh;
```
```text
# export JAVA_HOME=
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.el9.x86_64/jre
# export HADOOP_HOME=
export HADOOP_HOME=/opt/hadoop-3.4.1
# export HADOOP_HEAPSIZE_MAX=
export HADOOP_HEAPSIZE_MAX=512
# export HADOOP_HEAPSIZE_MIN=
export HADOOP_HEAPSIZE_MIN=512
```
Next, configure the following Hadoop XML files to set up the core components.
1. core-site.xml
```bash
# Run on test1.hadoop.com as hadoop
vi /opt/hadoop-3.4.1/etc/hadoop/core-site.xml;
```
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop-cluster</value>
        <description>The default filesystem URI. All HDFS paths will be resolved relative to this.</description>
    </property>
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>test1.hadoop.com:2181,test2.hadoop.com:2181,test3.hadoop.com:2181</value>
        <description></description>
    </property>
</configuration>
```
2. hdfs-site.xml
```bash
# Run on test1.hadoop.com as hadoop
vi /opt/hadoop-3.4.1/etc/hadoop/hdfs-site.xml;
```
```xml
<configuration>
    <property>
        <name>dfs.nameservices</name>
        <value>hadoop-cluster</value>
        <description></description>
    </property>
    <property>
        <name>dfs.ha.namenodes.hadoop-cluster</name>
        <value>nn1,nn2</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.hadoop-cluster.nn1</name>
        <value>test2.hadoop.com:8020</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.http-address.hadoop-cluster.nn1</name>
        <value>test2.hadoop.com:9870</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.hadoop-cluster.nn2</name>
        <value>test3.hadoop.com:8020</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.http-address.hadoop-cluster.nn2</name>
        <value>test3.hadoop.com:9870</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.rpc-bind-host</name>
        <value>0.0.0.0</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://test1.hadoop.com:8485;test2.hadoop.com:8485;test3.hadoop.com:8485/hadoop-cluster</value>
        <description></description>
    </property>
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/data/journalnode</value>
        <description></description>
    </property>
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
        <description></description>
    </property>
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>shell(/bin/true)</value>
        <description></description>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.hadoop-cluster</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
        <description></description>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/data/namenode</value>
        <description></description>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/data/datanode</value>
        <description></description>
    </property>
    <property>
        <name>dfs.datanode.address</name>
        <value>0.0.0.0:9866</value>
        <description></description>
    </property>
    <property>
        <name>dfs.datanode.http.address</name>
        <value>0.0.0.0:9864</value>
        <description></description>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
        <description>The default number of data block replications. Set to 1 for single-node or test environments.</description>
    </property>
</configuration>
```
3. yarn-site.xml
```bash
# Run on test1.hadoop.com as hadoop
vi /opt/hadoop-3.4.1/etc/hadoop/yarn-site.xml;
```
```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>yarn-cluster</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>test2.hadoop.com</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm1</name>
        <value>test2.hadoop.com:8088</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>test3.hadoop.com</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm2</name>
        <value>test3.hadoop.com:8088</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.store.class</name>
        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
        <description></description>
    </property>
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>test1.hadoop.com:2181,test2.hadoop.com:2181,test3.hadoop.com:2181</value>
        <description></description>
    </property>
    <property>
        <name>yarn.client.failover-proxy-provider.yarn-cluster</name>
        <value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
        <description></description>
    </property>
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>file:/data/yarn/local</value>
        <description>List of directories to store localized files in. An application's localized file directory will be found in: ${yarn.nodemanager.local-dirs}/usercache/${user}/appcache/application_${appid}. Individual containers' work directories, called container_${contid}, will be subdirectories of this.</description>
    </property>
    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>file:/data/yarn/logs</value>
        <description>Where to store container logs. An application's localized log directory will be found in ${yarn.nodemanager.log-dirs}/application_${appid}. Individual containers' log directories will be below this, in directories named container_{$contid}. Each container directory will contain the files stderr, stdin, and syslog generated by that container.</description>
    </property>
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>2048</value>
        <description>Amount of physical memory, in MB, that can be allocated for containers.</description>
    </property>
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>2</value>
        <description>Number of vcores that can be allocated for containers. This is used by the RM scheduler when allocating resources for containers. This is not used to limit the number of physical cores used by YARN containers.</description>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>2048</value>
        <description>The maximum allocation for every container request at the RM, in MBs. Memory requests higher than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>2</value>
        <description>The maximum allocation for every container request at the RM, in terms of virtual CPU cores. Requests higher than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>512</value>
        <description>The minimum allocation for every container request at the RM, in MBs. Memory requests lower than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value>
        <description>The minimum allocation for every container request at the RM, in terms of virtual CPU cores. Requests lower than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
        <description>Enables shuffle service for MapReduce jobs in the NodeManager.</description>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
        <description>Specifies which environment variables are passed from NodeManager to containers.</description>
    </property>
</configuration>
```
4. mapred-site.xml
```bash
# Run on test1.hadoop.com as hadoop
vi /opt/hadoop-3.4.1/etc/hadoop/mapred-site.xml;
```
```xml
<configuration>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>test1.hadoop.com:10020</value>
        <description>MapReduce JobHistory Server IPC host:port</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>test1.hadoop.com:19888</value>
        <description>MapReduce JobHistory Server Web UI host:port</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>/mr-history/done</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>/mr-history/tmp</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.map.memory.mb</name>
        <value>512</value>
        <description>The amount of memory to request from the scheduler for each map task.</description>
    </property>
    <property>
        <name>mapreduce.map.java.opts</name>
        <value>-Xmx400m</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>512</value>
        <description>The amount of memory to request from the scheduler for each reduce task.</description>
    </property>
    <property>
        <name>mapreduce.reduce.java.opts</name>
        <value>-Xmx400m</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
        <description>The runtime framework for executing MapReduce jobs. Can be one of local, classic or yarn.</description>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
        <description>CLASSPATH for MR applications. A comma-separated list of CLASSPATH entries. If mapreduce.application.framework is set then this must specify the appropriate classpath for that archive, and the name of the archive must be present in the classpath. If mapreduce.app-submission.cross-platform is false, platform-specific environment vairable expansion syntax would be used to construct the default CLASSPATH entries. For Linux: $HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*, $HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*. For Windows: %HADOOP_MAPRED_HOME%/share/hadoop/mapreduce/*, %HADOOP_MAPRED_HOME%/share/hadoop/mapreduce/lib/*. If mapreduce.app-submission.cross-platform is true, platform-agnostic default CLASSPATH for MR applications would be used: {{HADOOP_MAPRED_HOME}}/share/hadoop/mapreduce/*, {{HADOOP_MAPRED_HOME}}/share/hadoop/mapreduce/lib/* Parameter expansion marker will be replaced by NodeManager on container launch based on the underlying OS accordingly.</description>
    </property>
</configuration>
```
Create the workers file to specify the hosts where DataNodes and NodeManagers will run.
```bash
# Run on test1.hadoop.com as hadoop
cat /etc/hosts | grep hadoop | awk '{print $2}' > /opt/hadoop-3.4.1/etc/hadoop/workers
```
After configuring the Hadoop files, copy the Hadoop directory to the other servers.
```bash
# Run on test1.hadoop.com as root
scp -r /opt/hadoop-3.4.1 test2.hadoop.com:/opt;
scp -r /opt/hadoop-3.4.1 test3.hadoop.com:/opt;
```
Change the ownership of `/opt/hadoop-3.4.1` to the hadoop user.
```bash
# Run on test2.hadoop.com, test3.hadoop.com as root
chown -R hadoop. /opt/hadoop-3.4.1;
```
We will now start Hadoop.
Make sure you’re on the correct server before running each command.
```bash
# Run on all server as hadoop
/opt/hadoop-3.4.1/bin/hdfs --daemon start journalnode;
```
```bash
# Run on nn1 Server as hadoop. Here test2.hadoop.com is nn1.
/opt/hadoop-3.4.1/bin/hdfs namenode -format;
/opt/hadoop-3.4.1/bin/hdfs --daemon start namenode;
```
```bash
# Run on nn2 Server as hadoop. Here test3.hadoop.com is nn2.
/opt/hadoop-3.4.1/bin/hdfs namenode -bootstrapStandby;
/opt/hadoop-3.4.1/bin/hdfs --daemon start namenode;
```
```bash
# Run on nn1 Server as hadoop. Here test2.hadoop.com is nn1.
/opt/hadoop-3.4.1/bin/hdfs zkfc -formatZK
```
```bash
# Run on nn1, nn2 Server as hadoop. Here test2.hadoop.com is nn1, test3.hadoop.com is nn2.
/opt/hadoop-3.4.1/bin/hdfs --daemon start zkfc
```
```bash
# Run on all server as hadoop
/opt/hadoop-3.4.1/bin/hdfs --daemon start datanode
```
```bash
# Run on rm1, rm2 Server as hadoop. Here test2.hadoop.com is rm1, test3.hadoop.com is rm2.
/opt/hadoop-3.4.1/bin/yarn --daemon start resourcemanager
```
```bash
# Run on all server as hadoop
/opt/hadoop-3.4.1/bin/yarn --daemon start nodemanager
```
```bash
# Run on test1.hadoop.com as hadoop
/opt/hadoop-3.4.1/bin/mapred --daemon start historyserver
```
Run the following command on each server to confirm that all required Hadoop and ZooKeeper services are running.
If you see outputs similar to the ones below, it indicates that Hadoop services are running correctly.
```bash
# Run on test1.hadoop.com as hadoop
jps -m;
```
```text
17125 JournalNode
17493 Jps -m
17208 DataNode
16957 QuorumPeerMain /opt/apache-zookeeper-3.8.4-bin/bin/../conf/zoo.cfg
17311 NodeManager
17375 JobHistoryServer
```
```bash
# Run on test2.hadoop.com as hadoop
jps -m;
```
```text
17808 QuorumPeerMain /opt/apache-zookeeper-3.8.4-bin/bin/../conf/zoo.cfg
18000 JournalNode
18563 ResourceManager
18135 NameNode
18935 Jps -m
18365 DFSZKFailoverController
18463 DataNode
18831 NodeManager
```
```bash
# Run on test3.hadoop.com as hadoop
jps -m;
```
```text
17200 Jps -m
16417 QuorumPeerMain /opt/apache-zookeeper-3.8.4-bin/bin/../conf/zoo.cfg
16577 JournalNode
16897 DataNode
16834 DFSZKFailoverController
16707 NameNode
16997 ResourceManager
17096 NodeManager
```
You can access the Web UIs of each Hadoop service using the following URLs.

| Service  | URL |
| ------------- | ------------- |
| NameNode | http://test2.hadoop.com:9870 |
| NameNode | http://test3.hadoop.com:9870 |
| DataNode | http://test1.hadoop.com:9864 |
| DataNode | http://test2.hadoop.com:9864 |
| DataNode | http://test3.hadoop.com:9864 |
| ResourceManager | http://test2.hadoop.com:8088 |
| ResourceManager | http://test3.hadoop.com:8088 |
| NodeManager | http://test1.hadoop.com:8042 |
| NodeManager | http://test2.hadoop.com:8042 |
| NodeManager | http://test3.hadoop.com:8042 |
| JobHistoryServer | http://test1.hadoop.com:19888 |

After creating the required HDFS directories for MapReduce, stop all Hadoop and ZooKeeper services before proceeding with the Hive installation.
```bash
# Run on test1.hadoop.com as hadoop
/opt/hadoop-3.4.1/bin/hdfs dfs -chmod -R 1777 /mr-history;
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir -p /tmp/hadoop-yarn/staging;
/opt/hadoop-3.4.1/bin/hdfs dfs -chmod -R 1777 /tmp;
/opt/hadoop-3.4.1/bin/mapred --daemon stop historyserver;
/opt/hadoop-3.4.1/sbin/stop-all.sh;
```
```bash
# Run on all server as hadoop
/opt/apache-zookeeper-3.8.4-bin/bin/zkServer.sh stop
```
---
## Install Hive
In this step, we’ll install Hive and configure PostgreSQL 16 as its metastore database. Use the following commands to install PostgreSQL 16 on your system via `dnf`.
```bash
dnf install https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm;
dnf install https://rpmfind.net/linux/centos-stream/9-stream/CRB/x86_64/os/Packages/perl-IO-Tty-1.16-4.el9.x86_64.rpm;
dnf install https://rpmfind.net/linux/centos-stream/9-stream/CRB/aarch64/os/Packages/perl-IPC-Run-20200505.0-6.el9.noarch.rpm;
dnf install postgresql16 postgresql16-devel postgresql16-server;
```
PostgreSQL 16 requires an initial database setup before it can be started.
```bash
postgresql-16-setup initdb;
```
To allow both local and remote connections to PostgreSQL, set `listen_addresses = '*'` in `/var/lib/pgsql/16/data/postgresql.conf`, and modify `/var/lib/pgsql/16/data/pg_hba.conf` to allow remote client access.
```bash
vi /var/lib/pgsql/16/data/postgresql.conf;
```
```text
# listen_addresses = '*'          # what IP address(es) to listen on;
listen_addresses = '*'          # what IP address(es) to listen on;
```
This change allows PostgreSQL to accept connections from all IP addresses.
```bash
vi /var/lib/pgsql/16/data/pg_hba.conf;
```
```text
# host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             YOUR_TEST1_IP/24            scram-sha-256
```
This rule allows clients in the `${YOUR_IP}/24` subnet to connect to any database as any user using scram-sha-256 authentication.

Let’s start PostgreSQL 16 and connect to it.
```bash
systemctl start postgresql-16;
su - postgres -c "psql";
```
Now let’s create the Hive user and database.
```sql
CREATE USER hive WITH PASSWORD 'hive';
CREATE DATABASE hive OWNER hive;
\q
```
The Hive metastore is now ready to be installed. Let’s create the hive user and add it to the hadoop group
```bash
useradd -g hadoop hive;
```
Download Hive and the PostgreSQL JDBC driver
```bash
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz;
wget https://repo1.maven.org/maven2/org/postgresql/postgresql/42.5.4/postgresql-42.5.4.jar;
```
Extract Hive to the `/opt` directory, and move the JDBC driver into Hive’s lib directory so it can connect to the PostgreSQL metastore.
```bash
tar -zxf apache-hive-4.0.1-bin.tar.gz -C /opt;
mv postgresql-42.5.4.jar /opt/apache-hive-4.0.1-bin/lib;
chown -R hive. /opt/apache-hive-4.0.1-bin;
```
Now log in as the hive user, create the `hive-env.sh` file from the provided template, and configure the `HADOOP_HOME` environment variable. Also, set conditional logging configuration for the metastore and HiveServer2 based on the service type.
```bash
su - hive;
```
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-env.sh.template /opt/apache-hive-4.0.1-bin/conf/hive-env.sh;
```
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-env.sh;
```
```text
# HADOOP_HOME=${bin}/../../hadoop
export HADOOP_HOME=/opt/hadoop-3.4.1

export HADOOP_OPTS="$HADOOP_OPTS -Dhive.log.dir=$HIVE_HOME/logs"
if [[ "$SERVICE" == "metastore" ]]; then
   export HIVE_LOG4J_FILE=$HIVE_HOME/conf/hive-metastore-log4j2.properties
   export HADOOP_OPTS="$HADOOP_OPTS -Dlog4j2.configurationFile=$HIVE_LOG4J_FILE"
elif [[ "$SERVICE" == "hiveserver2" ]]; then
   export HIVE_LOG4J_FILE=$HIVE_HOME/conf/hive-server2-log4j2.properties
   export HADOOP_OPTS="$HADOOP_OPTS -Dlog4j2.configurationFile=$HIVE_LOG4J_FILE"
fi
```
Create `/opt/apache-hive-4.0.1-bin/conf/hive-site.xml` and set the configuration variables according to your environment.
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-site.xml;
```
```xml
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://test1.hadoop.com:5432/hive?createDatabaseIfNotExist=true</value>
        <description>
          JDBC connect string for a JDBC metastore.
          To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
          For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
        </description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
        <description>Driver class name for a JDBC metastore</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
        <description>Username to use against metastore database</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hive</value>
        <description>password to use against metastore database</description>
    </property>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
        <description>location of default database for the warehouse</description>
    </property>
    <property>
        <name>hive.server2.enable.doAs</name>
        <value>true</value>
        <description>
          Setting this property to true will have HiveServer2 execute
          Hive operations as the user making the calls to it.
        </description>
    </property>
    <property>
        <name>hive.server2.webui.host</name>
        <value>test1.hadoop.com</value>
        <description>The host address the HiveServer2 WebUI will listen on</description>
    </property>
    <property>
        <name>hive.server2.webui.port</name>
        <value>10002</value>
        <description>The port the HiveServer2 WebUI will listen on. This can beset to 0 or a negative integer to disable the web UI</description>
    </property>
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>test1.hadoop.com</value>
        <description>Bind host on which to run the HiveServer2 Thrift service.</description>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
        <description>Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'binary'.</description>
    </property>
    <property>
        <name>hive.exec.scratchdir</name>
        <value>/tmp/hive</value>
        <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: ${hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.</description>
    </property>
</configuration>
```
For convenience, create `/opt/apache-hive-4.0.1-bin/conf/beeline-hs2-connection.xml` to preconfigure Beeline login credentials.
```bash
vi /opt/apache-hive-4.0.1-bin/conf/beeline-hs2-connection.xml;
```
```xml
<configuration>
    <property>
        <name>beeline.hs2.connection.user</name>
        <value>hive</value>
        <description></description>
    </property>
    <property>
        <name>beeline.hs2.connection.password</name>
        <value>hive</value>
        <description></description>
    </property>
</configuration>
```
Create the following three configuration files from `/opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template` for logging purposes. For each file, update the log directory and file name appropriately.
1. hive-log4j2.properties
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties;
```
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
```
2. hive-server2-log4j2.properties
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties;
```
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
# property.hive.log.file = hive.log
property.hive.log.file = hiveserver2.log
```
3. hive-metastore-log4j2.properties
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-metastore-log4j2.properties;
```
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-metastore-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
# property.hive.log.file = hive.log
property.hive.log.file = metastore.log
```
---
Append the following to the end of `/opt/hadoop-3.4.1/etc/hadoop/core-site.xml` as user hadoop. 
```bash
su - hadoop;
```
```
vi /opt/hadoop-3.4.1/etc/hadoop/core-site.xml;
```
```xml
    <property>
        <name>hadoop.proxyuser.hive.hosts</name>
        <value>*</value>
        <description></description>
    </property>
    <property>
        <name>hadoop.proxyuser.hive.groups</name>
        <value>*</value>
        <description></description>
    </property>
```
Distribute core-site.xml to other servers.
```bash
scp /opt/hadoop-3.4.1/etc/hadoop/core-site.xml test2.hadoop.com:/opt/hadoop-3.4.1/etc/hadoop;
scp /opt/hadoop-3.4.1/etc/hadoop/core-site.xml test3.hadoop.com:/opt/hadoop-3.4.1/etc/hadoop;
```
Now start ZooKeeper and Hadoop, and Create the default directory for Hive.
```bash
# Run on all server as hadoop
/opt/apache-zookeeper-3.8.4-bin/bin/zkServer.sh start;
```
```bash
# Run on test1.hadoop.com as hadoop
/opt/hadoop-3.4.1/sbin/start-all.sh;
/opt/hadoop-3.4.1/bin/mapred --daemon start historyserver;
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir -p /user/hive/warehouse;
/opt/hadoop-3.4.1/bin/hdfs dfs -chmod 770 /user/hive/warehouse;
/opt/hadoop-3.4.1/bin/hdfs dfs -chown -R hive:hadoop /user/hive;
logout;
```
---
Initialize the Hive metastore schema in the PostgreSQL database using the following command.
```bash
/opt/apache-hive-4.0.1-bin/bin/schematool -dbType postgres -initSchema;
```
Start the Hive Metastore in the background using nohup so it continues running after you log out.
```bash
nohup /opt/apache-hive-4.0.1-bin/bin/hive --service metastore > /dev/null 2>&1 &
```
Start HiveServer2 in the background using nohup so it keeps running after logout.
```bash
nohup /opt/apache-hive-4.0.1-bin/bin/hiveserver2 > /dev/null 2>&1 &
```
If you see the following output from `jps -m`, it means that the Hive Metastore and HiveServer2 are running properly.
```bash
jps -m;
```
```text
29139 Jps -m
27764 RunJar /opt/apache-hive-4.0.1-bin/lib/hive-service-4.0.1.jar org.apache.hive.service.server.HiveServer2
27671 RunJar /opt/apache-hive-4.0.1-bin/lib/hive-metastore-4.0.1.jar org.apache.hadoop.hive.metastore.HiveMetaStore
```
You can access the Web UI for Hive service using the following URLs.
| Service  | URL |
| ------------- | ------------- |
|HiveServer2|http://test1.hadoop.com:10002|

---
## Simple Hadoop Test Jobs.
```bash
su - hadoop;
# Generate 1GB of synthetic data (100 bytes × 10 million rows) for TeraSort benchmark
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar teragen 10000000 /tmp/teragen.dat;

# Sort the generated data using TeraSort
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar terasort /tmp/teragen.dat /tmp/terasort.dat;

# Validate the sorted data to ensure correct ordering
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar teravalidate /tmp/terasort.dat /tmp/teravalidate.dat;

# Generate 1GB of random text data (used to test WordCount)
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar randomtextwriter -D mapreduce.randomtextwriter.totalbytes=1000000000 /tmp/randomtextwriter.dat;

# Run WordCount job on the generated random text data
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar wordcount /tmp/randomtextwriter.dat /tmp/wordcount.dat;

/opt/hadoop-3.4.1/bin/hadoop dfs -ls /tmp;
```
## Simple Hive Test.
```bash
su - hive;
# Download Titanic dataset CSV file
wget https://raw.githubusercontent.com/datasciencedojo/datasets/master/titanic.csv -O titanic.csv;

# Remove header from CSV file
tail -n +2 titanic.csv > titanic_noheader.csv;

# Create directory in HDFS for Titanic data
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir /tmp/ext_titanic;

# Upload Titanic CSV (no header) to HDFS
/opt/hadoop-3.4.1/bin/hdfs dfs -put -f titanic_noheader.csv /tmp/ext_titanic/titanic_noheader.csv;

/opt/apache-hive-4.0.1-bin/bin/beeline;
```
```sql
CREATE EXTERNAL TABLE titanic (
  passengerid INT,
  survived INT,
  pclass INT,
  name STRING,
  sex STRING,
  age DOUBLE,
  sibsp INT,
  parch INT,
  ticket STRING,
  fare DOUBLE,
  cabin STRING,
  embarked STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  "separatorChar" = ",",
  "quoteChar"     = "\""
)
STORED AS TEXTFILE
LOCATION '/tmp/ext_titanic';
```
```sql
-- Survival rate by gender
SELECT sex, AVG(survived) AS survival_rate FROM titanic GROUP BY sex;

-- Survival rate by passenger class (1st, 2nd, 3rd)
SELECT pclass, AVG(survived) AS survival_rate FROM titanic GROUP BY pclass;

-- Survival rate by gender and class
SELECT sex, pclass, AVG(survived) AS survival_rate FROM titanic GROUP BY sex, pclass;

-- Survival rate by family status (alone vs. with family)
SELECT
  CASE WHEN sibsp + parch = 0 THEN 'Alone' ELSE 'With Family' END AS family_status,
  AVG(survived) AS survival_rate
FROM titanic
GROUP BY CASE WHEN sibsp + parch = 0 THEN 'Alone' ELSE 'With Family' END;

-- Average age by survival status
SELECT survived, AVG(age) AS avg_age FROM titanic GROUP BY survived;

-- Fare distribution by survival status
SELECT survived, AVG(fare) AS avg_fare FROM titanic GROUP BY survived;

-- Average age and fare overall
SELECT AVG(age) AS avg_age, AVG(fare) AS avg_fare FROM titanic;

-- Average fare by passenger class
SELECT pclass, AVG(fare) AS avg_fare FROM titanic GROUP BY pclass;

-- Survival rate by embarkation port
SELECT embarked, AVG(survived) AS survival_rate FROM titanic GROUP BY embarked;

-- Average fare by embarkation port
SELECT embarked, AVG(fare) AS avg_fare FROM titanic GROUP BY embarked;
```


