# GalaxyInstall - Prod

**1ère Mise à jour Juin 2015 du serveur 137**

https://wiki.galaxyproject.org/ReleaseAndUpdateProcess

**Download**

***

 > git clone https://github.com/galaxyproject/galaxy/
 
 > cd galaxy
 
 > git checkout -b master origin/master
 
**Installation de la dernière version de python et virtualenv**
 
 
***

 > su 'user_root'
 
 > brew install python
 
 > /usr/local/Cellar/python/2.7.10  # Path install nouveau python
 
 > pip install virtualenv
 
 > cd galaxy
 
 > virtualenv .venv

**Copie de l'ancienne BDD PostgreSQL de Galaxy** 

> su davidbaux (password)

> sudo launchctl unload /Library/LaunchDaemons/edu.psu.galaxy_dev.GalaxyServer.plist (password)

> psql -U _postgres galaxy_dev (password)

> CREATE DATABASE galaxy_dev0112015 template galaxy_dev;

> GRANT ALL PRIVILEGES ON DATABASE galaxy_dev0112015 TO galaxy_dev_user;

> Ctrl+D

> su davidbaux (password)

> sudo launchctl load /Library/LaunchDaemons/edu.psu.galaxy_dev.GalaxyServer.plist (password)

> mv galaxi.ini.sample galaxi.ini

> database_connection=postgresql://galaxy_dev_user:galaxy_dev0112015@localhost/galaxy_dev0112015 (fichier galaxy.ini)



**Structuration des "Handlers"** 

**[server:web1]** 

>             use = egg:Paste#http

>             port = 8081

>             host = 194.167.35.137

>             use_threadpool = True

>             \#threadpool_workers = 10

>             threadpool_kill_thread_limit = 10800


**[server:web2]**

>             use = egg:Paste#http

>             port = 8082

>             host = 194.167.35.137

>             use_threadpool = True

>             \#threadpool_workers = 10

>             threadpool_kill_thread_limit = 10800


**[server:main] NE PAS CHANGER LE NOM ** 

>         use = egg:Paste#http

>         port = 8083

>         host = 194.167.35.137

>         use_threadpool = True

>         \#threadpool_workers = 10

>         threadpool_kill_thread_limit = 10800


> cp ./config/job_conf.xml.sample_basic ./config/job_conf.xml

**Configuration de Apache pour le web balancing** 

http://jason.pureconcepts.net/2014/11/configure-apache-virtualhost-mac-os-x/

> Include /private/etc/apache2/vhosts/*.conf (Ajouter dans fichier /etc/apache2/httpd.conf)

> mkdir /etc/apache2/vhosts/

> touch  galaxy.dev.conf

>         <VirtualHost *:80>

>         DocumentRoot "/Library/WebServer/Documents"

>         ServerName localhost

>         ServerAdmin webmaster@localhost

>         ErrorLog "/private/var/log/apache2/galaxy.dev.local-error_log"

>         CustomLog "/private/var/log/apache2/galaxy.dev.local-access_log" common

>         RewriteEngine on

>         <Proxy balancer://galaxy>

>            BalancerMember http://localhost:8081

>            BalancerMember http://localhost:8082

>        </Proxy>

>        RewriteRule ^(.*) balancer://galaxy$1 [$P]

>        </VirtualHost>


 > sudo apachectl restart
 
 > GALAXY_RUN_ALL=1 ./run.sh --daemon
 
**Authentification des utilisateurs** 

 https://wiki.galaxyproject.org/Admin/Config/ApacheExternalUserAuth
 
 https://wiki.galaxyproject.org/Admin/Config/ExternalUserAuth
 
**Configuration FTP** 

https://wiki.galaxyproject.org/Admin/Config/UploadviaFTP

http://galacticengineer.blogspot.co.uk/2015/02/ftp-upload-to-galaxy-using-proftpd-and.html

https://github.com/jlhg/galaxy-preinstall/blob/master/proftpd-galaxy.conf

> su davidbaux (Password)

> createuser -SDR galaxyftp

> psql galaxy_dev0112015

> ALTER ROLE galaxyftp PASSWORD 'galaxy_dev';

> GRANT SELECT ON galaxy_user TO galaxyftp;

> Ctrl-D

> su davidbaux (Password)

**Telechargement proFTPD et compilation** 

> cd /usr/local/

> wget ftp://ftp.proftpd.org/distrib/source/proftpd-1.3.5a.tar.gz ( install wget par brew si absent)

> tar xfvz proftpd-1.3.5a.tar.gz

> cd proftpd-1.3.5a/

> ./configure --prefix=/usr/local/proftpd-1.3.5a/my_install --disable-auth-file --disable-ncurses --disable-ident --disable-shadow --enable-openssl --with-modules=mod_sql:mod_sql_postgres:mod_sql_passwd --with-includes=/usr/include/postgresql:/usr/local/Cellar/openssl/1.0.2c/include --with-libraries=/usr/lib/postgresql:/usr/local/Cellar/openssl/1.0.2c/lib 

> make 
> sudo make install

sudo nano /usr/local/proftpd-1.3.5a/my_install/etc/proftpd_welcome.txt
sudo mkdir /usr/local/proftpd-1.3.5a/my_install/var/log

> sudo /usr/local/proftpd-1.3.5a/my_install/sbin/proftpd --config /usr/local/proftpd-1.3.5a/my_install/etc/proftpd.conf -n -d 10

/usr/bin/postgres_real -D /var/pgsql -c listen_addresses=localhost -c log_directory=/Library/Logs/PostgreSQL -c log_filename=PostgreSQL.log -c log_lock_waits=on -c log_statement=ddl -c log_line_prefix=%t  -c logging_collector=on -c unix_socket_directory=/var/pgsql_socket -c unix_socket_group=_postgres -c unix_socket_permissions=0770

sudo su _postgres serveradmin start postgres
sudo serveradmin stop postgres
sudo serveradmin fullstatus postgres

psql -U _postgres -h localhost galaxy_dev0112015 MARCHE
psql -U _postgres -h 194.167.35.167 galaxy_dev0112015 MARCHE PAS

sudo nano /var/pgsql/postgresql.conf

Ajout dans  /usr/local/proftpd-1.3.5a/my_install/etc/proftpd.conf 
DelayEngine off ( écriture dans un repertoire qui necessitait de lancer FTP en root)

sudo chown -R 516:20 /usr/local/proftpd-1.3.5a/my_install/var/log/ (initialement root:wheel)
516(galaxy_dev_user) gid=20(staff)

drwxr-xr-x  2 root             wheel     68 25 nov 16:57 garbage
drwxr-x--x  8 galaxy_dev_user  staff    272 25 nov 17:27 log
-rw-r--r--  1 root             wheel  12608 25 nov 17:57 proftpd.delay
-rw-------  1 root             wheel      6 25 nov 17:57 proftpd.pid

http://www.proftpd.org/docs/howto/Compiling.html


**Instance Galaxy Interne + Cluster Calcul Externe** 

https://github.com/galaxyproject/pulsar

**Purge Library/Dataset/History -> Cron Job** 

https://wiki.galaxyproject.org/Admin/Config/Performance/ProductionServer
https://wiki.galaxyproject.org/Admin/Config/Performance/Purge%20Histories%20and%20Datasets
