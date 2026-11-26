# ADR 005 : Conteneurisation de MaintAI avec Docker

- **Statut :** Accepté
- **Date :** 26/11/2026

## Contexte

Le projet MaintAI doit pouvoir être exécuté dans un environnement reproductible, avec ses dépendances et ses services séparés.

## Décision

Nous avons choisi de construire une image Docker basée sur Python 3.11-slim afin de contenir l'application MaintAI et ses dépendances nécessaires à son exécution. Docker Compose permet de lancer séparément l'API et l'interface Streamlit, de configurer leur communication et de partager les modèles et les données grâce aux volumes.

## Conséquences

- L'environnement d'exécution est reproductible.
- L'API et l'interface Streamlit fonctionnent dans des conteneurs séparés.
- Les modèles et les données sont accessibles grâce aux volumes.
- La clé API est fournie au lancement avec le fichier `.env` et n'est pas intégrée à l'image.
