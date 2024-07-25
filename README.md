# Symfony Test API Setup

Ce guide décrit comment initialiser un nouveau projet Symfony dans un conteneur Docker temporaire pour l'initialisation et configurer l'environnement de développement continu avec Docker Compose.

## 1. Initialisation d'un nouveau projet Symfony

### 1.1 Créer un Dockerfile d'initialisation `Dockerfile.init`

Le `Dockerfile.init` est utilisé pour créer un conteneur temporaire qui initialise un projet Symfony.

```Dockerfile
# Dockerfile d'initialisation pour créer le projet Symfony

# Utiliser une image PHP CLI avec Composer et Symfony CLI
FROM php:8.2-cli

# Installer les extensions PHP nécessaires et autres dépendances
RUN apt-get update && apt-get install -y \
    libpq-dev \
    git \
    unzip \
    && docker-php-ext-install pdo pdo_pgsql

# Installer Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Installer Symfony CLI
RUN curl -sS https://get.symfony.com/cli/installer | bash
RUN mv /root/.symfony*/bin/symfony /usr/local/bin/symfony

# Définir le répertoire de travail
WORKDIR /app
```

### 1.2 Créer un fichier Docker Compose d'initialisation `docker-compose.init.yml`
Ce fichier Docker Compose est utilisé pour lancer le conteneur d'initialisation.

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.init
    volumes:
      - .:/app
```

### 1.3 Exécuter le conteneur pour initialiser le projet Symfony
Utilisez Docker Compose pour lancer le conteneur d'initialisation et créer le projet Symfony dans un répertoire temporaire, puis déplacez les fichiers et supprimez le répertoire temporaire.

```bash
docker-compose -f docker-compose.init.yml run --rm app /bin/bash -c "symfony new temp --version='7.1.*' --no-git && mv temp/* temp/.* . && rmdir temp"
```

**Explication :**

- **docker-compose -f docker-compose.init.yml run --rm app** : Cette commande utilise le fichier `docker-compose.init.yml` pour lancer un conteneur Docker temporaire nommé `app`. Le flag `--rm` permet de supprimer le conteneur après son exécution.

- **/bin/bash -c "symfony new temp --version='7.1.*' --no-git && mv temp/* temp/.* . && rmdir temp"** : 
  - **symfony new temp --version='7.1.*' --no-git** : Crée un nouveau projet Symfony dans un répertoire temporaire nommé `temp` en spécifiant la version `7.1.*` de Symfony, sans initialiser de dépôt Git.
  - **mv temp/* temp/.* .** : Déplace tous les fichiers et dossiers (y compris les fichiers cachés) du répertoire `temp` vers le répertoire courant (`.`).
  - **rmdir temp** : Supprime le répertoire temporaire `temp` après le déplacement des fichiers.

### 1.4 Configurer Symfony pour utiliser PostgreSQL

Modifiez le fichier `.env` de votre projet Symfony pour utiliser PostgreSQL comme base de données :

```env
POSTGRES_USER=bads_club_user
POSTGRES_PASSWORD=bads_club_password
POSTGRES_DB=bads_club

DATABASE_URL="postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}?serverVersion=16&charset=utf8"
```

## 2. Configuration pour le développement continu
### 2.1 Créer un Dockerfile de développement `Dockerfile`
Le Dockerfile pour le développement continu installe toutes les dépendances nécessaires et configure l'environnement.
```Dockerfile
# Dockerfile

# Utiliser une image PHP avec FPM
FROM php:8.2-fpm

# Installer les extensions PHP nécessaires et autres dépendances
RUN apt-get update && apt-get install -y \
    libpq-dev \
    git \
    unzip \
    && docker-php-ext-install pdo pdo_pgsql

# Installer Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Installer Symfony CLI
RUN curl -sS https://get.symfony.com/cli/installer | bash
RUN mv /root/.symfony*/bin/symfony /usr/local/bin/symfony

# Copier les fichiers de l'application dans le conteneur
COPY . /var/www/html

# Définir le répertoire de travail
WORKDIR /var/www/html

# Exposer le port 8000 pour Symfony server
EXPOSE 8000

# Commande pour lancer Symfony server
CMD ["sh", "-c", "symfony serve --port=8000 || symfony serve --port=8000"]
```

### 2.2 Créer un fichier Docker Compose pour le développement continu `docker-compose.yml`
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: symfony_api
    ports:
      - "8000:8000" # Redirige le port 8000 du conteneur vers le port 8000 de l'hôte
    volumes:
      - .:/var/www/html # Monte le répertoire local dans /var/www/html du conteneur
    networks:
      - symfony  # Utilise le réseau défini pour les communications inter-conteneurs
    depends_on:
      - db  # Indique que le conteneur `app` dépend du conteneur `db`
    environment:
      - DATABASE_URL=${DATABASE_URL}

  db:
    image: postgres:latest
    container_name: symfony_db
    environment:
      POSTGRES_USER: ${POSTGRES_USER}  # Définit l'utilisateur PostgreSQL (config dans le .env)
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  # Définit le mot de passe PostgreSQL (config dans le .env)
      POSTGRES_DB: ${POSTGRES_DB}  # Définit la base de données PostgreSQL (config dans le .env)
    ports:
      - "6442:5432"  # Redirige le port 5432 du conteneur vers le port 6442 de l'hôte (5432 ne marche pas sur mon poste)
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - symfony  # Utilise le réseau défini pour les communications inter-conteneurs

volumes:
  postgres_data:

networks:
  symfony:
    driver: bridge
```

### 2.3 Lancer Docker Compose pour le développement continu
Lancez les conteneurs définis dans docker-compose.yml.
```bash
docker-compose build
```

Ensuite
```bash
docker-compose up -d
```

*Explication :**

- **-d** : pour "detached", ce qui permet de ne pas afficher les logs dans le terminal.

- **-d** : pour "detached", ce qui permet de ne pas afficher les logs dans le terminal.

## 3. Setup avancé de Symfony
Installez les packages nécessaires pour étendre les fonctionnalités de Symfony : (1 ligne à la fois)

```bash
docker-compose exec app composer require symfony/orm-pack

docker-compose exec app composer require symfony/uid

docker-compose exec app composer require symfony/security-bundle

docker-compose exec app composer require symfony/maker-bundle --dev

docker-compose exec app composer require lexik/jwt-authentication-bundle

docker-compose exec app composer require api
```
### 3. Créer le serveur dans postgree

Server, clic droit, `Register`
Utiliser les informations de `.env`
```bash
- En connection
--> Hostname/address: localhost
--> Port: 6442
--> Maintenance database: bads_club
--> username: bads_club_user
--> Password : xxx

- General
--> name: docker
```

### 3.1 Créer l'entité User et la table User dans la base de données
```bash
docker-compose exec app php bin/console make:user --with-uuid
```

ajouter dans l'entité
```bash
#[ApiResource] et l'importer
```


### 3.2 Générer une migration

```bash
docker-compose exec app php bin/console make:migration --formatted
```

### 3.3 Appliquer la migration
```bash
docker-compose exec app php bin/console doctrine:migrations:migrate
```

Vérifiez dans la base de données que la table `User` a bien été créée.

### 3.4 Générer les clés JWT pour l'authentification
```bash
docker-compose exec app php bin/console lexik:jwt:generate-keypair
```

### 3.5 Créer un contrôleur UserController
```bash
docker-compose exec app php bin/console make:controller UserController
```

### Optionnel
#### Ajouter une autre entité `Sport`
Ajouter le '#[ApiResource]' dans l'entité
```bash
docker-compose exec app php bin/console make:entity Sport

docker-compose exec app php bin/console make:migration --formatted

docker-compose exec app php bin/console doctrine:migrations:migrate
```
#### Ajouter le controller `SportController`
docker-compose exec app php bin/console make:controller SportController

### 4. Postgre
#### 4.1 Se connecter à la base de données dans la terminal en vs
```bash
docker-compose exec db psql -U bads_club_user -d bads_club
```
#### 4.2 Listez les tables pour vérifier la présence de la nouvelle table
```bash
\dt
```
# 5. Configuration Hoppscotch
## 5.1 Editer le fichier `security.yaml`
```bash
security:
    # https://symfony.com/doc/current/security.html#registering-the-user-hashing-passwords
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
    # https://symfony.com/doc/current/security.html#loading-the-user-the-user-provider
    providers:
        # used to reload user from session & other features (e.g. switch_user)
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
        # used to reload user from session & other features (e.g. switch_user)
    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false
        login:
            pattern: ^/api/login
            stateless: true
            json_login:
                username_path: email
                password_path: password
                check_path: /api/login_check
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure

        api:
            pattern:   ^/api/
            stateless: true
            jwt: ~

    access_control:
        - { path: ^/api/login, roles: PUBLIC_ACCESS }
        - { path: ^/api,       roles: IS_AUTHENTICATED_FULLY }
        # main:
        #   lazy: true
        #   provider: app_user_provider

            # activate different ways to authenticate
            # https://symfony.com/doc/current/security.html#the-firewall

            # https://symfony.com/doc/current/security/impersonating_user.html
            # switch_user: true

    # Easy way to control access for large sections of your site
    # Note: Only the *first* access control that matches will be used
    # access_control:
        # - { path: ^/admin, roles: ROLE_ADMIN }
        # - { path: ^/profile, roles: ROLE_USER }

when@test:
    security:
        password_hashers:
            # By default, password hashers are resource intensive and take time. This is
            # important to generate secure password hashes. In tests however, secure hashes
            # are not important, waste resources and increase test times. The following
            # reduces the work factor to the lowest possible values.
            Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface:
                algorithm: auto
                cost: 4 # Lowest possible value for bcrypt
                time_cost: 3 # Lowest possible value for argon
                memory_cost: 10 # Lowest possible value for argon
```
## 5.2 Editer fichier `lexik_jwt_authentication.yaml`
```bash
        login:
            pattern: ^/api/login
            stateless: true
            json_login:
                username_path: email
                password_path: password
                check_path: /api/login_check
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure

        api:
            pattern:   ^/api/
            stateless: true
            jwt: ~

    access_control:
        - { path: ^/api/login, roles: PUBLIC_ACCESS }
        - { path: ^/api,       roles: IS_AUTHENTICATED_FULLY }
```

## 5.3 Ajouter dans le fichier `routes.yaml`
```bash
    api_login_check:
        path: /api/login_check
```
