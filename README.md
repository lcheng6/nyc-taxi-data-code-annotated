# Unified New York City Taxi and Uber data

Code in support of this post: [Analyzing 1.1 Billion NYC Taxi and Uber Trips, with a Vengeance](http://toddwschneider.com/posts/analyzing-1-1-billion-nyc-taxi-and-uber-trips-with-a-vengeance/)

This repo provides scripts to download, process, and analyze data for over 1.3 billion taxi and Uber trips originating in New York City. The data is stored in a [PostgreSQL](http://www.postgresql.org/) database, and uses [PostGIS](http://postgis.net/) for spatial calculations, in particular mapping latitude/longitude coordinates to census tracts.

The [yellow and green taxi data](http://www.nyc.gov/html/tlc/html/about/trip_record_data.shtml) comes from the NYC Taxi & Limousine Commission, and [Uber data](https://github.com/fivethirtyeight/uber-tlc-foil-response) comes via FiveThirtyEight, who obtained it via a FOIL request. In August 2016, the TLC began providing for-hire vehicle trip records as well.

## Instructions

Your mileage may vary, but on my MacBook Air, this process took about 3 days to complete. The unindexed database takes up 330 GB on disk. Adding indexes for improved query performance increases total disk usage by another 100 GB.

##### 1. Install [PostgreSQL](http://www.postgresql.org/download/) and [PostGIS](http://postgis.net/install)
Enable PostGIS on AWS RDS Postgres [Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html#Appendix.PostgreSQL.CommonDBATasks.PostGIS)

Both are available via [Homebrew](http://brew.sh/) on Mac OS X

##### 2. Download raw taxi data

`./download_raw_data.sh`

##### 3. Initialize database and set up schema

`./initialize_database.sh`

##### 4. Import taxi data into database and map to census tracts

`./import_trip_data.sh`

##### 5. Optional: download and import Uber data from FiveThirtyEight's GitHub repository, and TLC for-hire vehicle records

`./download_raw_uber_data.sh`
<br>
`./import_uber_trip_data.sh`
<br>
`./import_fhv_trip_data.sh`

##### 6. Analysis

Additional Postgres and [R](https://www.r-project.org/) scripts for analysis are in the <code>analysis/</code> folder, or you can do your own!

## Schema

- `trips` table contains all yellow and green taxi trips, plus Uber pickups from April 2014 through September 2014. Each trip has a `cab_type_id`, which references the `cab_types` table and refers to one of `yellow`, `green`, or `uber`. Each trip maps to a census tract for pickup and dropoff
- `nyct2010` table contains NYC census tracts plus the Newark Airport. It also maps census tracts to NYC's official neighborhood tabulation areas
- `taxi_zones` table contains the TLC's official taxi zone boundaries. Starting in July 2016, the TLC no longer provides pickup and dropoff coordinates. Instead, each trip comes with taxi zone pickup and dropoff location IDs
- `uber_trips_2015` table contains Uber pickups from Jan–Jun, 2015. These are kept in a separate table because they don't have specific latitude/longitude coordinates, only location IDs. The location IDs are stored in the `taxi_zone_lookups` table, which also maps them (approximately) to neighborhood tabulation areas
- `fhv_trips` table contains all FHV trip records made available by the TLC
- `central_park_weather_observations` has summary weather data by date

## Other data sources

These are bundled with the repository, so no need to download separately, but:

- Shapefile for NYC census tracts and neighborhood tabulation areas comes from [Bytes of the Big Apple](http://www.nyc.gov/html/dcp/html/bytes/districts_download_metadata.shtml)
- Shapefile for taxi zone locations comes from the TLC
- Mapping of FHV base numbers to names comes from [the TLC](http://www.nyc.gov/html/tlc/html/about/statistics.shtml)
- Central Park weather data comes from the [National Climatic Data Center](http://www.ncdc.noaa.gov/)

## Data issues encountered

- Remove carriage returns and empty lines from TLC data before passing to Postgres `COPY` command
- Some raw data files have extra columns with empty data, had to create dummy columns `junk1` and `junk2` to absorb them
- Two of the `yellow` taxi raw data files had a small number of rows containing extra columns. I discarded these rows
- The official NYC neighborhood tabulation areas (NTAs) included in the shapefile are not exactly what I would have expected. Some of them are bizarrely large and contain more than one neighborhood, e.g. "Hudson Yards-Chelsea-Flat Iron-Union Square", while others are confusingly named, e.g. "North Side-South Side" for what I'd call "Williamsburg", and "Williamsburg" for what I'd call "South Williamsburg". In a few instances I modified NTA names, but I kept the NTA geographic definitions
- The shapefile includes only NYC census tracts. Trips to New Jersey, Long Island, Westchester, and Connecticut are not mapped to census tracts, with the exception of the Newark Airport, for which I manually added a fake census tract
- The Uber 2015 and FHV data uses location IDs instead of latitude/longitude. The location IDs do not exactly overlap with the NYC neighborhood tabulation areas (NTAs) or census tracts, but I did my best to map Uber location IDs to NYC NTAs

## Why not use BigQuery or Redshift?

[Google BigQuery](https://cloud.google.com/bigquery/) and [Amazon Redshift](https://aws.amazon.com/redshift/) would probably provide significant performance improvements over PostgreSQL. A lot of the data is already available on BigQuery, but in scattered tables, and each trip has only latitude and longitude coordinates, not census tracts and neighborhoods. PostGIS seemed like the easiest way to map coordinates to census tracts. Once the mapping is complete, it might make sense to load the data back into BigQuery or Redshift to make the analysis faster. Note that BigQuery and Redshift cost some amount of money, while PostgreSQL and PostGIS are free.

## TLC summary statistics

There's a Ruby script in the `tlc_statistics/` folder to import data from the TLC's [summary statistics reports](http://www.nyc.gov/html/tlc/html/technology/aggregated_data.shtml):

`ruby import_statistics_data.rb`

## Taxi vs. Citi Bike comparison

Code in support of the post ["When Are Citi Bikes Faster Than Taxis in New York City?"](http://toddwschneider.com/posts/taxi-vs-citi-bike-nyc/) lives in the `citibike_comparison/` folder

## Infrastructure Setup for Performing Analysis on AWS

### Hardware Requirement

1. AWS RDS Postgres Database.  The following instruction set was tested with RDS Postgres 10.4 
2. AWS EC2.  The instructions were tested with Ubuntu LTS 16.04.  It may be possible to perform data loading with Amazon Linux or Amazon Linux, but the program and set up process so difficult I gave up on Amazon Linux

### RDS Configuration
Taken from the AWS's documentation [Common DBA Tasks for PostgresSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html#Appendix.PostgreSQL.CommonDBATasks.PostGIS) Section Working with PostGIS
Log into the RDS database with psql client.  Issue the following commands against the RDS Postgres: 
```

--- enable the extensions 
create extension postgis;
create extension fuzzystrmatch;
create extension postgis_tiger_geocoder;
create extension postgis_topology;

--- verify tiger data are enabled. 
\dn

--- alter schema to transfer ownership of the schemas to the rds_superuser role.

alter schema tiger owner to rds_superuser;
alter schema tiger_data owner to rds_superuser;
alter schema topology owner to rds_superuser;

--- verify schema ownership are transfered
\dn


CREATE FUNCTION exec(text) returns text language plpgsql volatile AS $f$ BEGIN EXECUTE $1; RETURN $1; END; $f$;      

SELECT exec('ALTER TABLE ' || quote_ident(s.nspname) || '.' || quote_ident(s.relname) || ' OWNER TO rds_superuser;')
  FROM (
    SELECT nspname, relname
    FROM pg_class c JOIN pg_namespace n ON (c.relnamespace = n.oid) 
    WHERE nspname in ('tiger','topology') AND
    relkind IN ('r','S','v') ORDER BY relkind = 'S')
s; 

SET search_path=public,tiger; 


```

### Ubuntu EC2 Configuration
The version of Ubuntu should be LTS 16.04.  Other versions have not been very cooperative
[Source Instructions](http://trac.osgeo.org/postgis/wiki/UsersWikiPostGIS24UbuntuPGSQL10Apt)
Relevant installation instructions
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt xenial-pgdg main" >> /etc/apt/sources.list'
wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
sudo apt update


#excute these instructions one line at a time
sudo apt -y install postgresql-10
sudo apt -y install postgresql-10-postgis-2.4 
sudo apt -y install postgresql-10-postgis-scripts

#to get the commandline tools shp2pgsql, raster2pgsql you need to do this
sudo apt -y install postgis

# Install pgRouting 2.6 package 
sudo apt install postgresql-10-pgrouting

# to connect to the RDS Postgres Endpoint

psql -H <rds endpoint> -u <rds user> -d postgres

```


### Enable Ubuntu EC2 LTS 16.04 Xenial and GNOME UI 

Some data exploration tools, or GUI based SQL tools require a XServer or VNC Server based display.  Refer to this section if you are interested in using RStudio, Graphical Shapefile explorer.  Display R generated JPG etc, etc. 
Setting up the Ubuntu VNC & Desktop 
https://stackoverflow.com/questions/25657596/how-to-set-up-gui-on-amazon-ec2-ubuntu-server

Follow the 2 answers by sugunan + yuchien

##### Abstract from the sugunan + yuchien's answers: 
```
#Add a new user named awsgui
sudo useradd -m awsgui
sudo passwd awsgui
sudo usermod -aG admin awsgui

#modify sshd_config
sudo vim /etc/ssh/sshd_config # edit line "PasswordAuthentication" to yes
sudo /etc/init.d/ssh restart

#install vncserver and gnome desktop
sudo apt-get update
sudo apt install --no-install-recommends ubuntu-desktop
sudo apt install gnome-panel gnome-settings-daemon metacity nautilus gnome-terminal vnc4server

#open up Ubuntu's OS's firewall: 
sudo iptables -A INPUT -p tcp --dport 5901 -j ACCEPT

#configure user awsgui's vnc settings
su - awsgui
vncserver
vncserver -kill :1
vim /home/awsgui/.vnc/xstartup

#put this following section into the xstartup file 
#!/bin/sh

export XKL_XMODMAP_DISABLE=1
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
xsetroot -solid grey
vncconfig -iconic &

gnome-panel &
gnome-settings-daemon &
metacity &
nautilus &
gnome-terminal &
```

## Questions/issues/contact

liangcheng6@gmail.com 
