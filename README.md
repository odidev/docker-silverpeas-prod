#docker-silverpeas-prod

A Dockerfile that produces a container that will run [Silverpeas 6][silverpeas].

Be caution, Silverpeas 6 is currently in development and it is not production-ready. Silverpeas 6.0-SNAPSHOT is the development version of 6.0.
[silverpeas]: http://www.silverpeas.org

## Image creation

To create an image of Silverpeas 6 you have to specify the exact versions of Silverpeas and of Wildfly used for running it in this order:
```
$ ./build.sh 6.0 10.0.0
```
This builds an image containing Silverpeas 6.0 (not yet available) and Wildfly 10.0.0 with the tag `silverpeas-prod:6.0`. The versions passed as argument have to match the versions of Silverpeas and Wildfly available in the Web; indeed, Silverpeas and Wildfly are downloaded from their respective project Web site.

## Container running

Silverpeas 6 image depends on a one of the following database:
* the open-source and recommended PostgreSQL database system,
* MS-SQLServer database system.

So, to run Silverpeas 6 you have to first to run a container with one of the above database. 

### With a database from a container

In [Docker Hub][dockerhub], you will find a lot of images embedding PostgreSQL.
For example, with a container from the [official PostgreSQL docker image][docker-postgresql]:
```
$ docker run --name postgresql -d \
  -e POSTGRES_PASSWORD="mysecretpassword" \
  -v /var/lib/postgresql/data:/var/lib/postgresql/data \
  postgres"
```
This will start a PostgreSQL instance that will be initialized with a superuser `postgres` and with as password `mysecretpassword`. The directory with the database files is mounted on the host so it couldn't be lost when upgrading PostgreSQL to a new version. Once PostgreSQL is running, a database for Silverpeas, named `Silverpeas` for example, can be created and optionally a user with administrative rights on that database can be also added (recommended way).

Finally, you can run Silverpeas 6 by specifying the required database access data. For example:
```
$ docker run --name silverpeas -p 8080:8000 -d \
  -e DB_NAME="Silverpeas" \
  -e DB_USER="postgres" \
  -e DB_PASSWORD="mysecretpassword" \
  -e DB_NAME="Silverpeas"
  -v /var/log/silverpeas:/opt/silverpeas/log \
  -v /var/lib/silverpeas:/opt/silverpeas/data \
  --link postgresql:database \
  silverpeas-prod:6.0
```
This image exposes the 8000 port at which Silverpeas is listening and this port is here mapped to the 8080 port of the host.
It is recommended to mount the volumes `/opt/silverpeas/log` and `/opt/silverpeas/data` from the container. Indeed, the first volume contains the log files produced by the Silverpeas application whereas the second volume contains all the data that are managed by the users in Silverpeas. So, you can easily have a glance at the Silverpeas activity and you can switch to the next version of Silverpeas without losing the data. (Using a [Data Volume Container][data-volume] to map `/opt/silverpeas/log` and `/opt/silverpeas/data` is a better solution.)

In the case to have a finer configuration of Silverpeas, you can create directly a `config.properties` file with, additionally to the other configuration parameters, the database access parameters, and then you pass it to the container:
```
$ docker run --name silverpeas -p 8080:8000 -d \
  -v /etc/silverpeas/config.properties:/opt/silverpeas/configuration/config.properties
  -v /var/log/silverpeas:/opt/silverpeas/log \
  -v /var/lib/silverpeas:/opt/silverpeas/data \
  --link postgresql:database \
  silverpeas-prod:6.0
```
where `/etc/silverpeas/config.properties` is the Silverpeas global configuration file on the host.

[dockerhub]: https://hub.docker.com/
[docker-postgresql]: https://hub.docker.com/_/postgres/
[data-volume]: https://docs.docker.com/engine/userguide/containers/dockervolumes/

### With a database on the host

In the case PostgreSQL is running on the host, to enable the connection to the database from the container, you have to configure PostgreSQL accordingly:
* In the file postgresql.conf, edit the parameter `listen_addresses` with as value the address of the docker gateway (in my case 172.17.0.1)
```
listen_addresses = 'localhost,172.17.0.1'
```
* In the file pg_hba.conf, add an entry for the docker subnetwork
```
host    all             all             172.17.0.0/16           md5
```
* Don't forget to restart PostgreSQL

Finally, you can run now Silverpeas 6:
```
$ docker run --name silverpeas -p 8080:8000 -d \
  --add-host=database:172.17.0.1 \
  -v /etc/silverpeas/config.properties:/opt/silverpeas/configuration/config.properties \
  -v /var/log/silverpeas:/opt/silverpeas/log \
  -v /var/lib/silverpeas:/opt/silverpeas/data \
  silverpeas-prod:6.0
```
where `database` is the hostname on which run PostgreSQL and that is referred as such in the `config.properties`.

### Using a Data Volume Container

To manage more effectively the data produces by an application, the Docker team recommends to use a Data Volume Container. In Silverpeas, there are three types of data produced by the application:
* the logging stored in `/opt/silverpeas/log`,
* the user data and those produced by Silverpeas from the user data in `/opt/silverpeas/data`,
* the user domains and the domain authentication definitions in respectively `/opt/silverpeas/properties/org/silverpeas/domains` and `/opt/silverpeas/properties/org/silverpeas/authentication`.

The directories `/opt/silverpeas/log`, `/opt/silverpeas/data`, and `/opt/silverpeas/properties` are all defined as volumes in the docker image and then can be mounted on the host.

To define a Data Volume Container for Silverpeas:
```
$ docker create --name silverpeas-store \
  -v /opt/silverpeas/data \
  -v /opt/silverpeas/log \
  -v /opt/silverpeas/properties \
  -v /etc/silverpeas/config.properties:/opt/silverpeas/configuration/properties \
  silverpeas-prod:6.0 \
  /bin/true
```

Then to mount the volumes in the Silverpeas container:
```
$ docker run --name silverpeas -p 8080:8000 -d \
  --link postgresql:database \
  --volumes-from silverpeas-store \
  silverpeas-prod:6.0
``` 

If you have to customize the settings of Silverpeas or add, for example, a new database definition in Wildfly, then specify these settings with the Data Volume Container:
```
$ docker create --name silverpeas-store \
  -v /opt/silverpeas/data \
  -v /opt/silverpeas/log \
  -v /opt/silverpeas/properties \
  -v /etc/silverpeas/config.properties:/opt/silverpeas/configuration/properties \
  -v /etc/silverpeas/CustomerSettings.xml:/opt/silverpeas/configuration/silverpeas/CustomerSettings.xml \
  -v /etc/silverpeas/my-datasource.cli:/opt/silverpeas/configuration/jboss/my-datasource.cli \
  silverpeas-prod:6.0 \
  /bin/true
```

## Container activity

You can follow the activity of Silverpeas by watching the logs generated in the mounted `/opt/silverpeas/log` directory.

The output of Wildfly is redirected into the container standard output and as such it can be watched as following:
```
$ docker logs -f silverpeas
```

