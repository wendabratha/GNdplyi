# GeoNetwork 3.2.x deployment and configuration guide

---
#### Server setup :
- Debian jessie 8.7
- Apache 2.4.10
- Apache Tomcat 7.0.56
- JVM openJDK version "1.8.0_121"
- GeoNetwork 3.2.1
- PostgreSQL 9.4
- PostGIS 2.1

Before installing new packages it is recommended to update repositories.
`$ sudo apt-get update`

---
### Apache HTTP configuration
Install with command
`$ sudo apt-get install apache2`

Open in browser http://localhost or http://\*my-domain-name\*. You should see the contents of the __`index.html`__ file (located in _/var/www/html_).

Check apache version

    $ /usr/sbin/apache2 -v
    Server version: Apache/2.4.10 (Debian)
    Server built: <Date> <Time>

#### Configuring a new site

Configuration for a new site will be created from the modified default apache file which can serve as a backup. Copy file _/etc/apache/site-available/000-default.conf_

`$ sudo cp -a /etc/apache/site-available/000-default.conf /etc/apache/site-available/geonetwork.conf`

Now edit the file

`$ sudo nano /etc/apache/site-available/geonetwork.conf`

Add two lines inside the section *`<Virtualhost>`*.

    ProxyPass / http://localhost:8080/geonetwork
    ProxyPassEnable / http://localhost:8080/geonetwork

#### Enabling site
Enable the geonetwork site on apache server :
`$ sudo a2ensite geonetwork`
`$ sudo service apache2 reload`

Once finished with the [Database configuration](#database-configuration-H2) chapter the site will become functional.

#### OpenJDK install
>In Debian jessie openJDK 1.7 will be installed by specifying `default-jre` when executing the installation command. The version will be  
>**java version "1.7.0_121"**

Java 8 is required for GeoNetwork 3.2.x. Therefore the **debian packports** repository needs to be added in the repositories list.

In the _/etc/apt/sources.list_ add the line below (or optionally create a new file __`my-jessie-sources.list`__ in _/etc/apt/sources.list.d/_ and insert it here instead).

    deb http://ftp.debian.org/debian jessie-backports main
<sub>more @ [debianorg][debianorg]</sub>

Update the repository sources list.
`$ sudo apt-get update`

To install the correct version run :
`$ sudo apt-get -t jessie-backports install openjdk-8-jre`

and check the java version.

    $ java -version
    openjdk version "1.8.0_121"
    OpenJDK Runtime Environment (build 1.8.0_121-8u121-b13-1~bpo8+1-b13)
    OpenJDK 64-Bit Server VM (build 25.121-b13, mixed mode)
   
### Tomcat 7 installation

Install with :
`$ sudo apt-get install tomcat7`

Verify tomcat version :

    $ sudo /usr/share/tomcat7/bin/version.sh
    Using CATALINA_BASE: ...
    Using CATALINA_HOME: ...
    Using CATALINA_TMPDIR: ...
    Using JRE_HOME: ...
    Using CLASSPATH: ...
    Server version: Apache Tomcat/7.0.56 (Debian)
    ...

If the command does not output the result above, verify if tomcat is up and running. Opening the URL http://localhost:8080/ in browser should respond with the message **"It Works !"** with the rest of the default page in tomcat's *ROOT* folder.

In case you receive a _"Can’t establish a connection to the server at localhost:8080."_ that means tomcat server has not started properly.

Start the tomcat service :

`$ sudo service tomcat7 start`

### Tomcat heap size configuration

Allocating memory resources while tomcat service starts up will allow GeoNetwork to launch correctly. The heap size is set by passing appropriate parameters to *CATALINA_OPTS* variable using a script, defining tomcat's environment.
<sub>more @ [geonetwork documentation][gn docs 1]</sub>

Change directory to _/usr/share/tomcat7/bin/_ and create the file __`setenv.sh`__ with the contents below to adjust the variable parameters :
`$ sudo cat > setenv.sh`
`CATALINA_OPTS="$CATALINA_OPTS -Xms256m -Xmx1024m -XX:MaxPermSize=512m"`
`<Ctrl+D>`
<sub>\> see terrance's [example script][tsnyder] for other details</sub>

The script has to be an executable file.

`$ sudo chmod +x setenv.sh`
`$ sudo service tomcat7 restart`

---
### GeoNetwork deployment

Start deploying GeoNetwork by downloading geonetwork.war :

`$ cd ~`
`$ wget "geonetwork download link *"`
<sub>* https://sourceforge.net/projects/geonetwork/files/GeoNetwork_opensource/v3.2.1/geonetwork.war</sub>

Copy the downloaded file to the __webapps__ folder _/var/lib/tomcat7/webapps_.

`s sudo cp -a geonetwork.war /var/lib/tomcat7/webapps`

Tomcat will deploy the file in _/var/lib/tomcat7/webapps/geonetwork_. You might have to restart the tomcat service for the changes to take effect.

`$ sudo service tomcat7 restart`

### GeoNetwork configuration
#### Data directory

Externalizing the data directory is recommended to simplify the application upgrades. 
<sub>see geonetwork [documentation][gn docs 2]</sub>

However, if you are is not required to modify **meta-data schemas** keeping them in the default directory is probably the best choice as they update with each geonetwork version upgrade.
<sub>see geonetwork [mailing list][nabble osgeo 2] and [documentation][gn docs 3]</sub>

Externalising is achieved by adding new parameters in the variable *CATALINA_OPTS*.

+ **-Dgeonetwork.dir** for data directory
+ **-Dgeonetwork.schema.dir** for schemas
+ and as well as **-Dgeonetwork.codelist.dir** for thesaurus

The content of `setenv.sh` should now be :

    CATALINA_OPTS="$CATALINA_OPTS -Xms256m -Xmx1024m -XX:MaxPermSize=512m -Dgeonetwork.dir=/usr/share/tomcat7/gn_data_dir -Dgeonetwork.schema.dir=/var/lib/tomcat7/webapps/geonetwork/WEB-INF/data/config/schema_plugins"

The folder **gn\_data\_dir ** is the location where geonetwork saves the details about configuration, metadata and resources, search functionalities and optionally spatial data information <sup>**[1]**</sup>.

On linux systems users do not have writting permissions for system files and folders, except for the folder */home/&lt;user&gt;*. Granting writing permissions is covered in the database configuration [chapter](#granting-permissions-to-a-directory) as the database will be placed outside the geonetwork default location for the same reason - easier integration in case of application version update.
[1]: http://geonetwork-opensource.org/manuals/trunk/eng/users/maintainer-guide/installing/customizing-data-directory.html#structure-of-the-data-directory

### Database configuration - H2

H2 is a default GeoNetwork database, instantiated at the first launch. GeoNetwork will probably not start due to restriction of the writing permissions in the folder _/var/lib/tomcat7_.
<sub>more info @ [mailling list][nabble osgeo 1] and [geonetwork github][gn github]</sub>

The error can be inspected in __`geonetwork.log`__ file located in _/var/lib/tomcat7/logs/_.

#### Granting directory permissions

Tomcat's installation folder permissions can be changed, but it is safer to keep the ownership as **root**. Better option would be to use a directory such as _`/usr/share/tomcat7/.geonetwork`_.

Webapps running in tomcat server are executed and handled by the tomcat user and writing to the chosen location is possible by granting proper permissions to tomcat's home directory :

`$ sudo chgrp tomcat7 /usr/share/tomcat7`
`$ sudo chmod g+w /usr/share/tomcat7`

#### JDBC properties

Accordingly, the database location has to be configured in **`jdbc.properties`** file found in */webapps/geonetwork/WEB_INF/config-db/*. Modify the parameter _`jdbc.database`_ to :

    jdbc.database=~/.geonetwork/geonetwork

<sub>more @ [gis stex][gis stex]</sub>

At this point the geocatalogue should be properly configured and ready for metadata exchange. Try if everything works as planned @ http://localhost or http://\*my-domain-name\*.

---

Geonetwork database connection and database node configuration is managed in 2 principal configuration files. Next section describes how to plug to an external DBMS such as PostGreSQL.

### PostGreSQL installation

Install PostGreSQL.

`$ sudo apt-get update`
`$ sudo apt-get install postgresql-9.4`

If you're getting *"perl: warning: Setting locale failed."* errors while installing packages the postgresql installation on a debian remote server may fail. [These guides][askubuntu] should be taken in consideration.

The working solution for this installation was *"Stopping forwarding locale from the client"*. The line `SendEnv LANG LC_*` in the **local** _/etc/ssh/ssh_config_ file was commented out.

The command below should display if the installation was successful.

    $ ps -ef | grep postgre

Return the postgres status to see if the service is running.

    $ sudo service postgres status
    9.4/main (port 5432): online

Folow the [PostgreSQL how to](../divnotes/blob/master/psql_how_to.md) notes to set up a database user and the connection information details.

Now the database node can be configured properly for PostgreSQL. Open the **`srv.xml`** file in _/var/lib/tomcat7/webapps/geonetwork/WEB-INF/config-node_ 

`$ sudo nano srv.xml`

and comment out h2 and set postgresql node or change the connection information :
```xml
<!-- <import resource="../config-db/h2.xml"/> -->
<import resource="../config-db/postgres.xml"/>
```
Then modifiy the connection information in the jdbc.properties file :
```
jdbc.username=<postgresql_username>
jdbc.password=<postgresql_password>
jdbc.database=<postgresql_databasename>
```
GeoNetwork will from now on use the PostGreSQL DBMS.

### PostGIS installation

PostGIS spatial component is recommended to assure good database performance at large quantities of metadata in the catalogue.

`$ sudo apt-get install postgis`
<sub> more @ [geo-solutions][geo-solutions]</sub>

***
Next chapters will describe certain migration scenarios. Soon more to come ...

[debianorg]: https://backports.debian.org/Instructions/ "Backports instructions"
[gn docs 1]: http://geonetwork-opensource.org/manuals/trunk/eng/users/maintainer-guide/installing/installing-from-war-file.html "installing-from-war-file"
[tsnyder]: https://gist.github.com/terrancesnyder/986029
[gn github]: https://github.com/geonetwork/core-geonetwork/issues/1360 "issues"
[nabble osgeo 1]: http://osgeo-org.1560.x6.nabble.com/Install-Geonetwok-Ubuntu-16-4-Tomcat-tp5265471p5265690.html "Install-Geonetwok-Ubuntu-16-4-Tomcat"
[askubuntu]: https://askubuntu.com/questions/144235/locale-variables-have-no-effect-in-remote-shell-perl-warning-setting-locale-f#144448 "setting-locale-variables"
[gn docs 2]: http://geonetwork-opensource.org/manuals/trunk/eng/users/maintainer-guide/installing/customizing-data-directory.html "customizing-data-directory"
[nabble osgeo 2]: http://osgeo-org.1560.x6.nabble.com/GN-upgrade-tp5311933p5312451.html "GN-upgrade"
[gn docs 3]: http://geonetwork-opensource.org/manuals/trunk/eng/users/maintainer-guide/installing/customizing-data-directory.html#advanced-data-directory-configuration "advanced-data-directory-configuration"
[gis stex]: https://gis.stackexchange.com/questions/53478/geonetwork-install-issues "geonetwork-install-issues"
[geo-solutions]: http://demo.geo-solutions.it/share/bev/doc/gn-install/geonet_db.html#spatial-index "spatial-index"