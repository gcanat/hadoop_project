# Setting up a Hadoop cluster
sources:
- [Apache](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html)
- [Linode](https://www.linode.com/docs/guides/how-to-install-and-set-up-hadoop-cluster/)
- [Medium](https://medium.com/@jootorres_11979/how-to-set-up-a-hadoop-3-2-1-multi-node-cluster-on-ubuntu-18-04-2-nodes-567ca44a3b12)


"For a small cluster (on the order of 10 nodes), it is usually acceptable to run the namenode and the resource manager (yarn) on a single master machine" *source: Hadoop The Definitive Guide (T. White 2015)*


Need to decide which machine will be master (namenode and resource manager) and which will be workers (datanodes).
We decide `tp-hadoop-25` will be master.


## Install java, shh, pdsh, net-tools
`sudo apt-get install default-jre ssh pdsh `


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
start-all.sh
```

## Put some data to HDFS
```bash
hadoop fs -mkdir -p /user/ubuntu
hadoop fs -mkdir books
cd ~/
wget -O alice.txt https://www.gutenberg.org/files/11/11-0.txt
wget -O holmes.txt https://www.gutenberg.org/files/1661/1661-0.txt
wget -O frankenstein.txt https://www.gutenberg.org/files/84/84-0.txt

hadoop fs -put alice.txt holmes.txt frankenstein.txt books
```
