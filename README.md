MISE EN PRODUCTION
CONNEXION AU SERVEUR

Serveur de connexion :

    user: <user>
    password: <ask_for_password>

DEPLOIEMENT MANUEL DE L'APPLICATION

Pré-requis :

    Installer Docker
    Paramétrer le VPS
    Paramétrer les records DNS
    Paramétrer le reverse proxy Traefik
    Avoir une application fullstack qui tourne correctement en local

Déploiement sur le serveur de production

On va récupérer les images Docker depuis notre repository sur notre serveur :

docker image pull nicolashureau/crafted_by_spa:latest
docker image pull nicolashureau/crafted_by_api:latest
docker pull postgres

On va ensuite créer un fichier docker-compose.yml dans `/home :
N.B. : pour "db", bien mettre les même credentials qu'en local

  services:

    traefik:
      image: traefik:v2.2
      networks:
        - traefik-proxy
      command:
        - --api.insecure=true
        - --api.dashboard=true
        - --api.debug=true
        - --log.level=DEBUG
  
        - --providers.docker=true
        - --providers.docker.exposedbydefault=false
        - --providers.docker.network=traefik-proxy
  
        - --entrypoints.http.address=:80
        - --entrypoints.https.address=:443
  
        - --certificatesresolvers.tls.acme.tlschallenge=true
        - --certificatesresolvers.tls.acme.email=contact@mon-domaine.com
        - --certificatesresolvers.tls.acme.storage=/letsencrypt/acme.json
      ports:
        - "80:80"
        - "443:443"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./letsencrypt:/letsencrypt"
      labels:
        - "traefik.enable=true"
  
        - "traefik.http.middlewares.http_to_https.redirectscheme.scheme=https"
  
        - "traefik.http.middlewares.auth.basicauth.users=test:$$2y$$12$$iPbYdyAEZbyTYjEEU5Bneu/GKrciqcwYtZqZJwUxHsjgfrRXrBEEW"
  
        - "traefik.http.routers.monitor.rule=Host(`traefik.elaboradopor.com`)"
        - "traefik.http.routers.monitor.entrypoints=http"
        - "traefik.http.routers.monitor.middlewares=http_to_https@docker"
  
        - "traefik.http.routers.monitor-secured.rule=Host(`traefik.elaboradopor.com`)"
        - "traefik.http.routers.monitor-secured.service=api@internal"
        - "traefik.http.routers.monitor-secured.entrypoints=https"
        - "traefik.http.routers.monitor-secured.tls=true"
        - "traefik.http.routers.monitor-secured.tls.certresolver=tls"
        - "traefik.http.routers.monitor-secured.middlewares=auth@docker"
  
    crafted_by_db:
      extends:
        file: database.yml
        service: database
      networks:
        - traefik-proxy
  
    crafted_by_api:
      extends:
        file: backend.yml
        service: backend
      networks:
        - traefik-proxy
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.api-secured.rule=Host(`api.elaboradopor.com`)"
        - "traefik.http.routers.api-secured.entrypoints=https"
        - "traefik.http.routers.api-secured.tls=true"
        - "traefik.http.routers.api-secured.tls.certresolver=tls"
  
    crafted_by_spa:
      extends:
        file: frontend.yml
        service: frontend
      networks:
        - traefik-proxy
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.spa-secured.rule=Host(`elaboradopor.com`)"
        - "traefik.http.routers.spa-secured.entrypoints=https"
        - "traefik.http.routers.spa-secured.tls=true"
        - "traefik.http.routers.spa-secured.tls.certresolver=tls"

  networks:
    traefik-proxy:
      name: traefik-proxy

database.yml : 

      version: '3'
      services:
        database:
          image: postgres:latest
          container_name: crafted_by_db
          restart: always
          ports:
            - 5432:5432
          volumes:
            - ./crafted_by_db:/var/lib/postgresql/data
          environment:
            - POSTGRES_PASSWORD=password
            - POSTGRES_USER=nicolas
            - POSTGRES_DB=crafted_by
            - PGDATA=/var/lib/postgresql/data/

backend.yml : 

      version: '3'
      services:
        backend:
          image: nicolashureau/crafted_by_api:latest
          container_name: crafted_by_api
          ports:
            - 8080:80
          environment:
            DB_HOST: crafted_by_db

frontend.yml : 

      version: '3'
      services:
        frontend:
          image: nicolashureau/crafted_by_spa:v1.0.4
          container_name: crafted_by_spa
          ports:
            - 5173:80
          environment:
            VITE_API_ENDPOINT: https://api.elaboradopor.com/api/

traefik.yml :

      api:
        dashboard: true
        debug: true
      
      entryPoints:
        http:
          address: ":80"
        https:
          address: ":443"
      
      providers:
        docker:
          endpoint: "unix:///var/run/docker.sock"
          exposedByDefault: false
      log:
        level: "ERROR"
        filePath: "log/error.log"
        format: "json"
      
      accessLog:
        filePath: "log/access.log"
        format: "json"
      
      certificatesResolvers:
        http:
          acme:
            email: nicolas.hureau@campus-numerique.fr
            storage: acme.json
            httpChallenge:
              entryPoint: http

Une fois cette étape réalisée :

docker compose up -d

Ensuite, se logger dans le conteneur du backend :

sudo docker exec -it <id_du_conteneur> sh

et exécuter les commandes pour migrer et seeder la bdd :

php artisan migrate

php artisan db:seed

On peut controller que tout fonctionne correctement en se rendant sur les url de l'app :

    https://dashboard.elaboradopor.net
    https://nginx.elaboradopor.net
    https://elaboradopor.com
    https://backend.elaboradopor.net/api

