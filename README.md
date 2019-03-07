# Assignment 2

## Setup

First the docker image was downloaded from the docker cloud. This docker image contained Hadoop version 2.9.2.
Next the HDFS (Hadoop Distributed Files System) was initialized. We first format the hdfs, this ensures us that we don't accedentaly use a previous installation. This is done by running:
```
bin/hdfs namenode -format
```

We can now start the HDFS by running:
```
sbin/start-dfs.sh
```

Now we have the HDFS running it is time to make some directories. For this assignemnt we will work from the `/user/root` directory. Thuw we first have to create those directories. Since these will be inside the HDFS we have to call `dfs` before the `mkdir` command. This is a repeating pattern, to do anything withing the HDFS we always have to run `dfs` first. The directories are made using:
```
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/root
```

The HDFS can be stopped by running:
```
sbin/stop-dfs.sh
```

## Pseudo-Distributed Operation
In this assignment we run Hadoop pseudo-distributed. This means that we emulate a cluster on one machine (each Hadoop daemon runs in a separate Java process). There are two files that create this pseudo-distributed operation setup.

`core-site.xml`:

```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9001</value>
    </property>
</configuration>
```
This sets the namenode on localhost with port 9001.


`hdfs-site.xml`:
```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```
This defines the desired replication to 1. In real world usages you would usualy want this number to be at least 3. But since we are only simulating it localy 1 will suffice.