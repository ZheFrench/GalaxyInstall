# GalaxyInstall - Prod

**Download**

***

 > git clone https://github.com/galaxyproject/galaxy/
 
 > cd galaxy
 
 > git checkout -b master origin/master
 
 # Pas de pip. Initialement Python 2.6. Besoin install python 2.7 via virtualenv.
 
 su 'user_root'
 brew install python
 /usr/local/Cellar/python/2.7.5/bin/python  # Path install nouveau python
 pip install virtualenv
 virtualenv .venv
 
 Install PostgreSQL

psql -U postgres galaxy_prod (password demandé)
create database galaxy template galaxy_prod 
Eteindre Galaxy pour permettre la copie de l'ancienne BDD

Galaxy.ini 
port=8003
Decommenter le host 127.0.0.1 et mettre l'ip du serveur
database_connection=postgresql://galaxy_prod_usr:galaxy@localhost/galaxy

source .ven/bin/activate (INUTILE)
Dans ./manage_db.sh
Rajouter :
export PYTHONPATH='';
Puis Rajouter :
python /usr/local/Cellar/python/2.7.5/python
./run.sh
python scripts/scramble.py -c config/galaxy.ini -e pysam (INUTILE)
pip install pysam
pip install certifi
Retour sur la bdd sqlLite ds sample.ini
Dans run.sh , ajout  export PYTHONPATH=''; CA MARCHE !
Repassage sur postgres :
sh manage_db.sh upgrade CA MARCHE !

Dans Apache :
sudo nano /private/etc/apache2/sites/000_anay_00_80_.conf
On a fermé Apache
On a fermé Galaxy Prod. 
On est tjs dans la merde.




