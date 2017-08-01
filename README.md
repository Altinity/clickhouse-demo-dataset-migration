DEMO DATASET setup
==================

Preparation
===========

Ensure we have all `apt` - related tools installed
```bash
sudo apt install software-properties-common
```

Include keyserver for repo
```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4
```

Figure out disto codename
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

Install all ClickHouse-related packages
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


###Configuration edit
=====================


Setup ClickHouse to listen on all network interfaces for both IPv4 and IPv6

Edit file `/etc/clickhouse-server/config.xml`

Ensure `<listen_host>` tags have the following content:


```xml
        <listen_host>::</listen_host>
        <listen_host>0.0.0.0</listen_host>
```

Setup ClickHouse to keep data in specified dirs - in case default /var/lib/clickhouse is not OK

Edit file `/etc/clickhouse-server/config.xml`

Ensure `<path>` and `<tmp_path>` tags have the following content:

```xml
<!-- Path to data directory, with trailing slash. -->
        <path>/data1/clickhouse/</path>
        <!-- Path to temporary data for processing hard queries. -->
        <tmp_path>/data1/clickhouse/tmp/</tmp_path>
```
Setup ClickHouse to look for dictionaries in specified dir

Edit file `/etc/clickhouse-server/config.xml`

Ensure `<dictionaries_config>` tag has the following content:

```xml
<dictionaries_config>/etc/clickhouse-server/dicts/*.xml</dictionaries_config>
```

Setup users

Setup access for default user from localhost only.

Edit file `/etc/clickhouse-server/users.xml`

Ensure default user (located inside `<yandex><users><default>`) has `<ip>` tags specified with localhost values only

```xml
<default>
	<networks incl="networks" replace="replace">
		<ip>::1</ip>
		<ip>127.0.0.1</ip>
	</networks>
</default>
```

Setup read-only user for ClickHouse with access from all over the world.

Username would be testuser and it would not have any password.

Edit file `/etc/clickhouse-server/users.xml`


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

Prepare dictionaries specifications.

We’ll need SSH access to ‘etalon dataset server’, which is `209.170.140.239`

Store ssh-access key locally
```bash
mkdir -p ~/.ssh
touch ~/.ssh/chdemo
```

Edit `~/.ssh/chdemo` and save key in it

Also it has to have limited access rights
```bash
chmod 600 ~/.ssh/chdemo
```

Copy dictionaries specifications from ‘etalon dataset server’ to `/etc/clickhouse-server/dicts`

```bash
cd /etc/clickhouse-server/dicts
sudo scp -i ~/.ssh/chdemo -P 2222 "root@209.170.140.239:/etc/clickhouse-server/dicts/*" .
```

Ensure we have two files in `/etc/clickhouse-server/dicts`
```bash
ls -l /etc/clickhouse-server/dicts/*
-rw-r--r-- 1 clickhouse clickhouse  831 Jul  6 06:25 taxi_zones.xml
-rw-r--r-- 1 clickhouse clickhouse 2392 Jul  6 06:26 weather.xml
```

Ensure ClickHouse server is running
```bash
sudo service clickhouse-server restart
```

Also we need to have ClickHouse to have access to ‘etalon dataset server’. Since it is behind the firewall, we need to setup SSH-tunnel for this. 
Make local socket 127.0.0.1:9999 to be forwarded on server 209.170.140.239 to local socket 127.0.0.1:9000 on that server. 
Thus, connecting to 127.0.0.1:9999 we’ll have connect via SSH to 127.0.0.1:9000 on server 209.170.140.239
```bash
ssh -f -N -i ~/.ssh/chdemo -p 2222 root@209.170.140.239 -L 127.0.0.1:9999:127.0.0.1:9000
```

Now let’s setup tables and demo dataset

Create two databases we’ll use
```bash
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS star;"
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS nyc_taxi_rides;"
```

Drop existing tables if they already exist
```bash
clickhouse-client -q "DROP TABLE IF EXISTS star.starexp;"
clickhouse-client -q "DROP TABLE IF EXISTS nyc_taxi_rides.central_park_weather_observations;"
clickhouse-client -q "DROP TABLE IF EXISTS nyc_taxi_rides.taxi_zones;"
clickhouse-client -q "DROP TABLE IF EXISTS nyc_taxi_rides.tripdata;"
```

Create four tables we’ll use
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

Fill newly created tables with data from remote ‘etalon dataset server’\
\
**IMPORTANT:** This operation requires big amount of data to be copied and takes quite long time

```bash
clickhouse-client -q "INSERT INTO star.starexp SELECT * FROM remote('127.0.0.1:9999', 'star.starexp');"
clickhouse-client -q "INSERT INTO nyc_taxi_rides.central_park_weather_observations SELECT * FROM remote('127.0.0.1:9999', 'nyc_taxi_rides.central_park_weather_observations');"
clickhouse-client -q "INSERT INTO nyc_taxi_rides.taxi_zones SELECT * FROM remote('127.0.0.1:9999', 'nyc_taxi_rides.taxi_zones');"
clickhouse-client -q "INSERT INTO nyc_taxi_rides.tripdata SELECT * FROM remote('127.0.0.1:9999', 'nyc_taxi_rides.tripdata');"
```
After all data copied ensure we have main tables filled with dataset:
```bash
clickhouse-client -q "SELECT count() FROM star.starexp;"
clickhouse-client -q "SELECT count() FROM nyc_taxi_rides.central_park_weather_observations;"
clickhouse-client -q "SELECT count() FROM nyc_taxi_rides.taxi_zones;"
clickhouse-client -q "SELECT count() FROM nyc_taxi_rides.tripdata;"
```

Ensure all dictionaries are healthy via
```bash
clickhouse-client -q "SELECT * FROM system.dictionaries;"
```

There should be two dictionaries and no errors reported on their statuses
Now let’s terminate SSH-tunnel to ‘etalon data server’

Find SSH-tunnel process PID
```bash
SSHPID=`sudo netstat -antp|grep LIST|grep 9999|grep ssh|awk '{print $7}'| sed -e 's/\/.*//g'`
echo $SSHPID
```

Ensure found SSH PID is reasonable - it is our SSH-tunnel
```bash
ps ax | grep $SSHPID | grep -v grep
```

and kill it with kill command
```bash
kill $SSHPID
```

