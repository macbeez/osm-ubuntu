# OSM Server with TileMill

## Description
Create an OSM Server and Run TileMill on Ubuntu 20.04

## System Setup
* Ubuntu 20.04
* 32 Core CPU (64 Threads)
* 64GB RAM
* 1TB for OS, 2TB for the database

## Software Packages Used
* PostgreSQL 12
* Postgis 3
* git
* acl

## References
* [How to Set Up OpenStreetMap Tile Server on Ubuntu 20.04](https://www.linuxbabe.com/ubuntu/openstreetmap-tile-server-ubuntu-20-04-osm)
* [Load OpenStreetMap Data to PostGIS (2020 Update)](https://blog.rustprooflabs.com/2020/01/postgis-osm-load-2020)
* [OSM Bright Ubuntu Quickstart](https://tilemill-project.github.io/tilemill/docs/guides/osm-bright-ubuntu-quickstart/)

## Installation

### Database Setup

Install the needed packages:
```
sudo apt install postgresql-12 postgresql-contrib postgis postgresql-12-postgis-3 git acl
```

As part of the default installation of PostgreSQL 12, a user called `postgres` is created. Also the default DB location is: 
```
/var/lib/postgresql/12/main
```
which resides on the same drive as the operating system. Since I anticipate the database size to be quite large once I start loading the map data, I will be changing the location to a seperate 2TB hard drive which I have configured to mount at the following location: 
```
/var/lib/postgresql/12/2TB1
```
Initially I did not think the mount point of the hard drive was important but soon realized that the `postgres` user needs access to the DB location and all parent folders leading up to it. 

Now we change permissions so that postgres users owns `/var/lib/postgresql/12/2TB1`
```
sudo chown -R postgres:postgres /var/lib/postgresql/12/2TB1
```
As the database configuration for loading map tiles is radically different from a standard production server, it's a good idea to create a seperate cluster instance called `osm_psql_db` specific for this purpose. A PSQL cluster is a collection of databases managed by a given instance

Let's create a new cluster called `osm_psql_db` in the new hard drive location we mounted earlier: 

```
sudo pg_createcluster 12 osm_psql_db -d /var/lib/postgresql/12/2TB1/osm_psql_db -p 5433
```
Notice the use of port 5433 for this instance versus the default 5432. This is done so that the default instance will not clash with the OSM database. 

Once the new cluster is created there should be a new service that's created, let's enable it. 

```
sudo systemctl enable postgresql@12-osm_psql_db
```
I ran into an issue here that took awhile to track down. When the script used by `pg_createcluster` creates the service, it assumes where the cluster is located based on how it expects everthing to be setup. Since I used a different hard drive, this setup doesn't follow the normal convention. Thus the new service file needs to modified to reflect the correct location. Otherwise, the cluster doesn't come online. 

Let's to edit the service to reflect the actual DB mount point
```
sudo systemctl edit --full postgresql@12-osm_psql_db.service
```

Change the following statement: 

```
RequiresMountsFor=/etc/postgresql/%I /var/lib/postgresql/%I
```

To    : 
```
RequiresMountsFor=/etc/postgresql/%I /var/lib/postgresql/12/2TB1/osm_psql_db
```
Some explanation: 

`%I` expands to "version/cluster" so the standard installation is expecting the cluster to be located at:

```
/var/lib/postgresql/%I ==> /var/lib/postgresql/12/osm_psql_db
```

After that change, let's reload daemon for good measure:
```
sudo systemctl daemon-reload
```

Start the new cluster
```
sudo pg_ctlcluster 12 osm_psql_db start
```

Now we can check to make sure we have 2 cluster instances online with:

```
pg_lsclusters
```

The output should read something like the following: 

```
Ver Cluster     Port Status Owner    Data directory                          Log file
12  main        5432 online postgres /var/lib/postgresql/12/main             /var/log/postgresql/postgresql-12-main.log
12  osm_psql_db 5433 online postgres /var/lib/postgresql/12/2TB1/osm_psql_db /var/log/postgresql/postgresql-12-osm_psql_db.log

```

Moving forward use the following to control the server: `sudo systemctl start|stop|restart postgresql@12-osm_psql_db`

### Create Database for Map Data

Okay now the cluster setup is done, let's start create a database in the `pqsl_osm_db` instance and configure it to accept map data. Switch users to `postgres` for this configuration:

```
sudo -u postgres -i
```

We will eventually create a database called `gis`, but we need a user spefically for it. Let's create a user called `osm` for this: 

```
createuser -p 5433 osm 
```
Notice the port command, we need to make sure our psql commmands are being directed at the right instance. For this example this will be port 5433. 

Now we create a database called `gis` and make `osm` the owner
```
createdb -p 5433 -E UTF8 -O osm gis
```

Next we create extensions in in the `gis` database to allow it to store map data. Extensions in PSQL are like modules. 
```
psql -p 5433 -c "CREATE EXTENSION postgis;" -d gis
psql -p 5433 -c "CREATE EXTENSION hstore;" -d gis
```

Set `osm` as the owner of the table in the `gis` database
```
psql -p 5433 -c "ALTER TABLE spatial_ref_sys OWNER TO osm;" -d gis
```

Exit from the postgres user and back to the regular user
```
exit
```

Create password for postgres user so that we can log in via md5 later on.
```
sudo -u postgres psql -p 5433 postgres
alter user postgres with password 'enter_password';
```

Replace `enter_password` with your desired password.

### Download the Map Data

Add `osm` (user) to the operating system so that the tile server can run as the `osm` user
```
sudo adduser --system --group osm
```

We should have a home directory for osm now, let's modify permissions and go to it

```
sudo setfacl -R -m u:system_user:rwx /home/osm/
cd /home/osm/
```

Replace `system_user` with your username, we need this permission (for now) to download the map data to this directory. Later we will change permissions so that `postgres` as access. 


Create a downloads folder for the next steps and move into it
```
mkdir Downloads
cd Downloads
```

Get the latest CSS map stylesheets
```
git clone https://github.com/gravitystorm/openstreetmap-carto.git
```

Get the latest map data of the whole planet
```
wget -c http://planet.openstreetmap.org/pbf/planet-latest.osm.pbf
```

This will take awhile depending on your internet bandwidth. At the present time, the file was 70GB, it will only grow larger with time. 

### Optimize the PostgreSQL Instance For Processing OSM Data

Optimize the PostgreSQL server - edit config file
```
sudo gedit /etc/postgresql/12/osm_psql_db/postgresql.conf
```

Change values to the following in the config file

* `data_directory = /var/lib/postgresql/12/2TB1/osm_psql_db` (Location of the database)

* `shared_buffers = 15GB` (Shared buffers should be 25% of the total RAM)

* `work_mem = 1GB`

* `maintenance_work_mem = 8GB`

* `effective_cache_size = 20GB`

* `max_parallel_workers_per_gather= 3` (Based on some forums, 3 is the most efficient)

These values have to be modified based on your system characteristics. If you have a different amount of CPU core or RAM, adjust accordingly.


Save and close the config file. Restart the postgresql server
```
sudo systemctl restart postgresql
```

By default, PostgreSQL will try to use huge pages in RAM. However Ubuntu by default does not allocate huge pages so let's fix that. 

Find out the Process ID (PID) for PostgreSQL and determine it's peak memory usage:
```
sudo head -1 /var/lib/postgresql/12/2TB1/osm_psql_db/postmaster.pid
```

Sample output:
```
7180
```

Then check the VmPeak value (peak memory) of this process ID.
```
grep ^VmPeak /proc/7180/status
```

Sample output for peak memory used by PostgreSQL:
```
VmPeak:	16188732 kB
```

Find the size of huge page in Linux
```
cat /proc/meminfo | grep -i huge
```

Sample Output:
```
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```


Calculate the number of huge pages needed. The equation is `Pages = VmPeak/size of huge page`:
```
Pages = 16188732/2048 = 7904.65
Pages = 7904 # Rounded down
```

Edit `/etc/sysctl.conf` file:
```
sudo nano /etc/sysctl.conf
```

Then add `vm.nr_hugepages = 7904` at the end of the document

Save and close the file, then apply the changes:

```
sudo sysctl -p
```
Check the meminfo again

```
cat /proc/meminfo | grep -i huge
```
We should notice that `HugePages_Total` and `HugePages_Free` are equal to 7094

Restart PostgreSQL server
```
sudo systemctl restart postgresql
```

### Importing Map Data to PostgreSQL

Install `osm2pgsql`
```
sudo apt install osm2pgsql
```

Change permisions to postgres user
```
sudo chown -R postgres:postgres /home/osm
```

Switch to postgres user
```
sudo -u postgres -i
```
We are almost ready to start importing osm data into the database, but we need to calculate the import cache size. We can calculate the cache size using the formula: 
```
(Total RAM - PostgreSQL shared_buffers) * 70% 
```
And then convert the result to MB:

`(64 GB - 15 GB) * 0.7 = 34.3 GB = 35123.2 MB`

Rounded down = 35000 MB

We will input this value following the `-C` option in the command below.

This is the final command that will load the osm data data into the database:

```
osm2pgsql --port 5433 --slim -d gis --drop \
--flat-nodes /home/osm/Downloads/nodes.cache \
--hstore --multi-geometry \
--number-processes 32 \
--tag-transform-script /home/osm/Downloads/openstreetmap-carto/openstreetmap-carto.lua \
--style /home/osm/Downloads/openstreetmap-carto/openstreetmap-carto.style \
-C 35000 /home/osm/Downloads/planet-latest.osm.pbf
```
This process can take a very long time, go ahead and walk away from your computer and enjoy life a little. 





















