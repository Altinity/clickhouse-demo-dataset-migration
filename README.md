# DEMO DATASET setup

------

## Table of Contents

  * [Introduction](#introduction)
  * [Preparation](#preparation)
    * [Prepare ClickHouse Repo](#prepare-clickhouse-repo)
    * [Prepare Etalon Dataset Server Access](#prepare-etalon-dataset-server-access)
  * [Install and Configure ClickHouse](#install-and-configure-clickhouse)
    * [Install ClickHouse](#install-clickhouse)
    * [Configure ClickHouse](#configure-clickhouse)
      * [Setup Users](#setup-users)
      * [Setup Dictionaries](#setup-dictionaries)
  * [SSH-tunnel setup](#ssh-tunnel-setup)
  * [Datasets setup](#datasets-setup)
    * [Dataset NYC Taxi Rides](#dataset-nyc-taxi-rides)
      * [Setup NYC Taxi Rides Database](#setup-nyc-taxi-rides-database)
      * [Setup NYC Taxi Rides Tables](#setup-nyc-taxi-rides-tables)
      * [Copy NYC Taxi Rides Dataset](#copy-nyc-taxi-rides-dataset)
      * [Check NYC Taxi Rides Dataset](#check-nyc-taxi-rides-dataset)
    * [Dataset STAR](#dataset-star)
      * [Setup STAR Database](#setup-star-database)
      * [Setup STAR Tables](#setup-star-tables)
      * [Copy STAR Dataset](#copy-star-dataset)
      * [Check STAR Dataset](#check-star-dataset)
    * [Dataset AIRLINE](#dataset-airline)
      * [Setup AIRLINE Database](#setup-airline-database)
      * [Setup AIRLINE Tables](#setup-airline-tables)
      * [Copy AIRLINE Dataset](#copy-airline-dataset)
      * [Check AIRLINE Dataset](#check-airline-dataset)
  * [Close SSH-tunnel](#close-ssh-tunnel)
  * [Conclusion](#conclusion)

------


## Introduction

All instructions in this manual were tested on Ubuntu 16.04.
There is no need to setup all datasets from this manual - feel free to skip any of them, if you don't need it.
SSH-tunnel section is provided because 'etalon dataset server' is located behind the firewall, but you may not need this step.

## Preparation

### Prepare ClickHouse Repo

Ensure we have all `apt` - related tools installed
```bash
sudo apt install software-properties-common
```

Include keyserver for repo
```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4
```

Figure out distro codename
```bash
codename=`lsb_release -c|awk '{print $2}'`
echo $codename
```

Build URL to ClickHouse repo based on distro’s codename
```bash
REPOURL="http://repo.yandex.ru/clickhouse/$codename"
echo $REPOURL
```

Add ClickHouse repo located in `$REPOURL`
```bash
sudo apt-add-repository "deb $REPOURL stable main"
```

Update list of available packages
```bash
sudo apt update
```

### Prepare Etalon Dataset Server Access

You'll need to prepare: 
  * hostname or IP address of the 'etalon dataset server' 
  * access key in order to get SSH-access to 'etalon dataset server'

Replace 127.0.0.1 with your 'etalon dataset server' address/hostname

```bash
# replace 127.0.0.1 with your 'etalon dataset server' address/hostname
DATASET_SERVER="127.0.0.1" 
```

Ensure you either already have access key in your `~/.ssh/` folder

```bash
ls -l ~/.ssh/
...
-rw-------  1 user user 1675 Aug  2 11:55 chdemo
...

```

or, if you don't have access key, create access key file and store ssh-access key locally


```bash
mkdir -p ~/.ssh
touch ~/.ssh/chdemo
```

Edit `~/.ssh/chdemo` and save key in it\
Also it has to have limited access rights
```bash
chmod 600 ~/.ssh/chdemo
```

Specify **FULL PATH** to acess key file as ENV variable.\
**IMPORTANT** - please, do not use shortcuts like `~/.ssh/chdemo` - specify real **FULL PATH**. Shortcuts will lead to mess when expanding it in sub-shell's.\
**PLEASE** - full path only.

```bash
# replace "chdemo" with your FULL PATH to your 'etalon dataset server' access key file
DATASET_SERVER_KEY_FILENAME="/home/user/.ssh/chdemo" 
```

## Install and configure ClickHouse

### Install ClickHouse

Install all ClickHouse-related packages: server, client & tools
```bash
sudo apt install clickhouse-client 'clickhouse-server*'
```

Now let’s setup installed ClickHouse

Ensure service is down
```bash
sudo service clickhouse-server stop
```

Create folder where ClickHouse data would be kept. Default is `/var/lib/clickhouse`, if it is ok, just skip this section
```bash
sudo mkdir -p /data1/clickhouse
sudo mkdir -p /data1/clickhouse/tmp
sudo chown -R clickhouse.clickhouse /data1/clickhouse
```

Create folder where dictionaries specs would be kept
```bash
sudo mkdir -p /etc/clickhouse-server/dicts
sudo chown -R clickhouse.clickhouse /etc/clickhouse-server/dicts
```

### Configure ClickHouse

Setup ClickHouse to listen on all network interfaces for both IPv4 and IPv6 \
Edit file `/etc/clickhouse-server/config.xml` \
Ensure `<listen_host>` tags have the following content:

```xml
        <listen_host>::</listen_host>
        <listen_host>0.0.0.0</listen_host>
```

Setup ClickHouse to keep data in specified dirs - in case default `/var/lib/clickhouse` is not OK\
Edit file `/etc/clickhouse-server/config.xml`\
Ensure `<path>` and `<tmp_path>` tags have the following content:

```xml
	<!-- Path to data directory, with trailing slash. -->
	<path>/data1/clickhouse/</path>
	<!-- Path to temporary data for processing hard queries. -->
	<tmp_path>/data1/clickhouse/tmp/</tmp_path>
```
Setup ClickHouse to look for dictionaries in specified dir\
Edit file `/etc/clickhouse-server/config.xml`\
Ensure `<dictionaries_config>` tag has the following content:

```xml
<dictionaries_config>/etc/clickhouse-server/dicts/*.xml</dictionaries_config>
```

#### Setup Users

Setup access for default user from localhost only.\
Edit file `/etc/clickhouse-server/users.xml`\
Ensure `default` user (located inside `<yandex><users><default>`) has `<ip>` tags specified with localhost values only

```xml
<default>
	<networks incl="networks" replace="replace">
		<ip>::1</ip>
		<ip>127.0.0.1</ip>
	</networks>
</default>
```

Setup read-only user for ClickHouse with access from all over the world.\
Username would be testuser and it would not have any password.\
Edit file `/etc/clickhouse-server/users.xml`\
Add new profile called `readonly_set_settings` in `<yandex><profiles>` section right after `<default>` profile tag

```xml
<!-- Profile that allows only read queries and SET settings. -->
<readonly_set_settings>
	<!-- Maximum memory usage for processing single query, in bytes. -->
	<max_memory_usage>10000000000</max_memory_usage>

	<!-- Use cache of uncompressed blocks of data. Meaningful only for processing many of very short queries. -->
	<use_uncompressed_cache>0</use_uncompressed_cache>

	<!-- How to choose between replicas during distributed query processing.
	random - choose random replica from set of replicas with minimum number of errors
	nearest_hostname - from set of replicas with minimum number of errors, choose replica
	with minumum number of different symbols between replica's hostname and local hostname
	(Hamming distance).
	in_order - first live replica is choosen in specified order.
	-->
	<load_balancing>random</load_balancing>
	<readonly>2</readonly>
</readonly_set_settings>
```

Add new `<testuser>` tag with profile referring to just inserted `readonly_set_settings` profile in `<yandex><users>` section right after `<default>` user tag:

```xml
<testuser>
	<password></password>
	<networks incl="networks" replace="replace">
		<!-- access from everywhere -->
		<ip>::/0</ip>
		<ip>0.0.0.0</ip>
	</networks>
	<profile>readonly_set_settings</profile>
	<quota>default</quota>
</testuser>
```

#### Setup Dictionaries

Prepare dictionaries specifications.\
We’ll need **SSH** access to ‘etalon dataset server’.\
Copy dictionaries specifications from ‘etalon dataset server’ to `/etc/clickhouse-server/dicts`

```bash
cd /etc/clickhouse-server/dicts
sudo scp -i $DATASET_SERVER_KEY_FILENAME -P 2222 "root@$DATASET_SERVER:/etc/clickhouse-server/dicts/*" .
```

Ensure we have the following files in `/etc/clickhouse-server/dicts`

```bash
ls -l /etc/clickhouse-server/dicts/*
-rw-r--r-- 1 clickhouse clickhouse  831 Jul  6 06:25 taxi_zones.xml
-rw-r--r-- 1 clickhouse clickhouse 2392 Jul  6 06:26 weather.xml
```

Ensure ClickHouse server is running
```bash
sudo service clickhouse-server restart
```

## SSH-tunnel setup

Also we need to have ClickHouse to have access to ‘etalon dataset server’. Since it is behind the firewall, we need to setup SSH-tunnel for this. \
Make local socket `127.0.0.1:9999` to be forwarded on server `$DATASET_SERVER` to local socket `127.0.0.1:9000` on that server. \
Thus, connecting to `127.0.0.1:9999` we’ll have connect via **SSH** to `127.0.0.1:9000` on server `$DATASET_SERVER`

```bash
ssh -f -N -i $DATASET_SERVER_KEY_FILENAME -p 2222 root@$DATASET_SERVER -L 127.0.0.1:9999:127.0.0.1:9000
```

## Datasets setup

Now let’s setup demo datasets

### Dataset NYC Taxi Rides

Now let’s setup New-York City Taxi Rides dataset

#### Setup NYC Taxi Rides Database
Create database we’ll use

```bash
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS nyc_taxi_rides;"
```

#### Setup NYC Taxi Rides Tables

Drop existing tables if they already exist

```bash
clickhouse-client -q "DROP TABLE IF EXISTS nyc_taxi_rides.central_park_weather_observations;"
clickhouse-client -q "DROP TABLE IF EXISTS nyc_taxi_rides.taxi_zones;"
clickhouse-client -q "DROP TABLE IF EXISTS nyc_taxi_rides.tripdata;"
```

Create tables we’ll use

```bash
clickhouse-client -q "CREATE TABLE nyc_taxi_rides.central_park_weather_observations (
  station_id String,  
  station_name String,  
  weather_date Date,  
  precipitation Float32,  
  snow_depth Float32,  
  snowfall Int32,  
  max_temperature Float32,  
  min_temperature Float32,  
  average_wind_speed Float32
) ENGINE = MergeTree(weather_date, station_id, 8192);"

clickhouse-client -q "CREATE TABLE nyc_taxi_rides.taxi_zones (
  location_id UInt32,  
  zone String,  
  create_date Date DEFAULT toDate(0)
) ENGINE = MergeTree(create_date, location_id, 8192);"

clickhouse-client -q "CREATE TABLE nyc_taxi_rides.tripdata (
  pickup_date Date DEFAULT toDate(tpep_pickup_datetime),  
  id UInt64,  
  vendor_id String,  
  tpep_pickup_datetime DateTime,  
  tpep_dropoff_datetime DateTime,  
  passenger_count Int32,  
  trip_distance Float32,  
  pickup_longitude Float32,  
  pickup_latitude Float32,  
  rate_code_id String,  
  store_and_fwd_flag String,  
  dropoff_longitude Float32,  
  dropoff_latitude Float32,  
  payment_type String,  
  fare_amount String,  
  extra String,  
  mta_tax String,  
  tip_amount String,  
  tolls_amount String,  
  improvement_surcharge String,  
  total_amount Float32,  
  pickup_location_id UInt32,  
  dropoff_location_id UInt32,  
  junk1 String,  
  junk2 String
) ENGINE = MergeTree(pickup_date, (id, pickup_location_id, dropoff_location_id, vendor_id), 8192);"
```

#### Copy NYC Taxi Rides Dataset

Fill newly created tables with data from remote ‘etalon dataset server’

**IMPORTANT:** This operation requires big amount of data to be copied and takes quite long time

```bash
clickhouse-client -q "INSERT INTO nyc_taxi_rides.central_park_weather_observations SELECT * FROM remote('127.0.0.1:9999', 'nyc_taxi_rides.central_park_weather_observations');"
clickhouse-client -q "INSERT INTO nyc_taxi_rides.taxi_zones SELECT * FROM remote('127.0.0.1:9999', 'nyc_taxi_rides.taxi_zones');"
clickhouse-client -q "INSERT INTO nyc_taxi_rides.tripdata SELECT * FROM remote('127.0.0.1:9999', 'nyc_taxi_rides.tripdata');"
```

#### Check NYC Taxi Rides Dataset

After all data copied ensure we have main tables filled with data:

```bash
clickhouse-client -q "SELECT count() FROM nyc_taxi_rides.central_park_weather_observations;"
clickhouse-client -q "SELECT count() FROM nyc_taxi_rides.taxi_zones;"
clickhouse-client -q "SELECT count() FROM nyc_taxi_rides.tripdata;"
```

Ensure all dictionaries are healthy via

```bash
clickhouse-client -q "SELECT * FROM system.dictionaries;"
```

There should be two dictionaries and no errors reported on their statuses 

### Dataset STAR

Now let’s setup Star Observations dataset

#### Setup STAR Database

Create database we’ll use

```bash
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS star;"
```

#### Setup STAR Tables

Drop existing tables if they already exist

```bash
clickhouse-client -q "DROP TABLE IF EXISTS star.starexp;"
```

Create tables we’ll use

```bash
clickhouse-client -q "CREATE TABLE star.starexp (
  antiNucleus UInt32,
  eventFile UInt32,  
  eventNumber UInt32,
  eventTime Float64,
  histFile UInt32,
  multiplicity UInt32,
  NaboveLb UInt32,
  NbelowLb UInt32,
  NLb UInt32,
  primaryTracks UInt32,
  prodTime Float64,
  Pt Float32,
  runNumber UInt32,
  vertexX Float32,
  vertexY Float32,
  vertexZ Float32,
  eventDate Date DEFAULT  CAST(concat(substring(toString(floor(eventTime)), 1, 4), '-', substring(toString(floor(eventTime)), 5, 2), '-', substring(toString(floor(eventTime)), 7, 2)) AS Date)
) ENGINE = MergeTree(eventDate, (eventNumber, eventTime, runNumber, eventFile, multiplicity), 8192);"
```

#### Copy STAR Dataset

Fill newly created tables with data from remote ‘etalon dataset server’

**IMPORTANT:** This operation requires big amount of data to be copied and takes quite long time

```bash
clickhouse-client -q "INSERT INTO star.starexp SELECT * FROM remote('127.0.0.1:9999', 'star.starexp');"
```

#### Check STAR Dataset

After all data copied ensure we have main tables filled with data:

```bash
clickhouse-client -q "SELECT count() FROM star.starexp;"
```

### Dataset AIRLINE

Now let’s setup AIRLINE dataset

#### Setup AIRLINE Database

Create database we’ll use

```bash
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS airline;"
```

#### Setup AIRLINE Tables

Drop existing tables if they already exist

```bash
clickhouse-client -q "DROP TABLE IF EXISTS airline.ontime;"
```

Create tables we’ll use

```bash
clickhouse-client -q "CREATE TABLE IF NOT EXISTS airline.ontime (
  Year UInt16,
  Quarter UInt8,
  Month UInt8,
  DayofMonth UInt8,
  DayOfWeek UInt8,
  FlightDate Date,
  UniqueCarrier String,
  AirlineID UInt32,
  Carrier String,
  TailNum String,
  FlightNum String,
  OriginAirportID UInt32,
  OriginAirportSeqID UInt32,
  OriginCityMarketID UInt32,
  Origin String,
  OriginCityName String,
  OriginState String,
  OriginStateFips String,
  OriginStateName String,
  OriginWac UInt32,
  DestAirportID UInt32,
  DestAirportSeqID UInt32,
  DestCityMarketID UInt32,
  Dest String,
  DestCityName String,
  DestState String,
  DestStateFips String,
  DestStateName String,
  DestWac UInt32,
  CRSDepTime UInt32,
  DepTime UInt32,
  DepDelay Float32,
  DepDelayMinutes Float32,
  DepDel15 Float32,
  DepartureDelayGroups Int32,
  DepTimeBlk String,
  TaxiOut Float32,
  WheelsOff UInt32,
  WheelsOn UInt32,
  TaxiIn Float32,
  CRSArrTime UInt32,
  ArrTime UInt32,
  ArrDelay Float32,
  ArrDelayMinutes Float32,
  ArrDel15 Float32,
  ArrivalDelayGroups Int32,
  ArrTimeBlk String,
  Cancelled Float32,
  CancellationCode String,
  Diverted Float32,
  CRSElapsedTime Float32,
  ActualElapsedTime Float32,
  AirTime Float32,
  Flights Float32,
  Distance Float32,
  DistanceGroup Float32,
  CarrierDelay Float32,
  WeatherDelay Float32,
  NASDelay Float32,
  SecurityDelay Float32,
  LateAircraftDelay Float32,
  FirstDepTime String,
  TotalAddGTime String,
  LongestAddGTime String,
  DivAirportLandings String,
  DivReachedDest String,
  DivActualElapsedTime String,
  DivArrDelay String,
  DivDistance String,
  Div1Airport String,
  Div1AirportID UInt32,
  Div1AirportSeqID UInt32,
  Div1WheelsOn String,
  Div1TotalGTime String,
  Div1LongestGTime String,
  Div1WheelsOff String,
  Div1TailNum String,
  Div2Airport String,
  Div2AirportID UInt32,
  Div2AirportSeqID UInt32,
  Div2WheelsOn String,
  Div2TotalGTime String,
  Div2LongestGTime String,
  Div2WheelsOff String,
  Div2TailNum String,
  Div3Airport String,
  Div3AirportID UInt32,
  Div3AirportSeqID UInt32,
  Div3WheelsOn String,
  Div3TotalGTime String,
  Div3LongestGTime String,
  Div3WheelsOff String,
  Div3TailNum String,
  Div4Airport String,
  Div4AirportID UInt32,
  Div4AirportSeqID UInt32,
  Div4WheelsOn String,
  Div4TotalGTime String,
  Div4LongestGTime String,
  Div4WheelsOff String,
  Div4TailNum String,
  Div5Airport String,
  Div5AirportID UInt32,
  Div5AirportSeqID UInt32,
  Div5WheelsOn String,
  Div5TotalGTime String,
  Div5LongestGTime String,
  Div5WheelsOff String,
  Div5TailNum String
)
ENGINE = MergeTree(FlightDate, (FlightDate, Year, Month, DepDel15), 8192);"
```

#### Copy AIRLINE Dataset

Fill newly created tables with data from remote ‘etalon dataset server’

**IMPORTANT**: This operation requires big amount of data to be copied and takes quite long time

```bash
clickhouse-client -q "INSERT INTO airline.ontime SELECT * FROM remote('127.0.0.1:9999', 'airline.ontime');"
```

#### Check AIRLINE Dataset

After all data copied ensure we have main tables filled with data:

```bash
clickhouse-client -q "SELECT count() FROM airline.ontime;"
```

## Close SSH-tunnel

Now let’s terminate SSH-tunnel to ‘etalon dataset server’

Find `SSH`-tunnel process `PID`
```bash
SSHPID=`sudo netstat -antp|grep LIST|grep 9999|grep ssh|awk '{print $7}'| sed -e 's/\/.*//g'`
echo $SSHPID
```

Ensure found `SSH` `PID` is reasonable - it is our `SSH`-tunnel
```bash
ps ax | grep $SSHPID | grep -v grep
```

and kill it with kill command
```bash
kill $SSHPID
```

## Conclusion

In case all steps were completed successfully, we'll have local copy of one (or more) datasets migrated from 'etalon dataset server'

