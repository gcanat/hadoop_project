# Setting up a Hadoop cluster
sources:
- [Apache.org: Hadoop Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html)
- [Linode: how to install and set up hadoop cluster](https://www.linode.com/docs/guides/how-to-install-and-set-up-hadoop-cluster/)
- [Medium: how to set up hadoop multi node cluster on ubuntu 18.04](https://medium.com/@jootorres_11979/how-to-set-up-a-hadoop-3-2-1-multi-node-cluster-on-ubuntu-18-04-2-nodes-567ca44a3b12)


"For a small cluster (on the order of 10 nodes), it is usually acceptable to run the namenode and the resource manager (yarn) on a single master machine" *source: Hadoop The Definitive Guide (T. White 2015)*


Need to decide which machine will be master (namenode and resource manager) and which will be workers (datanodes).
We decide `tp-hadoop-25` will be master.


## Install java, shh, pdsh, net-tools
`sudo apt-get install default-jre ssh pdsh net-tools`
net-tools is needed to run ifconfig and get the IP address of the machines

## edit `/etc/hosts` on all machines:
```bash
192.168.3.215 tp-hadoop-25
192.168.3.17 tp-hadoop-35
192.168.3.199 tp-hadoop-36
192.168.3.46 tp-hadoop-33
```

## download hadoop, extract and rename the directory

```bash
wget https://dlcdn.apache.org/hadoop/common/stable/hadoop-3.3.1.tar.gz
tar zxf hadoop-3.3.1.tar.gz
mv hadoop-3.3.1 hadoop
```

## Distribute authentication key-pairs
generate ssh key on master node (without passphrase):

```bash
ssh-keygen -t rsa
```

copy content of `.ssh/id_rsa.pub` and paste it in `.ssh/authorized_keys` on each non-master node

## Set environment variables
edit `~/.profile` and add the following line:

```bash
PATH=/home/ubuntu/hadoop/bin:/home/ubuntu/hadoop/sbin:$PATH
```

edit `~/.bashrc` and add the following line:

```bash
export HADOOP_HOME=/home/ubuntu/hadoop
export PATH=${PATH}:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin
```

## Configure master node
Edit `~/hadoop/etc/hadoop/hadoop-env.sh`
```bash
export JAVA_HOME=/usr/lib/jvm/default-java
```

### set namenode location
File: `~/hadoop/etc/hadoop/core-site.xml`
```xml
<configuration>
        <property>
            <name>fs.default.name</name>
            <value>hdfs://tp-hadoop-25:9000</value>
        </property>
    </configuration>
```

### Set path for HDFS
File: `~/hadoop/etc/hadoop/hdfs-site.xml`
```xml
<configuration>
    <property>
            <name>dfs.namenode.name.dir</name>
            <value>file:///home/ubuntu/data/namenode</value>
    </property>

    <property>
            <name>dfs.datanode.data.dir</name>
            <value>file:///home/ubuntu/data/datanode</value>
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
    <!-- Example memory allocation for 2GB RAM node -->
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
            <value>192.168.3.215</value>    
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
tp-hadoop-25
tp-hadoop-33
tp-hadoop-35
tp-hadoop-36
```

## Duplicate config on each node
```bash
cd ~/

cat ~/hadoop/etc/hadoop/workers | while read worker; do
    scp hadoop-3.3.1.tar.gz $worker:~/
    ssh $worker
    tar -xzf hadoop-3.3.1.tar.gz
    mv hadoop-3.3.1 hadoop
    exit
    scp ~/hadoop/etc/hadoop/* $worker:~/hadoop/etc/hadoop/;
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
hadoop fs -mkdir -p /user/ubuntu
hadoop fs -mkdir books
cd ~/
wget -O alice.txt https://www.gutenberg.org/files/11/11-0.txt
wget -O holmes.txt https://www.gutenberg.org/files/1661/1661-0.txt
wget -O frankenstein.txt https://www.gutenberg.org/files/84/84-0.txt

hadoop fs -put alice.txt holmes.txt frankenstein.txt books
```

## Run YARN
```bash
start-yarn.sh
```
