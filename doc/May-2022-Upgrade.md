## Upgrading from an old version of kobo-docker (before March 2019)

Current versions of kobo-docker require PostgreSQL 9.5 and MongoDB 3.4.
Additionally, **Redis is now the Celery broker**, and RabbitMQ is no longer
needed.

If you are running a version of kobo-docker that was last updated prior to
March 2019
(i.e. commit [`5c2ef02`](https://github.com/kobotoolbox/kobo-docker/commit/5c2ef0273339bee5c374830f72e52945947042a8) or older),
you need to upgrade your databases prior to using the current version of
kobo-docker (this repository) or
[kobo-install](https://github.com/kobotoolbox/kobo-install).

This is a step-by-step procedure to upgrade PostgreSQL and MongoDB.

### PostgreSQL

**Be sure to have enough space left on the host filesystem before upgrading.**
Check the size of the PostgreSQL database in  `.vols/db`, e.g. with
`sudo du -hs .vols/db`, and ensure you have _more_ than this amount of space
free.


1. Stop the containers

   ```
   docker-compose stop
   ```

2. Edit composer file `docker-compose.primary.backend.template.yml`

   Add this `- ./.vols/db14:/var/lib/postgresql/data/` below `- ./.vols/db:/var/lib/postgresql/data`. It should look like this.

   ```
      - ./.vols/db:/var/lib/postgresql/data
      - ./.vols/db14:/var/lib/postgresql/data14
   ```

3. Run a one-off `postgres` container

    1. With kobo-install
    ```shell
    user@computer:kobo-install$ python run.py -cb run --rm postgres bash  
    ```

    2. Without kobo-install
    ```shell
    docker-compose -f docker-compose.primary.backend.template.yml -f docker-compose.primary.backend.yml [-f docker-compose.primary.backend.override.yml] [-f docker-compose.primary.backend.custom.yml] run --rm postgres bash
    ```
    
6. Install PostgreSQL 14

    ```shell
    root@postgres:/# apt-get update
    root@postgres:/# apt-cache policy postgresql-14-postgis-3
    root@postgres:/# apt-cache policy postgis
    ```

    _Store the PostGIS version in a variable to use later_
    ```
    POSTGIS_VERSION_14=$(apt-cache policy postgresql-14-postgis-3|grep Candidate:|awk '{print $2}')
    ```

    ```
    apt-get install -y --no-install-recommends postgresql-14-postgis-3=${POSTGIS_VERSION_14} postgresql-14-postgis-3-scripts=${POSTGIS_VERSION_14} postgis postgresql-contrib-14
    apt-get upgrade
    ```

7. Init DB

    ```
    chown -R postgres:postgres /var/lib/postgresql/data14/
    su - postgres -c "/usr/lib/postgresql/14/bin/initdb -U $POSTGRES_USER --encoding=utf8 --locale=en_US.utf-8 -D /var/lib/postgresql/data14/"
    ```
    Results should look like this:

    > ```
    > Success. You can now start the database server using:
    >      /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/data14/ -l logfile start
    > ```

8. Start PostgreSQL 9.6 to ensure database has been initialized successfully

    ```
    su - postgres -c '/usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/data14/ start'
    ```
    > ```
    > ...
    > LOG:  database system is ready to accept connections
    > ```

    Press `enter` to go back to prompt.


7. Stop the server

	```
	su - postgres -c '/usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/data14/ stop -m fast'
	```

    > ```
	> ...
	> server stopped
    > ```


8. Upgrade Postgres 9.5

    ```
    apt-cache policy postgresql-9.5-postgis-3
    POSTGIS_VERSION_95=$(apt-cache policy postgresql-9.5-postgis-3|grep Candidate:|awk '{print $2}')
    apt-get install -y --no-install-recommends postgresql-9.5-postgis-3=${POSTGIS_VERSION_95} postgresql-9.5-postgis-3-scripts=${POSTGIS_VERSION_95}
    apt-get upgrade
    ```
 
9. Start PostgreSQL 9.5

    ```
    su - postgres -c '/usr/lib/postgresql/9.5/bin/pg_ctl -D /var/lib/postgresql/data/ start'
    ```
    Press `enter` to go back to prompt.
    ```
    /usr/lib/postgresql/9.5/bin/psql -U $POSTGRES_USER -d postgres
    ```

10. Upgrade PostGIS extension

    You may see some warnings `WARNING:  'postgis.backend' is already set and cannot be changed until you reconnect`. That's ok, you can keep going ahead.

    ```
    \c postgres;
    CREATE EXTENSION IF NOT EXISTS postgis;
    ALTER EXTENSION postgis UPDATE;
    CREATE EXTENSION IF NOT EXISTS postgis_topology;
    ALTER EXTENSION postgis_topology UPDATE;
    CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;
    CREATE EXTENSION IF NOT EXISTS postgis_tiger_geocoder;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
    
    \c template_postgis;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
    
    \c koboform;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
     
    \c kobocat;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
     
    \c kobo;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
     
    \q
    ```
    
11. Restore postgres role
    ```
    /usr/lib/postgresql/9.5/bin/psql -U "$POSTGRES_USER" -d postgres -q -c "CREATE USER postgres WITH SUPERUSER CREATEDB CREATEROLE REPLICATION BYPASSRLS ENCRYPTED PASSWORD '$POSTGRES_PASSWORD';"
    ```
    
    _If user already exists, you should see `ERROR:  role "postgres" already exists`._

12. Stop PostgreSQL 9.5 

    ```
    su - postgres -c '/usr/lib/postgresql/9.5/bin/pg_ctl -D /var/lib/postgresql/data/ stop -m fast'
    ```
    
13. Check everything is ok

    ```
    su - postgres -c "/usr/lib/postgresql/14/bin/pg_upgrade --check --old-datadir=/var/lib/postgresql/data/ --new-datadir=/var/lib/postgresql/data14/ --old-bindir=/usr/lib/postgresql/9.5/bin --new-bindir=/usr/lib/postgresql/14/bin -U $POSTGRES_USER"
    ```
    Results should look like this:

    > ```
    > Performing Consistency Checks
    > -----------------------------
    > Checking cluster versions                                   ok
    > Checking database user is the install user                  ok
    > Checking database connection settings                       ok
    > Checking for prepared transactions                          ok
    > Checking for reg* system OID user data types                ok
    > Checking for contrib/isn with bigint-passing mismatch       ok
    > Checking for presence of required libraries                 ok
    > Checking database user is the install user                  ok
    > Checking for prepared transactions                          ok
    >
    > *Clusters are compatible*
    > ```

14. Upgrade databases

    ```
    su - postgres -c "/usr/lib/postgresql/14/bin/pg_upgrade --old-datadir=/var/lib/postgresql/data/ --new-datadir=/var/lib/postgresql/data14/ --old-bindir=/usr/lib/postgresql/9.5/bin --new-bindir=/usr/lib/postgresql/14/bin -U $POSTGRES_USER"
    ```

    Results should like this:

    > ```
    > Upgrade Complete
    > ---------------
    > Optimizer statistics are not transferred by pg_upgrade so,
    > once you start the new server, consider running:
    > ./analyze_new_cluster.sh
    > ```

15. Edit composer file `docker-compose.backend.template.yml` again

    Locate

    ```
    postgres:
        image: postgis/postgis:9.5-2.5
    ```

    Change it to `postgis/postgis:14-3.2` and change volume `./.vols/db14` to point to `/var/lib/postgresql/data`.

    ```
    postgres:
        image: postgis/postgis:14-3.2
        ...
        volumes:
          # - ./.vols/db:/var/lib/postgresql/data
          - ./.vols/db14:/var/lib/postgresql/data
          ...
    ```

16. Update PosGIS extensions once again

    Start again a one-off  `postgres` container (see Point 3 for commands)

    Start the server
    ```
    su - postgres -c '/usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/data/ start'
    ```

    ```
    \c postgres;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
    
    \c template_postgis;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
    ex
    \c koboform;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
     
    \c kobocat;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
     
    \c kobo;
    ALTER EXTENSION postgis UPDATE;
    ALTER EXTENSION postgis_topology UPDATE;
    ALTER EXTENSION postgis_tiger_geocoder UPDATE;
    SELECT postgis_extensions_upgrade();
     
    \q
    ```
    
17. Prepare container to new version

    New version of `kobo-docker` creates `kobotoolbox` database with PostGIS extension at first run.
    To avoid trying to this at each subsequent start, a file is created with date of first run.
    We need to add this file because extensions have been installed during this migration.

    ```
    echo $(date) > /var/lib/postgresql/data/kobo_first_run
    echo "listen_addresses = '*'" >> /var/lib/postgresql/data/postgresql.conf
    echo "host    all             all             10.0.0.0/8            trust" >> /var/lib/postgresql/data/pg_hba.conf
    echo "host    all             all             172.0.0.0/8            trust" >> /var/lib/postgresql/data/pg_hba.conf
    echo "host    all             all             192.0.0.0/8            trust" >> /var/lib/postgresql/data/pg_hba.conf
    ```

    
18. Test if upgrade is successful

    Start your containers as usual.

    1. With kobo-install
        ```shell
        user@computer:kobo-install$ python run.py  
        ```

    Log into one user account        

19. Clean up

    If everything is ok, you can now delete data from `PostgreSQL 9.5`
    Stop `postgres` container.

    1. With kobo-install
        ```shell
        user@computer:kobo-install$ python run.py --stop  
        ```
    2. Rename folder
    ```shell
    user@computer:kobo-docker$ sudo rm -rf .vols/db
    user@computer:kobo-docker$ sudo mv .vols/db14 .vols/db
    ```

    Done!

### MongoDB

**Upgrading Mongo is easy and only implies a couple of stop/start.**

1. Upgrade to 3.0

    Stop the container: `docker-compose stop mongo`
    We need to change few lines in `docker-compose.yml`

    - Change image to `mongo:3.0`
    - Change `srv` to `data`

    ```
    mongo:
      image: mongo:3.0
      environment:
        - MONGO_DATA=/data/db
      ...
      volumes:
        - ./.vols/mongo:/data/db
    ```
    Then start the container: `docker-compose up --force-recreate mongo`

1. Upgrade to 3.2

    Stop the container: `docker-compose stop mongo`
    We only need to change the image in `docker-compose.yml`

    - Change image to `mongo:3.2`

    ```
    mongo:
      image: mongo:3.2
      ...
    ```
    Then start the container: `docker-compose up --force-recreate mongo`

1. Upgrade to 3.4

    Stop the container: `docker-compose stop mongo`
    We only need to change the image in `docker-compose.yml`

    - Change image to `mongo:3.4`

    ```
    mongo:
      image: mongo:3.4
      ...
    ```
    Then start the container: `docker-compose up --force-recreate mongo`

    Done!


You can now use latest version of kobo-docker (or use kobo-install)
