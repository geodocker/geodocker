# GeoDocker Cluster

GeoDocker is a collection of Docker images encapsulating a distributed geo-processing platform based on [GeoTrellis](https://github.com/geotrellis/geotrellis), [GeoMesa](https://github.com/locationtech/geomesa), and [GeoWave](https://github.com/ngageoint/geowave). The emphasis is on providing integration between these projects and exposing geo-processing functionality in Hadoop ecosystem.

## Project Status

This project is in active development. The layout and composition of GeoDocker may change as we explore this our use-case further.
Despite that we're committed to maintaining sanity by providing publicly published, versioned, and tested images. Your feedback and contributions are always welcome.

### Goals
  - Integrate GeoTrellis, GeoWave, and GeoMesa as a unified platform  
  - Provide realistic and convenient distributed integration testing environment
  - Support deployment of GeoDocker to Amazon EMR
  - Explore and support other deployment options like DC/OS and ECS

## Environment

* [Hadoop (HDFS) 2.7.1](https://hadoop.apache.org/)
* [ZooKeeper 3.4.8](https://zookeeper.apache.org/)
* [Accumulo 1.7.2](https://accumulo.apache.org/)
* [Spark 2.0.0](http://spark.apache.org/)

## Images

* [![Build Status](https://api.travis-ci.org/geodocker/geodocker-accumulo.svg)](http://travis-ci.org/geodocker/geodocker-accumulo) [accumulo 1.7.2](https://github.com/geodocker/geodocker-accumulo)
* [![Build Status](https://api.travis-ci.org/geodocker/geodocker-hdfs.svg)](http://travis-ci.org/geodocker/geodocker-hdfs) [hdfs 2.7.1](https://github.com/geodocker/geodocker-hdfs)
* [![Build Status](https://api.travis-ci.org/geodocker/geodocker-spark.svg)](http://travis-ci.org/geodocker/geodocker-spark) [spark 1.6.1](https://github.com/geodocker/geodocker-spark)
* [![Build Status](https://api.travis-ci.org/geodocker/geodocker-zookeeper.svg)](http://travis-ci.org/geodocker/geodocker-zookeeper) [zookeeper 3.4.6](https://github.com/geodocker/geodocker-zookeeper)

## Build and Publish

It is not necessary to build and publish these containers to use them as-is. Pre-built images are available on [quay.io](https://quay.io/organization/geodocker). Building is only necessary in-order to customize and develop GeoDocker.

All images contain a `Makefile` which provide following targets:
 - `build`: Builds the container with `latest` tag
 - `test`: Runs the container tests
 - `publish`: Publishes the  container with `latest` tag and tag provided by a `$TAG` environment variable (ex: `make publish TAG=ABC123`)

These targets are also used by Travis Ci as specified in `.travis.yml`

## Docker Compose: Local Cluster

Those images which contain multiple container roles or depend on instance of other containers to function also provide a `docker-compose.yml` file which allows to easily bring up an local cluster. This cluster can be used for exploration, integration testing, and debugging.

```console
# Build the latest container
~/proj/geodocker-accumulo
❯ make build
docker build -t quay.io/geodocker/accumulo:latest	.
Sending build context to Docker daemon 117.2 kB
Step 1 : FROM quay.io/geodocker/hdfs:latest
...

# Start a local multi-container cluster, use -d option to start in background mode
~/proj/geodocker-accumulo
❯ docker-compose up
Creating geodockeraccumulo_zookeeper_1
Creating geodockeraccumulo_hdfs-name_1
Creating geodockeraccumulo_hdfs-data_1
Creating geodockeraccumulo_accumulo-master_1
Creating geodockeraccumulo_accumulo-tserver_1
Creating geodockeraccumulo_accumulo-monitor_1
Attaching to geodockeraccumulo_hdfs-name_1, geodockeraccumulo_zookeeper_1, geodockeraccumulo_hdfs-data_1, geodockeraccumulo_accumulo-master_1, geodockeraccumulo_accumulo-monitor_1, geodockeraccumulo_accumulo-tserver_1
...

# Inspect running containers
~/proj/geodocker-accumulo
❯ docker-compose ps
                Name                              Command               State                     Ports
--------------------------------------------------------------------------------------------------------------------------
geodockeraccumulo_accumulo-master_1    /sbin/entrypoint.sh master ...   Up
geodockeraccumulo_accumulo-monitor_1   /sbin/entrypoint.sh monitor      Up      0.0.0.0:50095->50095/tcp
geodockeraccumulo_accumulo-tserver_1   /sbin/entrypoint.sh tserver      Up
geodockeraccumulo_hdfs-data_1          /sbin/entrypoint.sh data         Up
geodockeraccumulo_hdfs-name_1          /sbin/entrypoint.sh name         Up      0.0.0.0:50070->50070/tcp
geodockeraccumulo_zookeeper_1          /sbin/entrypoint.sh zkServ ...   Up      0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp

# Inspect logs from running container
~/proj/geodocker-accumulo
❯ docker-compose logs hdfs-name
hdfs-name_1         | Formatting namenode root fs in /data/hdfs/name...
hdfs-name_1         | 16/07/14 02:30:16 INFO namenode.NameNode: STARTUP_MSG:
hdfs-name_1         | /************************************************************
hdfs-name_1         | STARTUP_MSG: Starting NameNode
hdfs-name_1         | STARTUP_MSG:   host = 46c38f89156b/172.19.0.3
hdfs-name_1         | STARTUP_MSG:   args = [-format]
...

# Run a command inside the cluster container
~/proj/geodocker-accumulo
❯ docker-compose run --rm accumulo-master bash -c "set -e \
		&& source /sbin/hdfs-lib.sh \
		&& wait_until_hdfs_is_available \
		&& with_backoff hdfs dfs -test -d /accumulo \
		&& accumulo shell -p GisPwd -e 'createtable test_table'"
Safe mode is OFF
2016-07-14 02:49:25,809 [trace.DistributedTrace] INFO : SpanReceiver org.apache.accumulo.tracer.ZooTraceClient was loaded successfully.
2016-07-14 02:49:25,973 [shell.Shell] ERROR: org.apache.accumulo.core.client.TableExistsException: Table test_table exists
make: *** [test] Error 1

```

## License

* Licensed under the Apache License, Version 2.0: http://www.apache.org/licenses/LICENSE-2.0
