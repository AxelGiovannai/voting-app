# Documentation Technique 3DOKR

## Informations importantes

Le fichier `.env` a volontairement été fourni pour faciliter la configuration.


## 1. Présentation du Projet

Ce projet consiste en la modernisation d'une application distribuée existante initialement lancée par scripts. L'objectif a été de conteneuriser l'ensemble des modules et de déployer l'architecture sur un cluster **Docker Swarm** et/ou via un **Docker Compose**.

L'application se compose de 5 services :

- **Vote** : Interface utilisateur pour voter.
- **Result** : Interface utilisateur affichant les résultats en temps réel.
- **Worker** : Service de fond traitant les votes.
- **Redis** : File d'attente en mémoire pour les votes entrants.
- **PostgreSQL** : Base de données persistante pour le stockage définitif.


## 2. Architecture Technique

### 2.1 Conteneurisation

Chaque service applicatif dispose de son propre `Dockerfile`. Nous avons privilégié les images légères et respecté les bonnes pratiques (non-root users quand possible, nettoyage des caches).

### 2.2 Orchestration (Docker Swarm)

L'application utilise deux réseaux overlay distincts pour la sécurité :

1. **`frontend`** : Réseau public exposant les interfaces Web (`vote`, `result`).
2. **`backend`** : Réseau privé interne. Seuls les services backend et la base de données y communiquent. La base de données n'est jamais exposée directement.


## 3. Infrastructure du Cluster (Mise en place)

Le cluster a été déployé sur un environnement virtualisé **VMware Workstation Pro 25**.

### 3.1 Configuration des 3 Machines Virtuelles

- **OS :** Debian 13.
- **Réseau :** Mode NAT (Les VMs partagent une plage IP privée, ex: `192.168.100.x`).
- **Installation :** Installation classique de Debian avec les paramètres par défaut.
- **Post-Installation :** Installation de Docker, ajout de l'utilisateur au groupe docker (`usermod -aG docker $USER`) puis clonage de la machine virtuelle pour en faire trois. Pensez à bien sélectionner "Créer un clone complet" lors du clonage des machines virtuelles sur VMWare.
### 3.2 Préparation des VMs (Procédure de clonage)

Afin d'éviter les conflits d'identité Docker lors du clonage des VMs sous VMware, la procédure suivante a été appliquée sur chaque nœud :

1. Changement du `hostname` (manager, worker1, worker2) via `sudo hostnamectl set-hostname <nouveau-hostname>`
2. **Nettoyage de l'ID Docker** (Crucial pour le Swarm) :
```bash
sudo rm /etc/docker/key.json
sudo systemctl restart docker
```

### 3.3 Initialisation du Swarm

**Sur le nœud Manager :**

```bash
docker swarm init --advertise-addr <IP_MANAGER>
```
*(Cette commande va renvoyer une commande à copier impérativement (voir ci-dessous)).*

**Sur les nœuds Workers :**
Utilisation du token fourni par le manager :

```bash
docker swarm join --token <TOKEN> <IP_MANAGER>:2377
```

**Validation :**

```bash
docker node ls
```

*(Cette commande confirme que les 3 nœuds sont en statut `Ready` et `Active`).*


## 4. Déploiement de l'Application

Le déploiement s'effectue via le fichier `stack.yaml`, mais l'application au complet (.zip rendu) est nécessaire sur la machine Manager pour fonctionner.

### 4.1 Gestion des Secrets (Sécurité)

Pour ne pas exposer le mot de passe de la base de données en clair dans le code, nous utilisons **Docker Secrets**.
Avant le déploiement, le secret est créé manuellement sur le Manager :

```bash
echo "postgres" | docker secret create db_password -
```

### 4.2 Configuration des Variables d'Environnement

Un fichier `.env` est créé sur le Manager pour définir les paramètres non sensibles :

```bash
# Fichier .env
VOTE_PORT=5000
RESULT_PORT=5001
# Doit correspondre au contenu du secret pour que les apps s'authentifient
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=postgres
```

### 4.3 Lancement de la Stack

```bash
# Export des variables
export $(cat .env | xargs)

# Déploiement
docker stack deploy -c stack.yaml vote-stack
```

## 5. Vérification du Fonctionnement

### État du Cluster

Commande : `docker service ls`

### Accès aux Interfaces

L'application est accessible via le Routing Mesh de Swarm sur l'IP de n'importe quel nœud :

- **Vote :** `http://<IP_VM>:5000`
- **Result :** `http://<IP_VM>:5001`
- **Visualizer :** `http://<IP_VM>:8080`

Voici la section à ajouter à ta documentation. Le meilleur endroit pour l'insérer est **juste après la partie "2. Architecture Technique"** et **avant "3. Infrastructure du Cluster"**.

Cela respecte la logique du projet : d'abord on développe/test en local (Compose), ensuite on prépare l'infra (VMs), et enfin on déploie en prod (Swarm).

## 6. Développement et Build (Docker Compose)

Avant le déploiement sur le cluster, l'application est validée dans un environnement local à l'aide de **Docker Compose**. Cette étape permet également de **construire les images Docker** à partir des sources (dossiers `vote/`, `worker/`, `result/`).

### 3.1 Configuration

Un fichier `.env` identique à celui de production est requis à la racine du projet pour définir les ports et identifiants.

### 3.2 Lancement et Build

La commande suivante lancée à la racine du projet permet de compiler les images et de lancer l'application en arrière-plan :

```bash
docker compose up --build -d

```

* L'option `--build` force la reconstruction des images à partir du code source local.
* L'option `-d` lance les conteneurs en arrière-plan.

L'application est alors accessible localement sur `http://localhost:5000` (Vote) et `http://localhost:5001` (Result).

### 3.3 Arrêt de l'environnement

Pour arrêter l'application et supprimer les conteneurs ainsi que les réseaux temporaires :

```bash
docker compose down

```

*(Note : Les volumes de données sont conservés sauf si l'option `-v` est ajoutée).*