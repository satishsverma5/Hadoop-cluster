### Apache Oozie Configuration with Hadoop 2.7.0


Install Oozie with Hadoop 2+ environment.

## Step 1: 
# Download Oozie 4.1 from the Apache URL and save the tarball to any directory
```
cd ~/Downloads
tar -zxf oozie-4.1.0.tar.gz
sudo mv oozie-4.1.0 /usr/local/oozie-4.1.0
```
### Step 2: 
# Assuming you have maven installed, if not, refer to the installation instructions here (http://www.mkyong.com/maven/install-maven-on-mac-osx/)

### Step 3:
# Update the pom.xml to change the default hadoop version to 2.3.0. The reason we’re not changing it to hadoop version 2.6.0 here is because 2.3.0-oozie-4.1.0.jar is the latest available jar file. Luckily it works with higher versions in 2.x series
  ```
  cd /usr/local/oozie-4.1.0
vim pom.xml
--Search for
<hadoop.version>1.1.1</hadoop.version>
--Replace it with
<hadoop.version>2.3.0</hadoop.version>
```
### Step 4:
# Build Oozie executable
```
mvn clean package assembly:single -P hadoop-2 -DskipTests
```
### Step 5:
# The executable will be generated in the target sub directory under distro dir. Move it to a new folder under /usr/local/
```
cd ~/Downloads/oozie-4.1.0/distro/target
tar -zxf oozie-4.1.0-distro.tar.gz
sudo mv oozie-4.1.0 /user/local/oozie-4.1.0
```
###Step 6:
# Copy the hadoop lib files and Download ext-2.2.zip, this file is no longer available on the extjs.com, that’s why downloading from cloudera archive
```
cd /usr/local/oozie-4.1.0
mkdir libext
cp -R ~/Downloads/oozie-4.1.0/hadooplibs/hadoop-2/target/hadooplibs/hadooplib-2.3.0.oozie-4.1.0/* libext
cd libext
curl -O http://archive.cloudera.com/gplextras/misc/ext-2.2.zip
```
### Step 7:
# Install zip  if it is not installed 
```
apt-get install zip
```
### Step 8:
# Build the war file for oozie web console
```
cd ..
./bin/oozie-startup.sh prepare-war
```
### Step 9:
# Create sharelib directory under user home dir on HDFS. Please check for the Namenode port in your configuration, I’m using 54310. This command will create ~/share/lib/ directory in your HDFS home and will copy the required lib file
```
./bin/oozie-setup.sh sharelib create fs -hdfs://localhost:54310
```
### Step 10:
# Setup oozie metadata repository. By default oozie uses a derby database as the metadata store, however, I’ve chosen local MySQL instance to get the real feel of enterprise installation.  To install MySQL, please refer here (https://dev.mysql.com/doc/refman/5.6/en/osx-installation-pkg.html)
```
 For Derby metastore:
./bin/ooziedb.sh create -sqlfile oozie.sql -run
For MySQL metastore: login to MySQL and run create the database and schema for repository
mysql>CREATE DATABASE OOZIEDB;
mysql>CREATE USER "OOZIE" IDENTIFIED BY "PASSWORD";
mysql>GRANT ALL PRIVILEGES ON *.* TO OOZIE;
mysql>FLUSH PRIVILEGES;
```
Ensure that mysql driver is placed in libext folder, change the following properties under con/oozie-site.xml
oozie.service.JPAService.jdbc.driver, set it to com.mysql.jdbc.Driver
oozie.service.JPAService.jdbc.url set it to jdbc:mysql://OOZIEDB
oozie.service.JPAService.jdbc.username, set it to mysql user name
oozie.service.JPAService.jdbc.password for password
Run the below command to create metadata tables in the repository:
```
./bin/ooziedb.sh create db -run
```
### Step 11:
# Configure Hadoop, etc/hadoop/core-site.xml  for Oozie proxy process. Replace $USER with the user which will run oozie process. For Mac OS, you may have to provide group name explicitly
```
<property>
<name>hadoop.proxyuser.$USER.hosts</name>
<value>*</value>
</property>
<property>
<name>hadoop.proxyuser.$USER.groups</name>
<value>*/value>
</property>
```
### Step 12:
# All set, now let’s start Oozie. Once started, the web console should be up at http://localhost:11000/oozie/
```
cd $OOZIE_HOME/bin
./oozied.sh start
```
### Step 13:
# Test the installation by submitting an example workflow. Oozie workflow is defined in an xml file and its execution properties are defined in job.properties file.Change the port numbers according to your setup
```
cd examples/apps/map-reduce/
vim job.properties
##########################################
nameNode=hdfs://localhost:54310
jobTracker=localhost:8050
queueName=default
examplesRoot=examples
oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/apps/map-reduce
outputDir=map-reduce
```
Before submitting the workflow, we need to copy the examples directory under $OOZIE_HOME to the user home on HDFS.
```
hadoop fs -put examples examples
```
Now submit the workflow from $OOZIE_HOME dir
```
./bin/oozie job -oozie http://localhost:11000/oozie -config /examples/apps/map-reduce/job.properties -run
```
### Step 14:
# Check the job status from oozie web console and job tracker console

