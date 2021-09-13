# Setting up a Hadoop cluster
source: [Linode](https://www.linode.com/docs/guides/how-to-install-and-set-up-hadoop-cluster/)

"For a small cluster (on the order of 10 nodes), it is usually acceptable to run the namenode and the resource manager (yarn) on a single master machine" *source: Hadoop The Definitive Guide (T. White 2015)*


Decide which machine will be master (namenode and resource manager) and which will be slave (datanode)

## edit `/etc/hosts` on all machines:
```bash
# for example
192.0.2.1    node-master
192.0.2.2    node1
192.0.2.3    node2
```

## Install java, shh and pdsh
`sudo apt-get install default-jre ssh pdsh`

## download hadoop, extract and rename the directory

```bash
wget https://dlcdn.apache.org/hadoop/common/stable/hadoop-3.3.1.tar.gz
tar zxf hadoop-3.3.1.tar.gz
mv hadoop-3.3.1 hadoop
```

## Distribute Authentication Key-pairs
on master node do:

```bash
ssh-keygen -b 4096
```
copy content of `.ssh/id_rsa.pub` and paste it in `.ssh/authorized_keys` on each non-master node

## Set environment variables
edit `~/.profile` and add the following line:

```bash
PATH=/home/hadoop/hadoop/bin:/home/hadoop/hadoop/sbin:$PATH
```

edit `~/.bashrc` and add the following line:

```bash
export HADOOP_HOME=/home/hadoop/hadoop
export PATH=${PATH}:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin
```

## Configure master node
Edit `~/hadoop/etc/hadoop/hadoop-env.sh`
```bash
export JAVA_HOME=/usr/lib/jvm/.... <- need to complete this
```

### set namenode location
File: `~/hadoop/etc/hadoop/core-site.xml`
```xml
<configuration>
        <property>
            <name>fs.default.name</name>
            <value>hdfs://node-master:9000</value>
        </property>
    </configuration>
```

### Set path for HDFS
File: `~/hadoop/etc/hadoop/hdfs-site.xml`
```xml
<configuration>
    <property>
            <name>dfs.namenode.name.dir</name>
            <value>/home/hadoop/data/nameNode</value>
    </property>

    <property>
            <name>dfs.datanode.data.dir</name>
            <value>/home/hadoop/data/dataNode</value>
    </property>

    <property>
            <name>dfs.replication</name>
            <value>3</value>
    </property>
</configuration>
```
### Set YARN as Job Scheduler
File: `~/hadoop/etc/hadoop/mapred-site.xml`
```xml
<configuration>
    <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
    </property>
    <property>
            <name>yarn.app.mapreduce.am.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <property>
            <name>mapreduce.map.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <property>
            <name>mapreduce.reduce.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <!-- Example memory allocation for 2G BRAM node -->
    <property>
            <!-- memory allocated to the application master -->
            <name>yarn.app.mapreduce.am.resource.mb</name>
            <value>512</value>
    </property>

    <property>
            <!-- memory allocated to each map operation -->
            <name>mapreduce.map.memory.mb</name>
            <value>256</value>
    </property>

    <property>
            <!-- memory allocated to each reduce operation -->
            <name>mapreduce.reduce.memory.mb</name>
            <value>256</value>
    </property>
</configuration>
```

### Configure YARN
File: `~/hadoop/etc/hadoop/yarn-site.xml`
```xml
<configuration>
    <property>
            <name>yarn.acl.enable</name>
            <value>0</value>
    </property>

    <property>
            <name>yarn.resourcemanager.hostname</name>
            <!-- replace value with public IP address of "node-master" -->
            <value>127.0.0.1</value>    
    </property>

    <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
    </property>
    <property>
            <!-- How much memory can be allocated for YARN containers on a single node -->
            <name>yarn.nodemanager.resource.memory-mb</name>
            <value>1536</value>
    </property>

    <property>
            <!-- How much memory a single container can consume -->
            <name>yarn.scheduler.maximum-allocation-mb</name>
            <value>1536</value>
    </property>

    <property>
            <!-- minimum memory allocation allowed -->
            <name>yarn.scheduler.minimum-allocation-mb</name>
            <value>128</value>
    </property>

    <property>
            <!-- disable virtual-memory checking to avoid problems with JDK8 -->
            <name>yarn.nodemanager.vmem-check-enabled</name>
            <value>false</value>
    </property>

</configuration>
```

### Configure workers
File: `~/hadoop/etc/hadoop/workers`
```bash
node1
node2
```

## Duplicate config on each node
```bash
cd /home/hadoop/
scp hadoop-*.tar.gz node1:/home/hadoop
scp hadoop-*.tar.gz node2:/home/hadoop

for node in node1,node2; do
    ssh node
    tar -xzf hadoop-3.1.2.tar.gz
    mv hadoop-3.1.2 hadoop
    exit
    scp ~/hadoop/etc/hadoop/* $node:/home/hadoop/hadoop/etc/hadoop/;
done
```

## Format HDFS
on node master:
```bash
hdfs namenode -format
```

## Start HDFS
```bash
start-dfs.sh
```    
## Put data to HDFS
```bash
hdfs dfs -mkdir -p /user/hadoop
hdfs dfs -mkdir books
cd /home/hadoop
wget -O alice.txt https://www.gutenberg.org/files/11/11-0.txt
wget -O holmes.txt https://www.gutenberg.org/files/1661/1661-0.txt
wget -O frankenstein.txt https://www.gutenberg.org/files/84/84-0.txt

hdfs dfs -put alice.txt holmes.txt frankenstein.txt books
```

## Run YARN
```bash
start-yarn.sh
```
