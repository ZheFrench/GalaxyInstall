# GalaxyInstall - Prod

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

> CREATE DATABASE galaxy_prod0112015 template galaxy_dev;

> GANT ALL PRIVILEGES ON DATABASE galaxy_dev0112015 TO galaxy_dev_user;

> Ctrl+D

> su davidbaux (password)

> sudo launchctl load /Library/LaunchDaemons/edu.psu.galaxy_dev.GalaxyServer.plist (password)

> mv galaxi.ini.sample galaxi.ini

> database_connection=postgresql://galaxy_dev_user:galaxy_dev0112015@localhost/galaxy_dev0112015

(fichier galaxy.ini)

> GALAXY_RUN_ALL=1 sh ./run.sh --daemon

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


**[server:main_job_handler]** 

>         use = egg:Paste#http

>         port = 8083

>         host = 194.167.35.137

>         use_threadpool = True

>         \#threadpool_workers = 10

>         threadpool_kill_thread_limit = 10800

**Configuration de Apache pour le web balancing"** 

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
