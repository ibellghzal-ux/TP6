# Compte rendu · TP6 : conteneuriser MaintAI (Docker, Docker Compose)

**Groupe :** À compléter  
**Membres :** À compléter  
**Date :** 26/11/2026

---

## 1. Mise en place (dans `TP/TP6`)

Sortie de `docker run hello-world` et versions installées :

```text
Hello from Docker!
Docker version 28.5.1, build e180ab8
Docker Compose version v2.40.3-desktop.1
```

Les lignes détaillées Client et Server de `docker version` ne sont pas disponibles dans les résultats conservés.

Dernière ligne de `python -m pytest` :

```text
38 passed
```

## 2. Image et conteneur (pendant le premier build)

**1. Quelle différence entre une image et un conteneur ?**

Une image Docker est un modèle qui contient l'application, ses dépendances et sa configuration. Un conteneur est une instance de cette image en cours d'exécution.

**2. Pourquoi `COPY requirements-docker.txt` est-il placé avant `COPY maintai` ?**

Cela permet à Docker de mettre en cache l'installation des dépendances. Si le code source change, Docker n'a pas besoin de réinstaller toutes les dépendances, ce qui accélère la construction de l'image.

**3. Pourquoi `requirements-docker.txt` contient-il `scikit-learn==1.8.0` et pas `scikit-learn>=1.5` ?**

La version est fixée pour garantir un environnement stable et reproductible. Cela évite les différences de comportement et les incompatibilités liées à l'installation d'une version différente.

**4. Pourquoi ni `mlflow`, ni `dvc`, ni `pytest` dans l'image ?**

Ces outils servent principalement au développement, au suivi des expériences, à la gestion des données et aux tests. Ils ne sont pas nécessaires pour exécuter l'API. Leur absence permet de réduire la taille de l'image.

## 3. Sprint 1 : le Dockerfile

Notre `Dockerfile` :

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements-docker.txt .
RUN pip install --no-cache-dir -r requirements-docker.txt

COPY maintai ./maintai
COPY ui ./ui

ENV MAINTAI_MODELS_DIR=/app/models
ENV MAINTAI_DATA_PATH=/app/data/sensors.csv

EXPOSE 8000

CMD ["uvicorn", "maintai.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

| Build | Durée | Taille de l'image |
|---|---|---|
| Premier build | Non relevée | 1,13 Go |
| Reconstruction avec Docker Compose | 3,3 secondes | 1,13 Go |

Lors de la reconstruction, les étapes suivantes ont été réutilisées depuis le cache :

```text
CACHED [2/6] WORKDIR /app
CACHED [3/6] COPY requirements-docker.txt .
CACHED [4/6] RUN pip install --no-cache-dir -r requirements-docker.txt
CACHED [5/6] COPY maintai ./maintai
CACHED [6/6] COPY ui ./ui
```

**Que signifie `CACHED` dans le journal du second build ?**

`CACHED` signifie que Docker réutilise une couche déjà construite et enregistrée dans son cache. Il n'a donc pas besoin de refaire cette étape, ce qui permet de gagner du temps pendant la construction de l'image.

Remarque : la reconstruction a été effectuée sans modification préalable de `maintai/predict.py`. Le comportement précis du cache après une modification de ce fichier n'a donc pas été mesuré.

## 4. Sprint 1 : lancer un conteneur

Commande réellement utilisée sous PowerShell :

```powershell
docker run -d --name maintai-api -p 8000:8000 --env-file .env -v "${PWD}/models:/app/models:ro" maintai:0.4.0
```

Le conteneur a démarré correctement.

**Résultats des tests :**

| Essai | Résultat |
|---|---|
| `GET /health` | `status: ok`, `model_loaded: True` |
| `POST /v1/predict` sans clé | `Clé API absente ou invalide` |
| `POST /v1/predict` avec la clé | Prédiction réussie : probabilité 1.0, alerte True, seuil 0.3 |

Résultat de la prédiction avec la clé API :

```text
failure_probability : 1.0
alert               : True
threshold           : 0.3
model_version       : 2026-11-26T09:36:16Z
```

**Piège 1. Sans l'option `-v ...models...`, que répondent `/health` et `/v1/predict` ? Pourquoi ?**

Sans le volume contenant le modèle, le conteneur démarre, mais le modèle n'est pas chargé.

Résultat de `/health` :

```text
status       : ok
model_loaded : False
```

Résultat de `/v1/predict` :

```json
{"detail":"Modèle non chargé : lancez python -m maintai.train puis redémarrez l'API"}
```

Le volume `models` est donc nécessaire pour que le conteneur puisse accéder au modèle entraîné.

**Piège 2. Avec `--host 127.0.0.1`, que disent `docker logs test-localhost` et `http://localhost:8002/health` ? Pourquoi ?**

Les logs du conteneur indiquent :

```text
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000
```

La requête vers `http://localhost:8002/health` depuis Windows échoue avec une erreur de connexion interrompue.

Avec `--host 127.0.0.1`, le serveur écoute uniquement sur l'interface locale du conteneur. Le port publié par Docker ne permet donc pas d'accéder correctement à l'application depuis l'hôte.

Il faut utiliser `--host 0.0.0.0` pour rendre le serveur accessible depuis l'extérieur du conteneur.

## 5. Sprint 2 : Docker Compose

Notre `compose.yaml` :

```yaml
services:
  api:
    build: .
    image: maintai:0.4.0
    env_file: .env
    ports:
      - "8000:8000"
    volumes:
      - ./models:/app/models:ro

  ui:
    image: maintai:0.4.0
    command:
      - python
      - -m
      - streamlit
      - run
      - ui/app.py
      - --server.address=0.0.0.0
      - --server.port=8501
      - --server.headless=true
    environment:
      MAINTAI_API_URL: http://api:8000
    ports:
      - "8501:8501"
    volumes:
      - ./data:/app/data:ro
    depends_on:
      - api
```

Sortie de `docker compose ps` :

```text
NAME        IMAGE           COMMAND                  SERVICE   CREATED         STATUS         PORTS
tp6-api-1   maintai:0.4.0   "uvicorn maintai.api…"   api       2 minutes ago   Up 2 minutes   0.0.0.0:8000->8000/tcp
tp6-ui-1    maintai:0.4.0   "python -m streamlit…"   ui        2 minutes ago   Up 2 minutes   0.0.0.0:8501->8501/tcp
```

**Quelle URL affiche la barre latérale de l'interface ? D'où vient-elle ?**

L'URL est `http://api:8000`. Elle provient de la variable d'environnement `MAINTAI_API_URL`, définie dans le fichier `compose.yaml` pour le service `ui`.

Le nom `api` correspond au nom du service API dans le réseau Docker Compose.

**Résultat des deux commandes `docker compose exec ui ...` (avec `api` puis avec `localhost`). Pourquoi la seconde échoue-t-elle ?**

Avec `api` :

Commande :

```powershell
docker compose exec ui python -c "import urllib.request; print(urllib.request.urlopen('http://api:8000/health').read().decode())"
```

Résultat :

```json
{"status":"ok","model_loaded":true}
```

La communication fonctionne car Docker Compose permet aux services de communiquer entre eux en utilisant leur nom de service comme nom d'hôte.

Avec `localhost` :

Commande :

```powershell
docker compose exec ui python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8000/health').read().decode())"
```

Résultat :

```text
urllib.error.URLError: <urlopen error [Errno 111] Connection refused>
```

La seconde commande échoue car, depuis le conteneur UI, `localhost` désigne le conteneur UI lui-même et non le conteneur API.

L'interface Streamlit a également été vérifiée dans le navigateur à l'adresse `http://localhost:8501`. Le tableau de bord s'affiche correctement et une prédiction a été réalisée avec succès depuis le formulaire.

Dernière ligne de `python -m pytest` conservée après la mise en place :

```text
38 passed
```

## 6. Décision : l'ADR 005

Nous avons choisi de construire une image Docker basée sur Python 3.11-slim afin de contenir l'application MaintAI et ses dépendances nécessaires à son exécution. Docker Compose permet de lancer séparément l'API et l'interface Streamlit, de configurer leur communication et de partager les modèles et les données grâce aux volumes.

Le fichier `docs/adr/005-image-docker.md` doit être créé avec cette décision.

## 7. Questions de compréhension

**1. Qu'est-ce qui entre dans l'image, et qu'est-ce qui arrive au lancement ? Donnez le moyen pour chaque élément.**

L'image contient Python, les dépendances installées depuis `requirements-docker.txt`, le code de l'application MaintAI et le code de l'interface Streamlit. Ces éléments sont ajoutés avec les instructions `FROM`, `RUN` et `COPY` du Dockerfile.

Au lancement, les variables d'environnement, les ports et les volumes sont configurés. Le modèle et les données sont fournis par les volumes Docker Compose. La clé API est fournie par le fichier `.env` grâce à la directive `env_file`.

**2. Pourquoi la clé API n'est-elle jamais dans l'image ?**

La clé API est une information confidentielle. Elle ne doit pas être intégrée dans l'image afin d'éviter qu'elle soit exposée aux personnes qui récupèrent ou partagent cette image. Elle est fournie au lancement grâce au fichier `.env`.

**3. Livrer un modèle (TP4, TP5) : qu'est-ce que Docker ajoute à la liste ?**

Docker permet de livrer l'application avec son environnement d'exécution et ses dépendances dans une image reproductible. Il facilite également le déploiement, l'isolation des services et leur communication avec Docker Compose.

## 8. Le rendu Git

- Lien de la pull request : à créer.
- Le tag `tp6` existe sur GitHub : non, pas encore.
- `git status` avant le premier commit : les fichiers `.env`, `models/` et `data/*.csv` étaient ignorés grâce au `.gitignore`.

Le dépôt Git local a été initialisé et le premier commit a été créé.

Le dépôt distant GitHub est configuré à l'adresse :

https://github.com/ibellghzal-ux/TP6

Il reste à envoyer les dernières modifications sur GitHub, créer une pull request et ajouter le tag `tp6`.

## 9. Ce qui reste flou

Les tests principaux de Docker et Docker Compose ont été réalisés avec succès.

Les points qui restent à finaliser sont :
- La création du fichier ADR 005 dans `docs/adr/`.
- La vérification finale des cinq fichiers à rendre.
- L'envoi des modifications sur GitHub.
- La création de la pull request.
- La création et l'envoi du tag `tp6`.

La durée exacte du premier build et le test de reconstruction après modification de `maintai/predict.py` n'ont pas été relevés.
