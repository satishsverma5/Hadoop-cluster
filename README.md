# Multi-Node Apache Hadoop 2.7.0 Cluster Setup

The project uses the following technologies:
1. Install Java SDK 8
2. Install Hadoop 2.7.0
3. Install Spark
4. Install SSH Remote Access
5. 
6. Test
7. 

Allocate four 8GBRam/2Core, Ubuntu 14.04 instances in the cloud (use a cloud service provider of your choice).
We called them: master (IP addr: x.y.z.156) slave1 (IP addr: x.y.z.157) and slave2 (IP addr: x.y.z.158).

# Install Java JDK 8
```
ubuntu@master:~$ sudo apt-add-repository ppa:webupd8team/java
ubuntu@master:~$ sudo apt-get update
ubuntu@master:~$ sudo apt-get install -y oracle-java8-installer
```
set ENV for all use
```
vim /etc/bash.bashrc
# append JAVA_HOME values at end of file
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
```

Check Java Version
```
ubuntu@master:~$ java -version

java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)
```

# Disabling IPV6 on Ubuntu
```
ubuntu@master:~$ sudo vim /etc/sysctl.conf
```
Add the following lines at the end:
```
#disable ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```
Check:
```
ubuntu@master:~$ cat /proc/sys/net/ipv6/conf/all/disable_ipv6 
0

ubuntu@master:~$ sudo sysctl -p
# (Must do: sudo sysctl -p for these changes to take effect).
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

ubuntu@master:~$ cat /proc/sys/net/ipv6/conf/all/disable_ipv6 
1
```
Note: 0 means it's enabled and 1 - disabled.

# Prepare to Install Hadoop.
### STEP-1. Create user group "hadoop".
```
ubuntu@master:~$ sudo addgroup hadoop
Adding group `hadoop' (GID 1000) ...
Done.

ubuntu@master:~$ sudo adduser --ingroup hadoop hduser
Adding user `hduser' ...
Adding new user `hduser' (1001) with group `hadoop' ...
Creating home directory `/home/hduser' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: (Keep same on all nodes for convenience)
Retype new UNIX password: 
passwd: password updated successfully
Changing the user information for hduser
Enter the new value, or press ENTER for the default
    Full Name []: 
    Room Number []: 
    Work Phone []: 
    Home Phone []: 
    Other []: 
Is the information correct? [Y/n] Y
```
### STEP-2. Download Hadoop 2.7.0 and extract it to /opt/ directory on All Node
Below are the commands,
```
ubuntu@master:~$ cd /opt/
ubuntu@master:~$ sudo wget http://mirrors.sonic.net/apache/hadoop/common/hadoop-2.7.0/hadoop-2.7.0.tar.gz
ubuntu@master:~$ sudo tar –xvf hadoop-2.7.0.tar.gz
ubuntu@master:~$ sudo ln -s /opt/hadoop-2.7.0 /opt/hadoop
ubuntu@master:~$ sudo chown hduser.hadoop /opt/hadoop
ubuntu@master:~$ sudo chown hduser.hadoop /opt/hadoop-2.7.0
```
### STEP-3. Create Hadoop data directories
```
ubuntu@master:~$ sudo mkdir -p /data/hadoop-data/nn 
ubuntu@master:~$ sudo mkdir -p /data/hadoop-data/snn 
ubuntu@master:~$ sudo mkdir -p /data/hadoop-data/dn 
ubuntu@master:~$ sudo mkdir -p /data/hadoop-data/mapred/system 
ubuntu@master:~$ sudo mkdir -p /data/hadoop-data/mapred/local
ubuntu@master:~$ sudo chown -R hduser.hadoop /data/hadoop-data
```

### STEP-4. Configure Variables in hduser
#
```
ubuntu@master:~$  su - hduser
Password:
```
append following values at end of file
```
hduser@master:~$ vim .bashrc

#HADOOP VARIABLES START
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export HADOOP_HOME=/opt/hadoop
export HADOOP_INSTALL=/opt/hadoop
export PATH=$PATH:$HADOOP_INSTALL/bin
export PATH=$PATH:$HADOOP_INSTALL/sbin
export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
export HADOOP_COMMON_HOME=$HADOOP_INSTALL
export HADOOP_HDFS_HOME=$HADOOP_INSTALL
export YARN_HOME=$HADOOP_INSTALL
export YARN_CONF_DIR=$HADOOP_INSTALL/etc/hadoop
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_INSTALL/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_INSTALL/lib"
#HADOOP VARIABLES END
```
Reload Configuration using below command.
```
hduser@master:~$ source .bashrc
```
### STEP-5. Update Configuration Files
Now edit $HADOOP_HOME/etc/hadoop/hadoop-env.sh file and set JAVA_HOME environment variable with JDK base directory path.
```
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
```

Modify core-site.xml on Master and Slave nodes with following options. Master and slave nodes should use the same value for this property: fs.defaultFS, and should be pointing to master node only.
```
hduser@master: vim /opt/hadoop/etc/hadoop/core-site.xml

# Insert between "<configuration>  </configuration>" tags:

    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/tmp/hadoop</value>
        <description>A base for other temporary directories.</description>
    </property>

    <property>
        <name>fs.default.name</name>
        <value>hdfs://master:9000</value>
    </property>

```
Modify mapred-site.xml on Master node only with following options.
```
hduser@master: vim /opt/hadoop/etc/hadoop/mapred-site.xml

# Insert between "<configuration>  </configuration>" tags:

  <property>
        <name>mapreduce.jobtracker.http.address</name>
        <value>master.dev.nazara.com:50030</value>
        <description>The host and port that the MapReduce job tracker runs
            at.  If "local", then jobs are run in-process as a single map
            and reduce task.
        </description>
    </property>

    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
```
Modify hdfs-site.xml on Master and Slave Nodes. Before that, as hduser, create the following directories on all the nodes, master and slaves.
```
cd /home/hduser/hadoop
hduser@ubuntu:/home/hduser/hadoop$ mkdir -p ./yarn_data/hdfs/namenode
hduser@ubuntu:/home/hduser/hadoop$ mkdir -p ./yarn_data/hdfs/datanode
```

```
$ hdfs namenode –format
```
# Step 7 : Commands for starting and stopping Hadoop Cluster

### Start/Stop HDFS using below commands
```
sh $HADOOP_HOME/sbin/start-dfs.sh
sh $HADOOP_HOME/sbin/stop-dfs.sh
```
### Start/Stop YARN services using below commands
```
sh $HADOOP_HOME/sbin/start-yarn.sh
sh $HADOOP_HOME/sbin/stop-yarn.sh
```



Now you can access Hadoop Services in Browser

Name Node: http://master:50070/
YARN Services: http://master:8088/
Secondary Name Node: http://master:50090/
Data Node 1: http://master:50075/
Data Node 2: http://slave1:50075/


create topic (sample1)
./bin/kafka-topics.sh --create --zookeeper master.dev.nazara.com:2181 --replication-factor 1 --partitions 1 --topic sample1

show topic list
./bin/kafka-topics.sh --list --zookeeper master.dev.nazara.com:2181

produc message on topic sample1
./bin/kafka-console-producer.sh --broker-list master.dev.nazara.com:9092  --topic sample

check message 
./bin/kafka-console-consumer.sh --zookeeper master.dev.nazara.com:2181 --topic sample1 --from-beginning



