# Galeries Ponthé

### Installation

Exécuter
```
./install.sh
```

### Dévelopement

Activer l'environnement de développement Python :
```
source venv/bin/activate
```

Lancer l'application
```
FLASK_APP=ponthe flask run
```
Pour le rendre disponible sur le réseau ajouter `--host=0.0.0.0` (écoute toutes les IPs publiques)

## ORM

### Migration
Générer les migrations dans *web/migrations/versions* :
```
cd web/app
export FLASK_APP=ponthe
flask db migrate
flask db upgrade
```

### Fixtures
Supprimer la BDD, les images / photos et charger les fixtures :
```
cd web/app
ponthe/manager.py load_fixtures
```

## Mettre en production

Juste faire `deploy.sh` pour faire une sauvegarde de la bdd et redéployer. Pour des objectifs spécifique voir ci-dessous.

Pour rebuild l'image web après modification des fichiers copiés :
```
docker-compose up --build
```

Après modification du .env :
```
docker-compose rm db
docker-compose up
```

Configurer le proxy pass du VPS vers le container docker avec Nginx et Certbot :
Installer Certbot : https://certbot.eff.org/lets-encrypt/ubuntuxenial-nginx.html
```
sudo cp nginx/ponthe.enpc.org.conf /etc/nginx/sites-available/ponthe.enpc.org
sudo ln -s /etc/nginx/sites-available/ponthe.enpc.org /etc/nginx/sites-enabled/ponthe.enpc.org
sudo certbot --nginx --noninteractive --agree-tos --email root@clubinfo.enpc.fr -d ponthe.enpc.org
sudo systemctl reload nginx
```
Charger les données initiales (catégories) dans le container:
```
docker-compose exec web python ponthe/manager.py load_data
```

Charger les fixtures dans le container :
```
docker-compose exec web python ponthe/manager.py load_fixtures
```

Créer les comptes des nouveaux élèves :
Demander le csv *mon_csv.csv* de création de compte de uPont au KI et faire
```
scp -P 7502 mon_csv.csv ponthe@ponthe.enpc.org:accounts.csv
docker-compose exec web python ponthe/manager.py create_accounts
```

## Ajouter des fichiers aux galeries :

Les consulter :
```
ssh ponthe@localhost -p 7502
```

En ajouter l'event TOSS à l'année 2018 :
```
scp -P 7502 -r TOSS ponthe@ponthe.enpc.org:waiting_zone/2018/
```
où TOSS est un répertoire de photos et vidéos

## Mobile app

L'application sera téléchargeable en se rendant à partir d'un téléphone android à l'url https://ponthe.enpc.org/apk/djfh2qp71qhsLK82Z92tyHNn-v1.0.0-release.apk
Run the below command to avoid ENOSPC :
```
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

### ANDROID

Installer Android Studio.
Quelques variables d'environnement sont nécessaires :
```
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/platform-tools
```
Compiler et installer l'application sur un android :
* Activer le mode développeur puis le débogage USB sur le téléphone
* Connecter le téléphone par un cable USB à l'ordinateur
* Installer l'application React en installant *yarn* puis en lançant `yarn` dans *mobile/*
* Lancer `react-native run-android` depuis *mobile/*

Cela installe l'application sous forme de .apk sur le téléphone et exécute les commande pour connecter le téléphone au serveur Meteor sur l'ordinateur par USB, du type :
```
adb -s 9b52109f reverse tcp:8081 tcp:8081
adb -s 9b52109f shell am start -n fr.ponthe.galeries/fr.ponthe.galeries.MainActivity
```

Le serveur peut être relancé sans recompilation (si il n'y a pas de changement de dépendance) avec :
```
react-native start
```
Si le port 8081 est déjà occuper par le serveur lancé par `react-native run-android`, on peut kill le processus
```
sudo lsof -i :8081
kill -9 <PID>
```
ou écouter sur un autre port :
```
react-native start --port=8088
```

La connexion entre le téléphone et l'ordinateur peut se perdre, même sans déconnexion du cable USB, inutile de relancer 'run-android',
il suffit de faire :
```
adb reverse tcp:8081 tcp:8081
```

On peut rediriger les logs android sur le terminal avec :
```
react-native log-android
```
ou dans Google Chrome à l'url http://localhost:8081/debugger-ui/. Attention, seul un debugger peut écouter en même temps

Pour debugger le système app + serveur sans avoir à redéployer le serveur docker,
il suffit de faire tourner le serveur en local avec `python run.py` et de mettre l'adresse de ce serveur dans la variable url de l'app.
Par exemple sur mon réseau local, j'ai l'adresse :
```
const url = 'http://192.168.1.43:5000/api'
```

Les icones suivants sont disponibles : https://oblador.github.io/react-native-vector-icons/.

### IPhone

Projet non-configuré car il est nécessaire d'avoir un Mac et un iPhone, un appel au don est lancé !

### Release

Documentation : http://facebook.github.io/react-native/docs/signed-apk-android.html#content
L'apk est signé au format PKCS12.

Générer l'apk à `android/app/build/outputs/apk/app-release.apk` avec:
```
cd android && ./gradlew assembleRelease
```
Le mettre en ligne à https://ponthe.enpc.org/apk/{token}-v{version number}-release.apk
Le *token* de 24 lettres est aléatoire pour que personne n'ayant pas l'adresse puisse installer l'app, le numéro de version est de la forme [v?.?.?](https://semver.org) :
```
scp -P 7502 app/build/outputs/apk/app-release.apk ponthe@ponthe.enpc.org:apk/{token}-v{version number}-release.apk
```

Après désinstallation de la version debug, on installe la version release sur le téléphone avec :
```
react-native run-android --variant=release
```
ou en allant sur le lien de l'apk;

Tha launcher icons and more can be generated from an image with http://romannurik.github.io/AndroidAssetStudio/.

### Distributing on a private fdroid app repository

#### Serveur

Follow https://f-droid.org/en/docs/Setup_an_F-Droid_App_Repo/

#### Client

Avec le téléphone android aller sur https://f-droid.org et télécharger F-Droid.
Puis ouvrer F-Droid, allez dans *Paramètre*, puis *Gestion des dépôts*, décochez tout et ajouter en appuyant sur le *+* en haut à droite
le répôt à l'adresse du serveur setup à la partie précédente.
Puis installer l'application GaleriesPonthé.

## TODO

* SQLAlchemy example : https://jeffknupp.com/blog/2014/01/29/productionizing-a-flask-application/
* Générer la doc avec sphinxcontrib-httpdomain :
* set up linters : https://jeffknupp.com/blog/2016/12/09/how-python-linters-will-save-your-large-python-project/
* Docker : http://www.patricksoftwareblog.com/using-docker-for-flask-application-development-not-just-production/

## Ressources :

* http://flask.pocoo.org/docs/0.12/tutorial/introduction/
* http://www.patricksoftwareblog.com/flask-tutorial/
* https://jeffknupp.com/blog/2014/01/29/productionizing-a-flask-application/
* http://freemiumdownload.com/downloads/lifestyle-blog-free-bootstrap-template/
Public domain code snippets :
* http://flask.pocoo.org/snippets/
Slugification :
* http://flask.pocoo.org/snippets/5/
Password salt hashing:
* http://flask.pocoo.org/snippets/54/

## SQLAlchemy
On utilise un modèle de "joined table inheritance"
* http://flask-sqlalchemy.pocoo.org/2.3/customizing/
On pourrait rendre implicite les ids en prolongeant le modèle de SLQAlchemy
* http://docs.sqlalchemy.org/en/latest/orm/inheritance.html
Ambigüités de jointure dûs à l'héritage :
* http://docs.sqlalchemy.org/en/latest/orm/join_conditions.html

## TODO

* Thumbnail : https://github.com/silentsokolov/flask-thumbnails
* Map : https://developers.google.com/maps/documentation/javascript/tutorial
* OpenId Connect : https://github.com/ory/hydra
* Template de galerie
* WTForm
* Uppy : uppy-server et dashboard locales
* React Native
  * Map : https://github.com/react-community/react-native-maps
  * Firebase :
https://medium.com/@salonikogta/beginners-guide-to-implementing-push-notifications-in-android-e896ef54b831
https://firebase.google.com/pricing/
https://firebase.google.com/docs/web/setup?authuser=0
https://console.firebase.google.com/u/0/project/enpc-d6bbe/notification
  * (Web notif https://developers.google.com/web/fundamentals/codelabs/push-notifications/)

## Visualiser l'histoire du dépôt

Installer gource:
```
sudo apt-get install gource
```
Puis exécuter :
```
resolution="1900x1000"
scale="0.5"

gource --viewport $resolution --colour-images --stop-at-end -s $scale -a 0.01 --max-files 0 --highlight-users --highlight-dirs --key
--title "L'histoire de Pzartech OCR" -i 0
```