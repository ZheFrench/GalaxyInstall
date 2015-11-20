# GalaxyInstall - Prod 234


 git clone https://github.com/galaxyproject/galaxy/
 cd galaxy
 git checkout -b master origin/master
 
 # Pas de pip. Initialement Python 2.6. Besoin install python 2.7 via virtualenv.
 
 su 'user_root'
 brew install python
 /usr/local/Cellar/python/2.7.5/bin/python  # Path install nouveau python
 pip install virtualenv
 virtualenv .venv
 
 Install PostgreSQL
