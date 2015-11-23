# GalaxyInstall - Prod

**Download**

***

 > git clone https://github.com/galaxyproject/galaxy/
 
 > cd galaxy
 
 > git checkout -b master origin/master
 
**Install dernière version de python et virtualenv**
 
 
***

 > su 'user_root'
 > brew install python
 > /usr/local/Cellar/python/2.7.10  # Path install nouveau python
 > pip install virtualenv
 > cd galaxy
 > virtualenv .venv

**Copie ancienne bdd PostgreSQL de Galaxy et mise à jour** 

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




