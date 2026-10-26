# Infrastructure - Poweradmin

![Bannière Loutik](https://raw.githubusercontent.com/loutik/design-assets/main/banniere_loutik.png)

Déploiement conteneurisé de [Poweradmin](https://www.poweradmin.org/) pour administrer le DNS de Loutik. L'application est accessible en HTTPS derrière Nginx, avec un certificat de la PKI interne et une base de données dédiée.

> [!IMPORTANT]
> Cette procédure concerne un serveur Linux déjà intégré à l'infrastructure Loutik. Les valeurs d'exemple (`padm.infra.loutik.fr`, `poweradmin`, noms DNS et identifiants) doivent être adaptées à l'environnement cible.

## Prérequis

- un serveur Linux joignable par SSH, avec Docker Engine et le plugin Compose ;
- les droits `root` ou `sudo` ;
- un enregistrement DNS pour `padm.infra.loutik.fr` pointant vers le serveur ;
- l'accès au registre `ghcr.io` ;
- les flux entrants TCP `80` et `443` autorisés jusqu'au serveur ;
- un accès au serveur DNS que Poweradmin doit administrer ;
- les informations de connexion à la base de données.

## Informations à demander avant l'installation

Les deux demandes suivantes doivent être réalisées et validées **avant** de copier le dépôt ou de démarrer les conteneurs.

> [!WARNING]
> **Certificat TLS — demande à la PKI interne**
>
> Demander au service PKI un certificat serveur pour `padm.infra.loutik.fr`, avec sa clé privée et la chaîne de certification nécessaire à la validation par les postes et services Loutik.
>
> Vérifier que le certificat couvre le nom DNS réellement utilisé par l'interface et par l'API. Ne pas utiliser les fichiers présents dans `certs/` : ils sont uniquement fournis comme exemples. Demander également la procédure de renouvellement et les droits de diffusion de la clé privée.

> [!WARNING]
> **Base de données — demande au service DBA / Ansible**
>
> Demander au service responsable des bases de données d'exécuter le playbook disponible dans le dépôt Ansible de Loutik. Il doit créer une base dédiée à Poweradmin, un utilisateur dédié et les privilèges strictement nécessaires.
>
> À obtenir avant l'installation : type de SGBD (`mysql` ou `pgsql`), nom d'hôte, port si différent du port standard, nom de base, identifiant, mot de passe transmis par un canal sécurisé et politique de sauvegarde. Ne pas créer la base manuellement sur le serveur Poweradmin et ne pas réutiliser un compte partagé.

## Structure du dépôt

```text
.
├── certs/
│   ├── padm.infra.loutik.fr.crt   # Exemple uniquement, à remplacer
│   └── padm.infra.loutik.fr.key   # Exemple uniquement, à remplacer
├── nginx/
│   └── padm.conf                  # Redirection HTTP et reverse proxy HTTPS
├── docker-compose.yml             # Services Poweradmin et Nginx
├── LICENSE.md
└── readme.md
```

- **`docker-compose.yml`** : lance Poweradmin `4.5.0` et Nginx sur le réseau Docker privé `padm_net`.
- **`nginx/padm.conf`** : publie le nom DNS en HTTPS et transmet les requêtes à Poweradmin.
- **`certs/`** : emplacement des certificats monté en lecture seule dans Nginx. Les certificats d'exemple ne doivent pas être utilisés en production.
- **`.env`** : fichier local non versionné contenant les paramètres de connexion et de configuration de Poweradmin.

## Installation

### 1. Installer les prérequis sur le serveur

Installer Docker Engine et Compose selon la distribution du serveur, puis vérifier leur disponibilité :

```bash
docker --version
docker compose version
```

Le compte utilisé pour l'installation doit pouvoir exécuter Docker avec `sudo`.

### 2. Déployer le dépôt dans `/opt`

Depuis le serveur, cloner le dépôt dans le chemin utilisé par les volumes du fichier Compose :

```bash
sudo mkdir -p /opt
sudo git clone <URL_DU_DEPOT> /opt/poweradmin
cd /opt/poweradmin
```

Si le dépôt est copié par un autre moyen, conserver exactement le chemin `/opt/poweradmin`. Le fichier Compose référence notamment `/opt/poweradmin/nginx/padm.conf` et `/opt/poweradmin/certs`.

### 3. Installer le certificat de la PKI interne

Après validation de la demande PKI, copier les fichiers reçus dans `certs/`. Les noms ci-dessous correspondent à la configuration Nginx :

```bash
sudo install -o root -g root -m 0644 \
  /chemin/de-reception/padm.infra.loutik.fr.crt \
  /opt/poweradmin/certs/padm.infra.loutik.fr.crt

sudo install -o root -g root -m 0600 \
  /chemin/de-reception/padm.infra.loutik.fr.key \
  /opt/poweradmin/certs/padm.infra.loutik.fr.key
```

Si la PKI fournit une chaîne intermédiaire, utiliser le fichier recommandé par le service PKI pour `ssl_certificate` (généralement un fichier `fullchain`). Ne jamais committer une clé privée ni un certificat de production.

Contrôler le certificat et sa date d'expiration :

```bash
sudo openssl x509 -in /opt/poweradmin/certs/padm.infra.loutik.fr.crt \
  -noout -subject -issuer -dates -ext subjectAltName
```

### 4. Créer et sécuriser `.env`

Créer le fichier uniquement sur le serveur. Les valeurs ci-dessous sont un modèle : remplacer tous les éléments entre chevrons par les informations fournies par le service DBA.

```bash
cd /opt/poweradmin
sudo install -o root -g root -m 0600 /dev/null .env
sudoedit .env
```

Exemple pour PostgreSQL :

```dotenv
DB_TYPE=pgsql
DB_HOST=<hote-fourni-par-le-DBA>
DB_PORT=5432
DB_NAME=<base-dediee>
DB_USER=<utilisateur-dedie>
DB_PASS=<mot-de-passe-fourni-par-le-DBA>

PA_CREATE_ADMIN=1
PA_ADMIN_PASSWORD=<mot-de-passe-administrateur-temporaire>
DNS_NS1=<serveur-DNS-primaire>
DNS_NS2=<serveur-DNS-secondaire>
DNS_HOSTMASTER=<adresse-email-DNS>
```

Pour MySQL/MariaDB, remplacer `DB_TYPE=pgsql` par `DB_TYPE=mysql` et utiliser le port communiqué par le DBA. `DB_TYPE`, `DB_HOST`, `DB_USER`, `DB_PASS` et `DB_NAME` sont obligatoires pour une base distante.

Vérifier que le fichier n'est lisible que par `root` et ne contient pas de guillemets ou d'espaces superflus :

```bash
sudo chown root:root /opt/poweradmin/.env
sudo chmod 0600 /opt/poweradmin/.env
sudo stat -c '%U:%G %a %n' /opt/poweradmin/.env
```

Ne pas afficher `.env` dans les tickets, journaux, captures d'écran ou commandes partagées. Ajouter `.env` à `.gitignore` si le dépôt est amené à être modifié localement.

### 5. Vérifier les droits de l'ensemble du déploiement

Les fichiers de configuration et la clé privée doivent appartenir à `root`. Les répertoires peuvent rester traversables afin que Docker puisse monter les volumes :

```bash
sudo chown -R root:root /opt/poweradmin
sudo chmod 0755 /opt/poweradmin /opt/poweradmin/certs /opt/poweradmin/nginx
sudo chmod 0644 /opt/poweradmin/docker-compose.yml /opt/poweradmin/nginx/padm.conf
sudo chmod 0600 /opt/poweradmin/.env /opt/poweradmin/certs/*.key
```

### 6. Démarrer les services

Depuis `/opt/poweradmin`, télécharger l'image et démarrer les conteneurs :

```bash
cd /opt/poweradmin
sudo docker compose up -d
sudo docker compose ps
```

`PA_CREATE_ADMIN=1` et `PA_ADMIN_PASSWORD` permettent la création initiale du compte administrateur `admin` selon le comportement de l'image. Utiliser un mot de passe temporaire transmis par un canal sécurisé, puis le changer lors de la première connexion. Une fois l'installation initiale terminée, supprimer ces variables ou les désactiver dans `.env`, puis recréer le conteneur :

```bash
sudo sed -i -e '/^PA_CREATE_ADMIN=/d' -e '/^PA_ADMIN_PASSWORD=/d' /opt/poweradmin/.env
sudo docker compose up -d --force-recreate
```

### 7. Valider l'installation

Contrôler les journaux sans afficher `.env` :

```bash
sudo docker compose logs --tail=100 poweradmin
sudo docker compose logs --tail=100 nginx
```

Tester la redirection HTTP, le certificat et l'interface HTTPS depuis un poste disposant de la PKI interne :

```bash
curl -I http://padm.infra.loutik.fr
curl -fsS -I https://padm.infra.loutik.fr
```

Le premier appel doit retourner une redirection vers HTTPS et le second un code HTTP de succès ou une réponse d'authentification. Se connecter ensuite à l'interface avec le compte administrateur initial, changer son mot de passe et vérifier une opération DNS non destructive. Tester également l'URL d'API prévue par l'instance avec le même nom DNS et le même certificat.

## Exploitation courante

```bash
cd /opt/poweradmin

# État des conteneurs
sudo docker compose ps

# Suivi des journaux
sudo docker compose logs -f --tail=100

# Mise à jour vers une version explicitement validée
sudo docker compose pull
sudo docker compose up -d

# Arrêt et redémarrage
sudo docker compose down
sudo docker compose up -d
```

Ne pas utiliser `docker compose down -v` : cela supprimerait les volumes éventuellement ajoutés ultérieurement au déploiement.

## Sécurité et sauvegardes

1. Restreindre l'accès à l'interface et à l'API aux réseaux d'administration autorisés par le pare-feu.
2. Conserver `.env` et la clé privée avec le propriétaire `root` et le mode `0600`.
3. Renouveler le certificat avant son expiration et redémarrer Nginx après remplacement des fichiers.
4. Sauvegarder la base via la procédure DBA ; tester régulièrement la restauration.
5. Sauvegarder séparément la configuration nécessaire à la reconstruction, sans inclure les secrets dans Git.
6. Maintenir l'image Poweradmin sur une version explicitement validée et consulter les notes de version avant chaque mise à jour.
7. Superviser l'expiration du certificat, l'état des conteneurs, l'espace disque et les erreurs de connexion à la base.

## Dépannage

- **Nginx ne démarre pas** : vérifier les chemins, les permissions et la validité du certificat avec `docker compose logs nginx`.
- **Erreur de connexion à la base** : vérifier `DB_TYPE`, l'hôte, le port, les droits réseau et les identifiants fournis par le DBA.
- **Le nom DNS ne répond pas** : vérifier l'enregistrement DNS et les règles de pare-feu TCP `80/443`.
- **Certificat non reconnu** : vérifier que la chaîne PKI interne est installée sur le poste client et que le SAN contient `padm.infra.loutik.fr`.

## Mainteneur

- **Louis MEDO** | [LinkedIn](https://www.linkedin.com/in/louismedo/) | [Portfolio](https://louis.loutik.fr/) | [GitHub](https://github.com/FireToak) | [louis.medo@loutik.fr](mailto:louis.medo@loutik.fr)

<div align="center">
<br>
<small><i>Dernière mise à jour : 26 octobre 2026</i></small>
</div>
