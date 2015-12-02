# Galaxy Install - Prod - MAC OSX 10.7

**1ère Mise à jour Juin 2015 du serveur 137**

https://wiki.galaxyproject.org/ReleaseAndUpdateProcess

**Download**
***

 >         git clone https://github.com/galaxyproject/galaxy/
 
 >         cd galaxy
 
 >         git checkout -b master origin/master
 
 Note : Ouvert port FTP 21 / 8081 /8082 via Admin Serveur Interface
 
**Installation de la dernière version de python et virtualenv**

***

 >         su 'user_root'
 
 >         brew install python
 
 >         /usr/local/Cellar/python/2.7.10  # Path install nouveau python
 
 >         pip install virtualenv
 
 >         cd galaxy
 
 >         virtualenv .venv

**Copie de l'ancienne BDD PostgreSQL de Galaxy** 
***

Sur le 136 , problème de connexion acces denied.... obliger de faire un sudo même en adminuser alors que sur le 137 ce n'était pas nécéssaire. Avant toute chose dans postgres.

>         CREATE USER admin;
>         ALTER ROLE admin CREATEDB CREATEROLE SUPERUSER;
>         sudo dseditgroup -o edit -a $username_to_add -t user _postgres (jpense que juste ça suffit)

Pour info postgres.conf sont pas utilisés...Tout est dans un plist compiler au lancement quand on appelle serveradmin. J'ai pompé la conf du 137 sur le 136 qui était légèrement différente.

>         sudo nano /System/Library/LaunchDaemons/org.postgresql.postgres.plist

>         <?xml version="1.0" encoding="UTF-8"?>
>         <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
>         <plist version="1.0">
>         <dict>
>                 <key>Disabled</key>
>                 <true/>
>                 <key>GroupName</key>
>                 <string>_postgres</string>
>                 <key>Label</key>
>                 <string>org.postgresql.postgres</string>
>                 <key>OnDemand</key>
>                 <false/>
>                 <key>ProgramArguments</key>
>                 <array>
>                         <string>/usr/bin/postgres</string>
>                         <string>-D</string>
>                         <string>/var/pgsql</string>
>                         <string>-c</string>
>                         <string>listen_addresses=localhost</string>
>                         <!--<string>-c</string>
>                         <string>log_connections=on</string>-->
>                         <string>-c</string>
>                         <string>log_directory=/Library/Logs/PostgreSQL</string>
>                         <string>-c</string>
>                         <string>log_filename=PostgreSQL.log</string>
>                         <string>-c</string>
>                         <string>log_line_prefix=%t </string>
>                         <string>-c</string>
>                         <string>log_lock_waits=on</string>
>                         <string>-c</string>
>                         <string>log_statement=ddl</string>
>                         <string>-c</string>
>                         <string>logging_collector=on</string>
>                         <string>-c</string>
>                         <string>unix_socket_directory=/var/pgsql_socket</string>
>                         <string>-c</string>
>                         <string>unix_socket_group=_postgres</string>
>                         <string>-c</string>
>                         <string>unix_socket_permissions=0770</string>
>                 </array>
>                 <key>UserName</key>
>                 <string>_postgres</string>
>         </dict>
>         </plist>


La première partie qui suit n'est pas à executer. 
En fait , on veut avoir deux serveur en production sur lesquels on copie la base de données en production actuelle. Celle qui contient les utilisateurs (biologistes) . La dev ne contient que les utilisateurs informaticiens. Au départ j'avais fait le doc en bossant sur le 137. Donc cette partie on laisse tomber même si on réutilisera plus tard des commandes.

>         su davidbaux (password)

>         sudo launchctl unload /Library/LaunchDaemons/edu.psu.galaxy_dev.GalaxyServer.plist (password)

>         psql -U _postgres galaxy_dev (password)

>         CREATE DATABASE galaxy_dev0112015 template galaxy_dev;

>         GRANT ALL PRIVILEGES ON DATABASE galaxy_dev0112015 TO galaxy_dev_user;

>         Ctrl+D

>         su davidbaux (password)

>         sudo launchctl load /Library/LaunchDaemons/edu.psu.galaxy_dev.GalaxyServer.plist (password)

Ce qu'on doit faire , depuis le prod (pour récupérer tout les users) : 
>        pg_dump --host=localhost --username=postgres galaxy_prod > galaxy_prod.sql

Sur les autres serveurs : 
>        psql -U _postgres template1
>        CREATE DATABASE galaxy_112015;
>        CREATE USER galaxy_dev_user WITH PASSWORD 'XXXXX';
>        GRANT ALL PRIVILEGES ON DATABASE galaxy_112015 TO galaxy_dev_user;

>         sudo psql -U _postgres galaxy_112015 < galaxy_prod.sql

Important :
>         NOTE ERROR lOG : ( Je ne sais pas si ça va impacter la suite)
>         REVOKE
>         ERROR:  role "postgres" does not exist
>         ERROR:  role "postgres" does not exist

Oui en fait c'est important. Sur le 136/137 l'admin postgres s'appelle _postgres. Sur le 234 il s'appelle postgres.
Donc quand tu fais l'export , avant de recharger la base, il faut changer dans le schéma des tables postgres en _postgres comme ce qui suit :

>         --
>         -- Name: public; Type: ACL; Schema: -; Owner: **_postgres**
>         --

>         REVOKE ALL ON SCHEMA public FROM PUBLIC;
>         REVOKE ALL ON SCHEMA public FROM **_postgres**;
>         GRANT ALL ON SCHEMA public TO **_postgres**;
>         GRANT ALL ON SCHEMA public TO PUBLIC;

Sinon tu auras une belle erreur au moment du ./run.sh...aussi j'avais pas fait gaffe mais il faut aussi changer le owner dans le fichier .sql . (galaxy_prod_user 234 -> galaxy_dev_user 136)

Ensuite :

>         mv galaxi.ini.sample galaxi.ini
>         database_connection=postgresql://galaxy_dev_user:galaxy_dev112015@localhost/galaxy_dev112015 (fichier galaxy.ini)
>         ./run.sh

Et on repart pour une serie de problèmes : ( en théorie non car tu as fait ce que je t'ai dis avant )

>        File "/Users/galaxy_dev_user/galaxy/eggs/SQLAlchemy-1.0.8-py2.7-macosx-10.6-intel-ucs2.egg/sqlalchemy/engine/default.py", line 450, in do_execute
>            cursor.execute(statement, parameters)
>        DatabaseNotControlledError: (psycopg2.ProgrammingError) permission denied for relation migrate_version
>         [SQL: 'SELECT migrate_version.repository_id, migrate_version.repository_path, migrate_version.version \nFROM migrate_version >        \nWHERE migrate_version.repository_id = %(repository_id_1)s'] [parameters: {'repository_id_1': 'Galaxy'}]


Tu crois que c'est fini et noooooooon....

>        Exception: Your database has version '118' but this code expects version '129'.  Please backup your database and then migrate the schema by running 'sh manage_db.sh upgrade'.

>         ./manage_db.sh upgrade


**Structuration des "Handlers"** 
***

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

Important à faire pour utiliser le port 8083 comme job handler.

>         cp ./config/job_conf.xml.sample_basic ./config/job_conf.xml

**Configuration de Apache pour le web balancing** 
***

Voir plus bas pour la config complète au niveau de config apache proxy.

http://jason.pureconcepts.net/2014/11/configure-apache-virtualhost-mac-os-x/

>         Include /private/etc/apache2/vhosts/*.conf (Ajouter dans fichier /etc/apache2/httpd.conf)

>         mkdir /etc/apache2/vhosts/

>         touch  galaxy.dev.conf

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


 >         sudo apachectl restart
 
 >         GALAXY_RUN_ALL=1 ./run.sh --daemon
 
**Authentification des utilisateurs** 
***

 https://wiki.galaxyproject.org/Admin/Config/ApacheExternalUserAuth
 
 https://wiki.galaxyproject.org/Admin/Config/ExternalUserAuth
 
**Configuration FTP** 
***

https://wiki.galaxyproject.org/Admin/Config/UploadviaFTP

http://galacticengineer.blogspot.co.uk/2015/02/ftp-upload-to-galaxy-using-proftpd-and.html

https://github.com/jlhg/galaxy-preinstall/blob/master/proftpd-galaxy.conf

>         su davidbaux (Password)

>         createuser -SDR galaxyftp

>         psql galaxy_dev0112015

>         ALTER ROLE galaxyftp PASSWORD 'galaxy_dev';

>         GRANT SELECT ON galaxy_user TO galaxyftp;

>         Ctrl-D

>         su davidbaux (Password)

**Telechargement proFTPD et compilation** 
***

Pour compiler manuellement(c'est compliquer sur mac d'utiliser brew car il ne va pas vouloir de charger des modules avec la directive LoadModule) : 

http://www.proftpd.org/docs/howto/Compiling.html

>         cd /usr/local/

>         wget ftp://ftp.proftpd.org/distrib/source/proftpd-1.3.5a.tar.gz ( install wget par brew si absent)

>         tar xfvz proftpd-1.3.5a.tar.gz

>         cd proftpd-1.3.5a/

>         ./configure --prefix=/usr/local/proftpd-1.3.5a/my_install --disable-auth-file --disable-ncurses --disable-ident --disable-shadow --enable-openssl --with-modules=mod_sql:mod_sql_postgres:mod_sql_passwd --with-includes=/usr/include/postgresql:/usr/local/Cellar/openssl/1.0.2c/include --with-libraries=/usr/lib/postgresql:/usr/local/Cellar/openssl/1.0.2c/lib 

>         make 

>         sudo make install

>         sudo nano /usr/local/proftpd-1.3.5a/my_install/etc/proftpd_welcome.txt

>         sudo mkdir /usr/local/proftpd-1.3.5a/my_install/var/log

>          Permer de tester
>         sudo /usr/local/proftpd-1.3.5a/my_install/sbin/proftpd --config /usr/local/proftpd-1.3.5a/my_install/etc/proftpd.conf -n -d 10

>         sudo su _postgres serveradmin start postgres (marche pas en fait)
>         sudo serveradmin stop postgres
>         sudo serveradmin fullstatus postgres


Ca deconne, pourquoi ? Ca ma permis de comprendre :

>         psql -U _postgres -h localhost galaxy_dev0112015 MARCHE
>         psql -U _postgres -h X.X.X.X galaxy_dev0112015 MARCHE PAS

>         sudo nano /var/pgsql/postgresql.conf ( NON RIEN A CHANGER ICI, NON RIEN DE RIEN, JE NE REGRETTE RIEN)

On utilise pas ce postgresql.conf...je crois mais plutot ce qui va suivre...tadaaah

>         /System/Library/LaunchDaemons/org.postgresql.postgres.plist


Il fallait changer en fait X.X.X.X en localhost dans proftpd.conf .

**Ajout dans  /usr/local/proftpd-1.3.5a/my_install/etc/proftpd.conf**

_Deux choses m'ont sauver la vie :_

1- Dans Filezilla : choisir le mode actif. (Pourquoi ? Don't know ...si en fait ça vient de la plage configuré sur le serveur faudrait mettre les mêmes chiffres pour les ports passifs)
2- Dans le /var/pgsql/postgresql.conf, le 'home' crée par galaxy pour le FTP avait pour propriétaire le root.

http://www.proftpd.org/docs/directives/linked/config_ref_CreateHome.html
http://www.proftpd.org/docs/howto/CreateHome.html
https://www.howtoforge.com/community/threads/proftpd-and-dir-file-mask.26278/

>        # The correct directive is:
>         CreateHome on 700 dirmode 700 (j'ai rajouté les uid gid homegid)

 **LE SECRET C'EST DE FAIRE TOUT POUR QUE LE PROPRIETAIRE DU DOSSIER SOIT L'USER QUI A LANCE GALAXY**

Pour les tests , penser à virer le mail :
>         sudo rm -R /Users/galaxy_dev_user/galaxy/database/FTP/jpvillemin\@gmail.com/

Ca j'ai rien compris, c'était présent dans les configurations fournis par le GALAXY CREW.

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

Creation de lgm.proftpd.profpd.plist ds /usr/local/proftpf-1.3.5a/my_install :

>        sudo ln -sfv lgm.proftpd.proftpd.plist /Library/LaunchDaemons/lgm.proftpd.profpd.plist
>        sudo launchctl load /Library/LaunchDaemons/lgm.proftpd.profpd.plist

>        <?xml version="1.0" encoding="UTF-8"?>
>        <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd 
>        ">
>        <plist version="1.0">
>        <dict>
>        	<key>KeepAlive</key>
>        	<dict>
>        		<key>SuccessfulExit</key>
>        		<true/>
>        	</dict>
>        	<key>Label</key>
>        	<string>lgm.proftpd.proftpd</string>
>        	<key>ProgramArguments</key>
>        	<array>
>        		<string>/usr/local/proftpd-1.3.5a/my_install/sbin/proftpd</string>
>        		<string>-n</string>
>        	</array>
>        	<key>RunAtLoad</key>
>         <true/>
>         </dict>
>         </plist>

**QUOTAS** 
***

TODO
https://wiki.galaxyproject.org/Admin/DiskQuotas

**Config Apache Proxy** 
***

Dans /etc/apache2/httpd.conf :

>        	LoadModule xsendfile_module libexec/apache2/mod_xsendfile.so

Dans /galaxy.ini :

>        	# -- Advanced proxy features
>        	apache_xsendfile = True


**[filter:proxy-prefix]**

>        	use = egg:PasteDeploy#prefix
>        	prefix = /galaxy2015

**[app:main]**

>        	filter-with = proxy-prefix
>        	cookie_path = /galaxy2015

Dans /etc/apache2/vhosts/galaxy.dev.conf :

>        	<VirtualHost *:80>
>        	        DocumentRoot "/Library/WebServer/Documents"
>        	        RewriteEngine on

>        	        RewriteRule ^/galaxy2015$ /galaxy2015/ [R]
>        	        RewriteRule ^/galaxy2015/static/style/(.*) /Users/galaxy_dev_user/galaxy/static/june_2007_style/blue/$1 [L]
>        	        RewriteRule ^/galaxy2015/static/scripts/(.*) /Users/galaxy_dev_user/galaxy/static/scripts/packed/$1 [L]
>        	        RewriteRule ^/galaxy2015/static/(.*) /Users/galaxy_dev_user/galaxy/static/$1 [L]

>        	        RewriteRule ^/galaxy2015/qtrim/(.*) /Users/galaxy_dev_user/galaxy/tools/next_gen_conversion/qtrim/$1 [L]
>        	        RewriteRule ^/galaxy2015/star/(.*) $1 [L]
>        	        RewriteRule ^/galaxy2015/favicon.ico /Users/galaxy_dev_user/galaxy/static/favicon.ico [L]
>        	        RewriteRule ^/galaxy2015/robots.txt /Users/galaxy_dev_user/galaxy/static/robots.txt [L]

>        	        <Proxy balancer://galaxy2015>
>        	            BalancerMember http://X.X.X.X:8081
>        	            BalancerMember http://X.X.X.X:8082
>        	        </Proxy>

>        	        RewriteRule ^/galaxy2015(.*) balancer://galaxy2015$1 [P]

 >        	       <Location "/galaxy2015">
 >        	               # Compress all uncompressed content.
 >        	               SetOutputFilter DEFLATE
 >        	               SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png)$ no-gzip dont-vary
 >        	               SetEnvIfNoCase Request_URI \.(?:t?gz|zip|bz2)$ no-gzip dont-vary
 >        	               SetEnvIfNoCase Request_URI /history/export_archive no-gzip dont-vary

>        	                XSendFile on
>        	                XSendFileAllowAbove on
>        	        </Location>

>        	        <Location "/galaxy2015/static">
>        	                # Allow browsers to cache everything from /static for 6 hours
>        	                ExpiresActive On
>        	                ExpiresDefault "access plus 72 hours"

 >        	               XSendFile on
 >        	               XSendFileAllowAbove on
 >        	       </Location>

 >        	       <Directory "/Users/galaxy_dev_user/galaxy/database/files/">
 >        	               AllowOverride Fileinfo
 >        	       </Directory>

>        	</VirtualHost>

**Quotas & Config ** 
***
>        	[app:main]
>        	enable_quotas = True
>        	allow_user_dataset_purge = True

Ensuite, via l'interface connecté en user root, Preferences/


>        	require_login = True
>        	allow_user_creation = False
>        	allow_user_impersonation = True
>        	cleanup_job = always
>        	allow_user_deletion = True

**Rotate the logs** 
***
TODO

**Purge Library/Dataset/History -> Cron Job** 
***

TODO
https://wiki.galaxyproject.org/Admin/Config/Performance/ProductionServer
https://wiki.galaxyproject.org/Admin/Config/Performance/Purge%20Histories%20and%20Datasets

**Instance Galaxy Interne + Cluster Calcul Externe** 
***

https://github.com/galaxyproject/pulsar




