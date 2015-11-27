# Galaxy Install - Prod - MAC OSX 10.7

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

Pour compiler manuellement(c'est compliquer sur mac d'utiliser brew car il ne va pas vouloir de charger des modules avec la directive LoadModule) : 

http://www.proftpd.org/docs/howto/Compiling.html

> cd /usr/local/

> wget ftp://ftp.proftpd.org/distrib/source/proftpd-1.3.5a.tar.gz ( install wget par brew si absent)

> tar xfvz proftpd-1.3.5a.tar.gz

> cd proftpd-1.3.5a/

> ./configure --prefix=/usr/local/proftpd-1.3.5a/my_install --disable-auth-file --disable-ncurses --disable-ident --disable-shadow --enable-openssl --with-modules=mod_sql:mod_sql_postgres:mod_sql_passwd --with-includes=/usr/include/postgresql:/usr/local/Cellar/openssl/1.0.2c/include --with-libraries=/usr/lib/postgresql:/usr/local/Cellar/openssl/1.0.2c/lib 

> make 

> sudo make install

> sudo nano /usr/local/proftpd-1.3.5a/my_install/etc/proftpd_welcome.txt

> sudo mkdir /usr/local/proftpd-1.3.5a/my_install/var/log

>  Permer de tester
> sudo /usr/local/proftpd-1.3.5a/my_install/sbin/proftpd --config /usr/local/proftpd-1.3.5a/my_install/etc/proftpd.conf -n -d 10

> sudo su _postgres serveradmin start postgres
> sudo serveradmin stop postgres
> sudo serveradmin fullstatus postgres

Ca deconne, pourquoi ? Ca ma permis de comprendre.
> psql -U _postgres -h localhost galaxy_dev0112015 MARCHE
> psql -U _postgres -h X.X.X.X galaxy_dev0112015 MARCHE PAS

> sudo nano /var/pgsql/postgresql.conf ( NON RIEN A CHANGER ICI, NON RIEN DE RIEN, JE NE REGRETTE RIEN)

Il fallait changer en fait X.X.X.X en localhost dans proftpd.conf .

Ajout dans  /usr/local/proftpd-1.3.5a/my_install/etc/proftpd.conf 

Deux choses m'ont sauver la vie :

1- Dans Filezilla : choisir le mode actif. (Pourquoi ? Don't know)
2- Dans le /var/pgsql/postgresql.conf, le 'home' crée par galaxy pour le FTP avait pour propriétaire le root.

http://www.proftpd.org/docs/directives/linked/config_ref_CreateHome.html
http://www.proftpd.org/docs/howto/CreateHome.html
https://www.howtoforge.com/community/threads/proftpd-and-dir-file-mask.26278/

>        # The correct directive is:
>         CreateHome on 700 dirmode 700 (j'ai rajouté les uid gid homegid)

LE SECRET C'EST DE FAIRE TOUT POUR QUE LE PROPRIETAIRE DU DOSSIER SOIT L'USER QUI A LANCE GALAXY.

Pour les tests , penser à virer le mail
> sudo rm -R /Users/galaxy_dev_user/galaxy/database/FTP/jpvillemin\@gmail.com/

Ca j'ai rien compris, c'était présent dans les configurations fournis par le GALAXY CREW
http://www.proftpd.org/docs/howto/Limit.html

>        # ProFTPD configuration for Galaxy FTP

>        ServerName                      "FTP use with Galaxy'"
>        ServerType                      standalone
>        DefaultServer                   on
>        Port                            21
>        UseIPv6                         off

>        # Add to launch proftpd not as root
>        DelayEngine off

>        # Umask 022 is a good standard umask to prevent new dirs and files
>        # from being group and world writable. 0 - ALL PERMISSIONS 2 READ and Execute 7 NO PERM
>        Umask                           077

>        #TimeoutNoTransfer               600
>        #TimeoutStalled                  600
>        #TimeoutIdle                     1200

>        # To prevent DoS attacks, set the maximum number of child processes
>        # to 30.  If you need to allow more than 30 concurrent connections
>        # at once, simply increase this value.  Note that this ONLY works
>        # in standalone mode, in inetd mode you should use an inetd server
>        # that allows you to limit maximum number of processes per service
>        # (such as xinetd).
>        MaxInstances                    30

>        # Set the user and group under which the server will run. 
>        User                            galaxy_dev_user
>        Group                           staff

>        # Cosmetic changes, all files belongs to this user/group
>        # FakeGroup semlbe marcher...pas le 1er ... User/Group precedement suffise non ?
>        #DirFakeUser on galaxy_dev_user
>        #DirFakeGroup on staff

>        # Some logging formats
>        LogFormat            default "%h %l %u %t \"%r\" %s %b"
>        LogFormat            auth    "%v [%P] %h %t \"%r\" %s"
>        LogFormat            write   "%h %l %u %t \"%r\" %s %b"

>        # Log files
>        TransferLog                    /usr/local/proftpd-1.3.5a/my_install/var/log/xfer.log
>        SQLLogFile                     /usr/local/proftpd-1.3.5a/my_install/var/log/sql.log
>        ServerLog                      /usr/local/proftpd-1.3.5a/my_install/var/log/proftpd.log
>        SystemLog                      /usr/local/proftpd-1.3.5a/my_install/var/log/system.log

>        # Log file/dir access
>        ExtendedLog                     /usr/local/proftpd-1.3.5a/my_install/var/log/access.log    WRITE,READ write
>        # Record all logins
>        ExtendedLog                     /usr/local/proftpd-1.3.5a/my_install/var/log/auth.log      AUTH auth
>        ExtendedLog                     /usr/local/proftpd-1.3.5a/my_install/var/log/auth.log      AUTH auth

>        # Say Hello
>        DisplayConnect  /usr/local/proftpd-1.3.5a/my_install/etc/proftpd_welcome.txt

>        # Passive port range for the firewall (la ya peut être un truc à faire , en attendant on se met en active mode sur filezilla)
>        PassivePorts                    30000 40000

>        # To cause every FTP user to be "jailed" (chrooted) into their home
>        # directory, uncomment this line.
>        DefaultRoot ~

>        # Automatically create home directory if it doesn't exist
>        CreateHome                      on 755 dirmode 700 uid 516 gid 20 homegid 20

>        # Normally, we want files to be overwriteable.
>        AllowOverwrite          on

>        # (ADDED) Allow users to resume interrupted uploads
>        AllowStoreRestart               on

>        # J'ai foutrement rien compris mais http://www.proftpd.org/docs/howto/Limit.html

>        # Bar use of SITE CHMOD by default
>        #<Limit SITE_CHMOD>
>        #  DenyAll
>        #</Limit>
>        # Bar use of RETR (download) since this is not a public file drop
>        #<Limit RETR>
>        #   DenyAll
>        #</Limit>

>        # Do not authenticate against real (system) users
>        AuthPAM                         off

>        # Common SQL authentication options
>        SQLEngine                       on
>        SQLPasswordEngine               on
>        SQLBackend                      postgres
>        SQLConnectInfo                  galaxy_dev0112015@localhost:5432 galaxyftp galaxy_dev
>        SQLAuthenticate                 users

>        # Set up mod_sql to authenticate against the Galaxy database
>        SQLAuthTypes                    PBKDF2
>        SQLPasswordPBKDF2               SHA256 10000 24
>        SQLPasswordEncoding             base64

>        # For PBKDF2 authentication
>        # See http://dev.list.galaxyproject.org/ProFTPD-integration-with-Galaxy-td4660295.html
>        SQLPasswordUserSalt             sql:/GetUserSalt

>        # Define a custom query for lookup that returns a passwd-like entry.
>        # The file created use the uid in the sql request
>        # But uid is reattributed because it's below SQLMinUserUID set by default to 999
>        # The user launching galaxyis galaxy_user uid 516 so we se SQLMinUSERUID to 500 , this way it won't be modified

>        SQLMinUserUID 500
>        SQLUserInfo                     custom:/LookupGalaxyUser
>        SQLNamedQuery                   LookupGalaxyUser SELECT "email, (CASE WHEN substring(password from 1 for 6) = 'PBKDF2' THEN substring(password from 38 for 69) ELSE password END) AS password2,516,20,'/Users/galaxy_dev_user/galaxy/datab$

>        # Define custom query to fetch the password salt
>        SQLNamedQuery                   GetUserSalt SELECT "(CASE WHEN SUBSTRING (password from 1 for 6) = 'PBKDF2' THEN SUBSTRING (password from 21 for 16) END) AS salt FROM galaxy_user WHERE email='%U'"

>        ###################
>        #For SHA1 passwords
>        ###################
>        # Set up mod_sql/mod_sql_password - Galaxy passwords are stored as hex-encoded SHA1
>        #SQLAuthTypes                    SHA1
>        #SQLPasswordEncoding             hex

>        # An empty directory in case chroot fails
>        #SQLDefaultHomedir               /usr/local/proftpd-1.3.5a/my_install/var/garbage/

>        # Define a custom query for lookup that returns a passwd-like entry. Replace 512s with the UID and GID of the user running the Galaxy server
>        #SQLUserInfo                     custom:/LookupGalaxyUser
>        #SQLNamedQuery                   LookupGalaxyUser SELECT "email,password,516,20,'/Users/galaxy_dev_user/galaxy/database/FTP/%U','/bin/bash' FROM galaxy_user WHERE email='%U'


**Instance Galaxy Interne + Cluster Calcul Externe** 

https://github.com/galaxyproject/pulsar

**Purge Library/Dataset/History -> Cron Job** 

https://wiki.galaxyproject.org/Admin/Config/Performance/ProductionServer
https://wiki.galaxyproject.org/Admin/Config/Performance/Purge%20Histories%20and%20Datasets
