# 🐍 Projet Data Pokémon — Pipeline, Data Lake et Analytics

Ce projet met en place une architecture data complète autour de la PokéAPI, en combinant :

- un pipeline d’ingestion (n8n + PostgreSQL)
- un stockage objet (MinIO)
- une couche analytique (PostgreSQL + Metabase)
- une automatisation via Telegram

---

# 🚀 Lancer le projet

## 📋 Prérequis

- Docker + Docker Compose
- Node.js (optionnel si usage avancé)
- Compte Telegram
- Compte ngrok

---

## 🐳 1. Démarrer l’environnement

Lancer tous les services :

```bash
docker compose up -d
```

Services disponibles :

- n8n → http://localhost:5678
- PostgreSQL → localhost:5432
- MinIO → http://localhost:9001

---

## 🌐 2. Exposer n8n avec ngrok (OBLIGATOIRE pour Telegram)

Telegram nécessite une URL HTTPS pour fonctionner.

### Lancer ngrok

```bash
ngrok http 5678
```

Vous obtiendrez une URL du type :

```
https://xxxxx.ngrok-free.dev
```

---

## ⚙️ 3. Configurer n8n

Dans votre `docker-compose.yml`, ajouter :

```yaml
environment:
  - WEBHOOK_URL=https://xxxxx.ngrok-free.dev
```

⚠️ Remplacer par votre URL ngrok

Puis redémarrer :

```bash
docker compose down
docker compose up -d
```

---

## 🤖 4. Configurer le bot Telegram

1. Ouvrir Telegram
2. Chercher **BotFather**
3. Créer un bot avec `/newbot`
4. Récupérer le **TOKEN**

---

## 🔗 5. Configurer n8n avec Telegram

Dans n8n :

- Ajouter des credentials Telegram
- Coller le TOKEN
- Utiliser un node **Telegram Trigger**

---

## ▶️ 6. Activer le workflow

- Publier le workflow (IMPORTANT)
- Vérifier qu’il est actif

---

## 💬 7. Tester

Dans Telegram, envoyer :

```
/help
/kpi
/types
/top
```

---

## ⚠️ Problèmes fréquents

### ❌ Erreur webhook HTTPS

➡️ Vérifier :
- ngrok est lancé
- WEBHOOK_URL est bien configuré
- workflow activé

### ❌ Le bot ne répond pas

➡️ Vérifier :
- workflow publié
- bon token Telegram
- message envoyé au bon bot

---

## ✅ Résultat attendu

- n8n reçoit les messages Telegram
- interroge PostgreSQL
- renvoie une réponse formatée

---

# 🧱 TP 2 — Pipeline ETL (Data Warehouse)

## Objectif

Construire un pipeline permettant de :

- interroger la PokéAPI
- transformer les données
- les charger dans PostgreSQL
- suivre les exécutions

---

## 🐳 Services Docker

- PostgreSQL
- n8n

---

## 🗄️ Structure SQL

```sql
create table if not exists ingestion_runs (
  id serial primary key,
  source text not null,
  started_at timestamp not null,
  finished_at timestamp,
  status text not null,
  records_received integer default 0,
  records_inserted integer default 0
);

create table if not exists pokemon (
  id serial primary key,
  pokemon_id integer not null,
  pokemon_name text not null,
  base_experience integer,
  height integer,
  weight integer,
  main_type text,
  has_official_artwork boolean default false,
  has_front_sprite boolean default false,
  source_last_updated_at timestamp,
  ingested_at timestamp not null default now(),
  run_id integer references ingestion_runs(id)
);
```

---

## ⚙️ Workflow n8n

1. Appel API PokéAPI
2. Transformation des données
3. Création d’indicateurs
4. Insertion en base
5. Suivi via ingestion_runs

---

## 🔍 Requêtes de contrôle

```sql
select count(*) from pokemon;

select count(*) from pokemon where has_official_artwork = false;

select count(*) from pokemon where has_front_sprite = false;

select main_type, count(*) from pokemon group by main_type;

select count(*) from pokemon where pokemon_name is null;
```

---

## 🧠 Justification Data Warehouse

Cette architecture correspond à une logique Data Warehouse car elle met en place un pipeline ETL structuré, avec des données transformées, historisées et prêtes pour l’analyse.

---

# 🪣 TP Data Lake — MinIO

## Objectif

Ajouter un stockage objet pour :

- conserver les données brutes
- stocker fichiers et images
- enrichir l’architecture

---

## 📁 Organisation

- raw-pokemon
- pokemon-images
- reports

---

## 🗄️ Tables ajoutées

```sql
create table if not exists pokemon_files (
  file_id serial primary key,
  pokemon_id integer,
  bucket_name text,
  object_key text,
  file_name text,
  file_type text,
  created_at timestamp default now()
);

create table if not exists file_ingestion_log (
  id serial primary key,
  file_name text,
  bucket text,
  object_key text,
  processed_at timestamp default now(),
  source text,
  status text
);
```

---

## ⚙️ Workflow

- récupération fichier
- envoi vers MinIO
- stockage des métadonnées
- lien avec Pokémon

---

## 🧠 Justification Data Lake

MinIO permet de stocker les données brutes indépendamment de leur structure. PostgreSQL contient uniquement les données analytiques et les métadonnées. Cette séparation correspond à une logique Data Lake / Lakehouse.

---

# 📊 TP 3 — Analytics & Telegram

---

## 🧱 Couche analytique

```sql
create or replace view vw_pokemon_kpi as
select
  count(*) as total,
  avg(base_experience) as avg_experience
from pokemon;

create or replace view vw_pokemon_types as
select
  main_type,
  count(*) as count
from pokemon
group by main_type
order by count desc;

create or replace view vw_pokemon_top as
select
  pokemon_name,
  base_experience
from pokemon
order by base_experience desc
limit 10;
```

---

## 📈 KPI

- volume de Pokémon
- expérience moyenne
- répartition par type
- top Pokémon

---

## 📊 Dashboard

Créé avec Metabase :

- KPI globaux
- graphique types
- classement

---

## 🤖 Telegram Bot

### Commandes

- /kpi
- /types
- /top
- /help

---

## ⚙️ Workflow n8n

1. Réception message Telegram
2. Routage commande
3. Requête SQL
4. Formatage réponse
5. Envoi message

---

## 💬 Exemple réponse

```
📊 Tableau de bord Pokémon

━━━━━━━━━━━━━━━

🔢 Volume total
• 500 Pokémon référencés

⭐ Qualité des données
• Score moyen : 3.00 / 3

🧪 Analyse
• Données complètes et cohérentes
• Aucun problème détecté

━━━━━━━━━━━━━━━
🤖 Source : PostgreSQL

This message was sent automatically with n8n
```

---

## 🧠 Réponse finale

Une couche analytique permet de simplifier l’accès aux données en transformant des tables techniques en structures lisibles. Elle facilite la réutilisation dans des outils comme Metabase ou n8n.

Les KPI ont été choisis pour fournir une lecture rapide et compréhensible du référentiel. Ils permettent d’identifier les volumes, les répartitions et les performances.

La restitution visuelle permet de piloter efficacement la qualité des données grâce à une lecture synthétique.

Telegram offre une interface interactive permettant d’interroger les données sans outil technique.

Enfin, une requête SQL est statique, tandis qu’une automatisation permet une interaction dynamique avec les données.

---

## 📸 Captures

- Dashboard Metabase
- Workflow n8n
- Réponses Telegram
