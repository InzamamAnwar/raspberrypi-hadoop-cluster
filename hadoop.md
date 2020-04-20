##Configuration
Three Raspberrypi 4, one `name-node-master` `4GB RAM` and two 
`data-nodes` `2GB RAM` are used. All three are connected with
ethernet to same swtich with static IPs. However, Wifi can also be used.
All Raspberrypi's have same hostname so change it before going forward. These can be
changed with `sudo raspi-config`. Below is a sample configuration.

| Componnets          | Sample IP    | Sample Hostname         | 
| ------------------- |:------------:| :----------------------:|
| Raspberrypi 4 (4GB) | 192.168.1.8  | raspberrypi4-namenode   |
| Raspberrypi 4 (2GB) | 192.168.1.10 | raspberrypi4-datanode-1 |
| Raspberrypi 4 (2GB) | 192.168.1.11 | raspberrypi4-datanode-2 |


##Preliminary Steps
**1. Create hadoop user**

Create a user `hadoop` and add it to `sudo` group by running the following commands on all
the nodes (name-node and data-nodes).

1. `sudo adduser hadoop`
2. `sudo adduser hadoop sudo`


**2. Install Java Enviornment**

1. Install openjdk on all nodes. `sudo apt-get install openjdk-8-jdk`.
2. Add `JAVA_HOME` to environment.
    
   ```
   > sudo nano /etc/environment
   # add following line
   JAVA_HOME=/usr/lib/jvm/java-8-openjdk-armhf/jre
    ```

**3. Generate SSH Key-Pairs**

To connect to PIs without password we need to generate SSH key-pairs for each device. 

**Firstly**, generate key-pair on your `host computer` to connect with `name-node-master`.

1. `ssh-keygen -b 4006`
3. Make a new dir on `namenode` using `mkdir -p ~/.ssh && sudo chmod -R 700 ~/.ssh/`
2. `cd` to the directory in which you saved the keys and copy it to the server 
   `scp path\of\key.pub username@host:~/.ssh/authorized_keys`.

**Secondly**, `name-node-master` needs to connect with `data-nodes` without
passwords. Repeat below steps for each `data-node`:

1. Login to `name-node-master` and generate key-pair `ssh-keygen -b 4096`
2. On `data-node`, create a new file, `master.pub` in `~/.ssh` and copy the generated public key
3. Copy `master.pub` to authorized keys `cat ~/.ssh/master.pub >> ~/.ssh/authorized_keys`

**4. Create hosts file**

This file is used by the daemons to communicate between `name-node` and `data-nodes`. Using user `root`
run `sudo nano /etc/hosts` on `name-node` and add the `IPs` and `hostnames` to it.

```
# name-node-master
192.168.1.8         raspberrypi4-namenode

# data-nodes
192.168.1.10        raspberrypi4-datanode-0
192.168.1.11        raspberrypi4-datanode-1
```


##Install Hadoop

Using `hadoop` user, download hadoop on `name-node-master`. 

```
> cd ~
> wget https://downloads.apache.org/hadoop/common/hadoop-3.2.1/hadoop-3.2.1.tar.gz
> tar -xzf hadoop-3.2.1.tar.gz
> mv hadoop-3.2.1 hadoop
```

### Hadoop Environment Configuration

These settings should be done in both `namenodes` and `data-nodes`.

1.  Edit `/home/hadoop/.profile` add the following 
. You can use nano editor and copy paste it using `ctrl+shft+v`.
    
    ```
    PATH=/home/hadoop/hadoop/bin:/home/hadoop/hadoop/sbin:$PATH
    ```

2.  Edit `.bashrc` to add hadoop `PATH`.
    
    ```
    export HADOOP_HOME=/home/hadoop/hadoop
    export PATH=${PATH}:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin
    ```

3. Set `JAVA_HOME` variable in `hadoop-env.sh`
    
   ```
    # /home/hadoop/hadoop/etc/hadoop/hadoop-env.sh
    export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
    ```

4. Clone this repository on `host-computer` and copy the provided
`hadoop` folder to `name-node` at `/home/hadoop/hadoop/etc/hadoop/`. 
These files contain minimun configuration required to work with hadoop.
Some configurations depend on particular setup which are needed to be changed
accordingly and explained below:
    
    1. `core-site.xml` contains the hostname of `name-node-master`.
    For example it can be `raspberrypi4-namenode`. Although, It can be changed,
    but, keep it consistent.
    
    2. `hdfs-site.xml` contains configuration for `hdfs`. Sample filepath for 
    `name-node` and `data-node` are given which of course can be changed. `dfs.replication` indicates number of times 
    data is replicated in cluster. As this example includes two `data-nodes`, factor `1`
    is selected. For bigger clusters it is recommended to have factor `3`.
    
    3. `mapred-site.xml` contains configuration related to `map-reduce` jobs. 
    `YARN` is specified as job and resource scheduler. Important settings
    here are related to memory allocation to `map-reduce` jobs. These settings
    are for `raspberrypi` with `2GB RAM` data-nodes. Increase it if working with `4GB` variants.
    
    4. `yarn-site.xml` contains configuration for `YARN`. It creates `ResourceManager` on 
    `name-node` and `NodeManagers` on `data-nodes` where the job is run. For `ResourceManager`, 
    `IP` of `name-node` is added which of course can be changed. Min max numbers
    of memory allocation are provided for `2GB RAM` variant nodes.
    
    5. `workers` file contains `hostnames` for all the `data-nodes`. An extra
    `data-node` is created on `name-node`. Change hostnames as you like but
    make it consistent.

###Configure Data-Nodes
Copy downloaded hadoop binaries to each data-node

```
> cd ~
> scp hadoop-3.2.1.tar.gz raspberrypi4-datanode-0:/home/hadoop
> scp hadoop-3.2.1.tar.gz raspberrypi4-datanode-1:/home/hadoop
```

Repeat the following steps for both data-nodes.However, valid for as many data-nodes.

```
> ssh raspberrypi4-datanode-0
> tar -xzf hadoop-3.2.1.tar.gz
> mv hadoop-3.2.1 hadoop
```

Copy hadoop configuration files to data-nodes
```
# name-node-master
> scp /home/hadoop/hadoop/etc/hadoop/* raspberrypi4-datanode-0:/home/hadoop/hadoop/etc/hadoop/
```


##Run Hadoop

Installation is complete. Let's check it's working. Start 
with `formatting` hdfs on `name-node-master`

```
# name-node-master
hdfs namenode -format
```

Now, start `HDFS` with `start-dfs.sh`. This will start **NameNode**,
**SecondaryNameNode**, and **DataNode** on the `name-node-master`
and **DataNode** on `data-nodes`. Double check that everything is 
working by running `jps` command on each node. There is simple
web interface available to browse `hdfs filesystem`. It is
available at `http://node-master-ip:9870`. Here, for example
it is available at `http://192.168.1.8:9870`.

Similarly, start **YARN** with `start-yarn.sh`. This will start
**ResourceManager** and **NodeManager** on `name-node-master` and
**NodeManager** on all`data-nodes`. YARN also have a web interface
which is available at port `8088` on `name-node-master`

To stop **HDFS** and **YARN** run `stop-all.sh`