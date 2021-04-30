# Project_10 : Démarche
--------------------

# Table des matières

* [Sommaire](#sommaire)
* [Configuration du projet](#configuration-projet)
* [Configuration du serveur DigitalOcean](#configuration-serveur-DigitalOcean)
* [Configuration du serveur HTTP Nginx](#configuration-serveur-Nginx)
* [Configuration du serveur HTTP/WSGI Gunicorn](#configuration-serveur-Gunicorn)
* [Configuration de la gestion du serveur avec Supervisor](#configuration-Supervisor)
* [Configuration contrôle continu Travis](#configuration-contrôle-continu-Travis)
* [Configuration du monitoring sur DigitalOcean](#configuration-monitoring-DigitalOcean)
* [Configuration du monitoring sur DigitalOcean avec NewRelic](#configuration-monitoring-DigitalOcean-NewRelic)
* [Configuration du logging avec Sentry](#configuration-logging-Sentry)
* [Configuration d'un nom de domaine](#configuration-nom-domaine)
* [Automatisation de tâches avec Cron](#automatisation-Cron)

## Sommaire

L'objet de ce projet est de déployer sur un serveur de type IAAS un projet Django.
Pour y parvenir, nous devons configurer tout un environnement en lien avec Python et son framawork Django tout en assurant, une intégration continu, un contrôle des logs/évènements pouvant nuire au bon fonctionnement de l'application et une mise à jour automatique de la base de données.
Retrouvez ci-dessous notre environnement de travail:

- **Setup :** MacOs
- **Langage :** Python3
- **Framework :** Django2
- **Serveur IAAS :** Digital Ocean
- **Serveur HHTP :** Nginx
- **Serveur HTTP/WSGI :** Gunicorn
- **Contrôle continu :** Travis
- **Monitoring :** Digital Ocean & NewRelic
- **Logging :** Sentry
- **DNS :** 
- **Automatisation :** Cron

## Configuration-projet

### Séparation des environnements de DEVELOPPEMENT & PRODUCTION

*Projetc_8/off_project/settings
*  init.py*
*  production.py*

Pour mieux appréhender les différents environnemnts du projet, nous avons créé un module settings incorporant un fichier de configuration par défault (pour le développment) vie la fichier __init__.py ainsi qu'un second fichier venant mettre à jour ce même fichier mais avec des valeurs pour la production via le fichier production.py.

Pour de question de sécurité, le fichier *production.py* ne sera pas versionné.

### Configuration de l'environnement de PRODUCTION

*Project_8/off_project/settings/production.py*

```
SECRET_KEY = 'clésecrète deproduction'
DEBUG = False
ALLOWED_HOSTS = ['104.236.198.112']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'off',
        'USER': 'gontrand',
        'PASSWORD': 'password',
    }
}
```

- SECRET_KEY: Génération d'une nouvelle clé pour la production
- DEBUG: Doit être obligatoirement sur la valeur *false*.
- ALLOWED_HOSTS: IP de notre serveur Digital Ocean.
- DATABASES: Configuration pour le serveur Digital Ocean.

### Mis à jour du du repo distant Github

```
git add .
git commit -m "Settings project"
git push origin main
```

## Configuration-serveur-DigitalOcean

### Création d'un compte et d'un espace serveur

Digital Ocean permet de créer un serveur avec un nom de projet. 
Plusieurs configurations sont possibles, la plus significative a été la sélection du système d'exploitation (Ubuntu).

### Configuration sur serveur

Il est recommandé de créer un utilisateur pour effectuer toutes les manipulations sur le CLI de Digital Ocean.
Pour intégrer le projet dans de bonnes conditions nous avons du configurer la machine virtuelle. Cela passe par l'installation de:
- Python3 (langage)
```
sudo apt-get install python3-pip python3-dev
```
- PostgreSql (base de données)
```
sudo apt-get install libpq-dev postgresql postgresql-contrib
```
- Pipenv (environnement virtuel)
```
sudo apt install pipenv
```
(Erreur pour créer un environnement virtuel ? >> https://stackoverflow.com/questions/62820344/pipenv-fails-to-create-a-virtual-environment/62922485#62922485)

### Configuration d'une base de données

Création et paramêtrage d'un utilisateur ains que d'une base de données au sein du serveur.
La configuration doit être en accord avec le fichier de configuration settings/production.py

### Clone | Push du repo distant Github

Si première instace du projet:
```
git clone <lien github>
```

Mise à jour du projet:
```
git pull origin main
```

### Installation des dépendances & de l'environnement virtuel

```
pipenv shell
pipenv install
```

### Migrations des données

```
python3 manage.py migrate
python3 manage.py loaddata <fichier dump db>.json
```

## Configuration-serveur-Nginx

Nginx nous permet d'intégrer un serveur HTTP au sein du serveur Digital OCean.

### Installation

```
sudo apt-get install nginx
```

### Création d'un fichier de configuration

*/etc/nginx/sites-available/pur-beurre*
```
server {

    listen 80;
    server_name 104.236.198.112;
    root /home/gda/Project_8/;

    location /static {
         alias /home/gda/Project_8/staticfiles/;
    }

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
        if (!-f $request_filename) {
            proxy_pass http://127.0.0.1:8000;
            break;
        }
    }
}
```

- LISTEN: Ecoute le trafic sur le port 80
- SERVER_NAME: IP du serveur Digital OCean
- ROOT: Chemin de l'application sur le serveur
- LOCATION /static: Chemin des fichiers statiques
- LOCATION /: Redirige vers 127.0.0.1:8000



### Création d'un lien symbolique

Création du lien symbolique:
```
sudo ln -s /etc/nginx/sites-enabled/pur-beurre /etc/nginx/sites-enabled/
```

Recharger la configuration Nginx:
```
sudo service nginx reload
```

## Configuration-serveur-Gunicorn

Gunicorn nous permet de lire du code python au sein du serveur HTTP Nginx.

### Installation gunicorn

```
pipenv install gunicorn
```

### Tester gunicorn

S'assurer que gunicorn est effectif:
```
gunicron off_project.wsgi:application
```

## Configuration-Supervisor

Supervisor nous permet de gérer les processus au sein du serveur Digital Ocean.
Ici nous l'utilisons pour s'assurer que le processus gunicorn redémarrera en cas de besoin.

### Installation

```
sudo apt-get install supervisor
```

### Création d'un fichier de configuration

*../etc/supervisor/conf.d/pur-beurre-gunicorn.conf*
```
command = /home/gda/.local/share/virtualenvs/Project_8-6fpr4t17/bin/gunicorn off_project.wsgi:application
user = gda
directory = /home/gda/Project_8
autostart = true
autorestart = true
environment = DJANGO_SETTINGS_MODULE="off_project.settings.production"
```

- COMMAND: Chemin de l'executable de Gunicorn (```whereis gunicorn```)
- USER: Utilisateur de Digital Ocean
- DIRECTORY: Chemin du projet
- AUTORESTART: Oui, en cas d'arrêt
- ENVIRONNMENT: Nous permet de renseigner le nom du fichier de paramêtrage pour la production à destination du serveur HTTP/WSGI Gunicorn.

### Mise à jour de supervisor

Après toutes modifications du fichier vu ci-dessus:
```
sudo supervisorctl reread
sudo supervisorctl update
```

Afin de vérifier le bon fonctionnement de supervisor et de son script:
```
sudo supervisorctl status
```

Puis redémarrer le processus:
```
sudo supervisorctl restart pur-beurre-gunicorn
```

## Configuration-contrôle-continu-Travis

Travis est un service d'automatisation.
Dans notre, cas il nous permet de générer un script nous permettant d'automatiser des tests quand nous mettons à jour une branche de notre repo distant github. 
La sortie de ce script se solde par un status *success* ou *error* au sein de l'interface travis.com et de github.
Cette interface nous renseign une multitude d'informations quand aux logs, à l'hoistorique des scripts pour chaque éléments voulus.

### Configuration du compte travis

Après avoir créer un compte, nous avons mis en relation notre repo github avec son interface.
Cela permet à la fois de gérer le contrôle continu via l'interface de travis mais aussi de renvoyer des informations (telle que le status du script) sur le repo github.

### Implementation du fichier .travis.yml au sein du projet

Pour générer un script travis, il suffit de configurer un fichier .travis.yml au sein de notre projet.

*Project_8/.travis.yml*
```
language: python
python:
  - "3.8"

before_script:
  - pipenv install

branches:
  only:
    - dev

env: DJANGO_SETTINGS_MODULE="off_project.settings.travis"

services:
  - postgresql

script:
  - python3 manage.py test
```

Un ensemble de paramêtres peuvent être configurer via ce fichier. Ici ce qui nous intéresse, c'est générer le script ```python3 manage.py test``` à chaque push de la branche *dev* sur le repo github.

Après avoir effectué un ```git push origin dev```, nous obtenons sur github.com:
<img width="630" alt="Capture d’écran 2021-04-20 à 11 29 59" src="https://user-images.githubusercontent.com/52699053/115373135-dd68d480-a1cb-11eb-875b-64c2580a96d1.png">

Possibilité de visualiser les logs en cas d'erreur ou de succès sur travis.com:
<img width="1369" alt="Capture d’écran 2021-04-20 à 11 31 23" src="https://user-images.githubusercontent.com/52699053/115373416-2587f700-a1cc-11eb-86d2-fef311978f3e.png">

Possibilité de visualiser tout l'historique des logs générés par travis sur une même branche:
<img width="950" alt="Capture d’écran 2021-04-20 à 11 33 10" src="https://user-images.githubusercontent.com/52699053/115373617-57995900-a1cc-11eb-82ad-834ab4f43e12.png">



## Configuration-monitoring-DigitalOcean

Un ensemble de fonctionnalités permettent à l'utilisateur de visualiser par le biais de graphes, un ensemble de données issues du l'application en production.
Nous retrouvons des variables comme le processur, la mémoire et d'autres éléments. Toutefois ce qui nous importe vraiment, c'est d'être informé des anomalies du serveur. Pour cela nous créons des seuils à ne par dépasser qui en cas de réalisation, nous informe en temps réel. 

Il suffit pour cela de rendre sur l'onglet *Monitoring* et de sélectionner la ou les variables à monitorer avec un seuil à dépasser pour l'envoi d'une alerte sur sa propre boîte mail afin d'être informé.

## Configuration-monitoring-DigitalOcean-NewRelic

Un outil de monitoring plus puissant peut être couplé au projet en production. Il se nomme NewRelic. C'est avec une commande à faire courir dans la console du serveur et quelques instructions à suivre que l'on peut bénéficer d'un monitoring plus exaustif et modulable que celui fournit par Digital Ocean.

Ci-dessous, un apercu du dashboard de l'interface NewRelic pour ce projet en production:
<img width="1311" alt="Capture d’écran 2021-04-20 à 18 50 55" src="https://user-images.githubusercontent.com/52699053/115434865-78cc6a80-a209-11eb-80d5-a1548ceca0e4.png">

## Configuration-logging-Sentry

Cet outil quant à lui nous permet de visualiser ce qui se passe dans le code exactement contrairement au monitoring.
En plus de nous fournir des logs, l'application SAAS nous renvoie tout l'environnement du llg comme le navigateur de l'utilisateur, l'url et ainsi de suite.

### Implémentation de Sentry au sein du projet

Au sein de l'interface sentry, créer un nouveau projet avec la technologie utilisée.
Sentry renverra la documentation pour implémenter ce service au sein de votre application web.
Une fois implémenté, ce service sera opérationnel.

*home/gda/Project_8/off_project/settings/production.py*
```
sentry_sdk.init(
    dsn="https://dd13a8ddd7f7416e8ea47a8844308b61@o574849.ingest.sentry.io/5726357",
    integrations=[DjangoIntegration()],

    # Set traces_sample_rate to 1.0 to capture 100%
    # of transactions for performance monitoring.
    # We recommend adjusting this value in production.
    traces_sample_rate=1.0,

    # If you wish to associate users to errors (assuming you are using
    # django.contrib.auth) you may enable sending PII data.
    send_default_pii=True
)
```

## Configurattion-nom-domaine
Ìl est possible de configurer un nom de domaine qui a été acheté au préalable au sein de Digital Ocean.
Dans notre cas, nous n'avons pas opté pour cette solution.

## Automatisation-Cron

### Installation *django-crontab*

```
pipenv install django-crontab
```

*/Project_8/off_project/settings/init.py*
Ajouter à la variable INSTALLED_APPS += 'django_crontab'

### Création d'un fichier cron.py

Ce fichier nous permet d'intégrer des méthodes qui seront appelés par différents cron.
Dans notre cas nous créons une méthode qui appelle un script pour mettre à jour la base de données:

*Project_8/off/cron.py*
```
from .scripts import update_database

def update_database_cron():
    update_database()
```
### Configuration du fichier production.py (côté serveur)

*Project_8/off_project/settings/production.py*
```
CRONJOBS = [
     ('0 0 * * MON', 'off.cron.update_database_cron', '> update-database-$(date +\%Y\%m\%d\%H\%M\%S).log'),
     ('0 6 * * MON', 'django.core.management.call_command', ['dumpdata'], {'indent': 4}, "> dump-data-$(date +\%Y\%m\%d\%H\%M\%S).json"),
]
```
Tous les lundis à minuit > mis à jour de la base de données avec stockage des logs.
Tous les lundis à 6h00, dump de la base de données mis à jour sur un fichier json.

Ajout scripts au sein de la machine:
```
python3 manage.py crontab add  
```







