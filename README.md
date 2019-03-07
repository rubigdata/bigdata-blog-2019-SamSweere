# Assignment 2

In this assignment we will implement a Map-Reduce program on an HDFS (Hadoop Distributed Files System).

## Setup

First the docker image was downloaded from the docker cloud. This docker image contained Hadoop version 2.9.2.
Next the HDFS  was initialized. We first format the hdfs, this ensures us that we don't accedentaly use a previous installation. This is done by running:
```
bin/hdfs namenode -format
```

We can now start the HDFS by running:
```
sbin/start-dfs.sh
```

Now we have the HDFS running it is time to make some directories. For this assignment we will work from the `/user/root` directory. Thou we first have to create those directories. Since these will be inside the HDFS we have to call `dfs` before the `mkdir` command. This is a repeating pattern, to do anything withing the HDFS we always have to run `dfs` first. The directories are made using:
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

## Dataset
The data we will use to test our Map-Reduce algorithm will be the *Complete Shakespeare*. We can download this from the github of this course:

```
wget https://raw.githubusercontent.com/rubigdata-dockerhub/hadoop-dockerfile/master/100.txt
```

Next we have to set this data as the input for our Map-Reduce program:
```
bin/hdfs dfs -put 100.txt input
```

## Wordcount
The Map-Reduce program we are going to work with will be a word counter. The basis of this program will be taken from [Hadoop's example code](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0). The wordcount code is put into `Wordcount.java`.

### Setup
First we have to setup the enviroment vairables:
```
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
```

We can now compile `Wordcount.java` and creat a jar (Java Archive):
```
bin/hadoop com.sun.tools.javac.Main WordCount.java
jar cf wc.jar WordCount*.class
```
Finally we can run the code:
```
bin/hadoop jar wc.jar WordCount input output
```

### Reading and resetting output
Since the Wordcount Map-Reduce program will run on the Hadoop cluster the output will also be storen on the HDFS. To examine the output we can first copy the output of the Wordcount from the HDFS to our local (client) machine using the `dfs -get` command:

```
bin/hdfs dfs -get output output
```

We can now examine the output by running:
```
cat output/part-r-00000
```
This will show all the words and the corresponding amount of occurences in *Complete Shakespeare*:

```
...
unmuzzled	1
unnatural	23
unnatural!	1
unnatural,	5
unnatural.	6
unnatural;	1
unnaturally	1
...
```
Note that the Wordcount program doesn't take punctuation marks and interrobang (?!:() etc) into account.



A thing to note is that the Wordcount code is that if an previous output folder already exitst it will throw an error. In order to be able to run the wordcount code again we have to remove the previous output folder. This can be done using:

```
bin/hdfs dfs -rm -r -f /user/root/output
```
