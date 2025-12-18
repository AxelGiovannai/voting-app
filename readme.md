# Voting App

Ce projet est une application distribuée de vote conteneurisée et orchestrée via Docker Swarm.

## 1. Architecture

L'application repose sur une architecture microservices composée de 5 conteneurs interconnectés.

### Les Composants
* **Vote** (Python/Flask) : Interface Front-end exposée au public pour voter (Option A vs Option B).
* **Redis** (Queue) : Base de données clé-valeur en mémoire, servant de tampon pour les votes entrants.
* **Worker** (.NET Core) : Service Back-end qui consomme les votes depuis Redis et les persiste dans PostgreSQL.
* **DB** (PostgreSQL) : Base de données relationnelle pour le stockage durable des votes.
* **Result** (Node.js) : Interface Front-end affichant les résultats en temps réel via WebSockets.

### Réseau et Sécurité
L'architecture implémente une segmentation réseau stricte :
* **Réseau `frontend`** : Public. Seuls les services `vote` et `result` y sont connectés.
* **Réseau `backend`** : Privé. Contient `redis`, `db` et `worker`. Inaccessible depuis l'extérieur.

Les données sensibles (mots de passe BDD) sont gérées exclusivement via **Docker Secrets** (chiffrés au repos et en transit), et jamais via des variables d'environnement en clair.

---

## 2. Dockerfile

Chaque service dispose d'un `Dockerfile`.
### Optimisations mises en place :
* **Utilisateurs non-root** : Création et utilisation d'utilisateurs dédiés (`appuser`, `node`) pour éviter que les conteneurs tournent en `root`.
* **Images légères** : Utilisation de versions `slim` (Debian) ou `alpine` pour réduire la surface d'attaque.
* **Multi-stage builds** : Pour le service `.NET`, la compilation se fait dans une image lourde (SDK), mais l'exécution se fait dans une image runtime ultra-légère.

### Exemple : Le Worker (.NET)
```dockerfile
# Étape 1 : Build
FROM [mcr.microsoft.com/dotnet/sdk:7.0](https://mcr.microsoft.com/dotnet/sdk:7.0) AS build
WORKDIR /source
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

# Étape 2 : Runtime
FROM [mcr.microsoft.com/dotnet/runtime:7.0](https://mcr.microsoft.com/dotnet/runtime:7.0)
WORKDIR /app
RUN useradd -m appuser          # Création user sécurisé
COPY --from=build /app/publish .
RUN chown -R appuser /app
USER appuser                    # Exécution non-root
CMD ["dotnet", "Worker.dll"]
3. Compose
Pour le développement local et la construction des images, nous utilisons docker-compose. Ce fichier gère la configuration dynamique via un fichier .env.
```

Configuration .env
Un fichier .env à la racine permet de piloter l'infrastructure sans toucher au code :


Extrait de code

```
VOTE_PORT=5000
RESULT_PORT=5001
POSTGRES_USER=postgres
POSTGRES_DB=postgres
```

Fonctionnalités Clés
- Healthchecks : Le service db possède un test de santé (pg_isready). Les services dépendants (worker, result) attendent que la base soit healthy avant de démarrer.

- Volumes : Persistance des données PostgreSQL dans un volume Docker géré.

Commande pour construire les images :

```Bash
docker compose build
```

---
## 4. Swarm
Le déploiement en production est géré par un cluster Docker Swarm. C'est ici que l'orchestration, la haute disponibilité et la gestion des secrets prennent tout leur sens.

Initialisation
Si ce n'est pas fait :

```Bash
docker swarm init --advertise-addr <VOTRE_IP>
```

Gestion des Secrets
Création du mot de passe de la base de données (ne jamais le mettre dans le code) :

```Bash
printf "votre_mot_de_passe_secret" | docker secret create db_password -
```

### Déploiement de la Stack (stack.yml)
Le fichier stack.yml définit les règles de production :

Replicas : 2 instances pour l'app de vote (répartition de charge).

Update Config : Mise à jour sans interruption de service (Rolling Update).

Placement Constraints : La DB est épinglée au nœud Manager pour garantir l'accès à son volume disque.

Secrets : Injection sécurisée du fichier /run/secrets/db_password dans les conteneurs.

Commande de déploiement :

```Bash
export $(grep -v '^#' .env | xargs) && docker stack deploy -c stack.yml voting-stack
```


Vote : http://localhost:5000

Résultats : http://localhost:5001

Visualizer : http://localhost:8080 (Vue graphique du cluster)

```Bash
docker stack services voting-stack
```

