## Preliminary Steps
Spark does have its own cluster manager but, here, `YARN` will be used
as default resource manager for `Spark`. Another manager [Mesos](https://mesos.apache.org/) 
can also be used. However not in scope of this project.

`Spark` will be installed on the same `hadoop` cluster created earlier.
This guide assumes that there is a user `hadoop` in sudo group and hadoop
is already installed.

## Install Spark

Download pre built [binaries](https://downloads.apache.org/spark/spark-2.4.5/spark-2.4.5-bin-without-hadoop.tgz).
Here spark version `2.4.5` and without `hadoop`. Otherwise, download
it directly to `name-node-master`

```shell script

# ssh using user hadoop in name-node-master

> wget https://downloads.apache.org/spark/spark-2.4.5/spark-2.4.5-bin-without-hadoop.tgz
> tar -xzf spark-2.4.5-bin-without-hadoop.tgz
> mv spark-2.4.5-bin-without-hadoop.tgz spark

# add spark binaries to PATH at the end of .profile 
nano ~/.profile
PATH=/home/hadoop/spark/bin:$PATH

# make hadoop configuration visible to spark
export HADOOP_CONF_DIR=/home/hadoop/hadoop/etc/hadoop
export LD_LIBRARY_PATH=/home/hadoop/spark

# add SPARK_HOME
export SPARK_HOME=/home/hadoop/spark
```

Restart your `ssh` connection, by exiting and logging in again. 

## Configure YARN for SPARK
Copy configuration file provided in `spark` folder provided in this
repository. There are some import configurations which are discussed
below:
    
   1. `spark.master` property assigns `yarn` as default resource manager
   2. `spark.driver.memory` property assigns max memory allocated 
      by `yarn` for `spark driver`. This value ,512 MB, is a sample
      value for raspberrypi 4 2GB RAM versions. Furthermore, value for
      `spark.driver.memory` should be assigned with respect to value for
      `yarn.scheduler.maximum-allocation-mb` in `/home/hadoop/hadoop/etc/hadoop/yarn-site.xml`.
   3. `spark.executor.memory` property represents memory allocation for
      `spark` executors. 
   4. `spark.eventLog.dir` and `spark.history.fs.logDirectory` set path
      for history server where `spark` job logs are stored. `Hostname`
      for `HDFS` can also be changed at will. Here `hdfs://192.168.1.8:9000`
      is used as sample, in accordance with `hadoop` installation.
   5. `spark.history.ui.port` describe the port where `spark` job log 
      history is available on `http://192.168.1.8:18080` 
      
## Run Spark
Spark installation is complete. Let's run an axample provided with
`spark` package. It calculates *Pi* in distributed fashion. Run the 
following command on `name-node`

```shell script
> spark-submit --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  /home/hadoop/spark/examples/jars/spark-examples_2.11-2.4.5.jar 10
```  

`spark-submit` is a shell script used to submit `spark` jobs to `yarn`.
Yarn web UI, available at `http://192.168.1.8:8088`. Furthermore,
`spark` logs are also available at `http://192.168.1.8:18080`.
Make sure to use `IP` of your `name-node`