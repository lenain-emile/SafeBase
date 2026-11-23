# 1. Récupérer le projet
git clone https://github.com/lenain-emile/SafeBase.git
cd SafeBase

# 2. Créer le fichier d'environnement local
# (Copier le modèle d'environnement ou créer un fichier .env avec les variables de BDD)
cp .env.example .env   # Ou créer un .env manuellement

# 3. Lancer les conteneurs Docker (PostgreSQL + Node)
# Si la stack a déjà tourné avec d'autres identifiants, lancer d'abord
# `docker-compose down -v` : Postgres ne réinitialise pas un volume existant.
# -V recrée le volume node_modules : sans lui, une dépendance ajoutée
# au package.json n'est pas visible dans le conteneur.
docker-compose up -d --build -V

# 4. Appliquer les migrations de la base de données
docker-compose exec server-node npx prisma migrate dev

# 5. (Optionnel) Générer le client Prisma si besoin
docker-compose exec server-node npx prisma generate

## Sauvegardes planifiées et rétention

Deux variables du `.env` règlent les sauvegardes automatiques :

| Variable | Exemple | Rôle |
|---|---|---|
| `BACKUP_CRON` | `0 3 * * *` | Quand sauvegarder, au format cron `minute heure jour mois jour-de-semaine` (ici tous les jours à 3h00). Vide = désactivé. Pour tester : `* * * * *` (chaque minute). |
| `BACKUP_RETENTION` | `7` | Nombre de sauvegardes horodatées gardées par base. Les plus anciennes sont supprimées. Le fichier `<base>_latest.sql` n'est jamais supprimé. `0` = tout garder. |

Les sauvegardes planifiées apparaissent dans l'historique de tous les utilisateurs avec la mention « (auto) ».

## Restauration

Le bouton « Restaurer » de l'historique rejoue une sauvegarde dans sa base d'origine. Les données actuelles de la base sont remplacées.

La base `safebase_db` n'est pas restaurable depuis l'interface : elle contient les comptes et l'historique de SafeBase lui-même, qu'une restauration effacerait.

## CI/CD

Le pipeline `.github/workflows/ci.yml` tourne sur chaque PR vers `main`, sur chaque push sur `main` et sur chaque tag `v*` :

1. **CI** (en parallèle) :
   - `server` : vérification des types (`npm run typecheck`), tests (`npm test`), et migrations Prisma appliquées sur un Postgres 15 vide et synchronisées avec `schema.prisma`
   - `worker-go` : `go vet`, tests, `govulncheck` (failles connues)
   - `integration` : vraie sauvegarde (`pg_dump` 15) puis vraie restauration (`psql`) sur Postgres 15
   - `client` : build du client React
2. **CD** (seulement si toutes les vérifications passent) : construction des 3 images Docker, publiées sur GitHub Container Registry :
   - `ghcr.io/lenain-emile/safebase-server`
   - `ghcr.io/lenain-emile/safebase-worker-go`
   - `ghcr.io/lenain-emile/safebase-client`

   | Événement | Image construite | Tags publiés |
   |---|---|---|
   | PR vers `main` | production (vérification) | aucun, rien n'est publié |
   | push sur `main` | **développement** (étape `dev`) | `dev`, `dev-<commit>` |
   | tag `v1.2.3` | **production** (étape `prod`) | `1.2.3`, `latest` |
3. **Notification Google Chat** (à chaque exécution) : statut, commit, auteur, lien vers le run et, en cas d'échec, les jobs et étapes en échec avec les lignes d'erreur des logs. Nécessite le secret `GOOGLE_CHAT_WEBHOOK` (voir ci-dessous).

Lancer le test d'intégration en local (Postgres 15 et client 15 requis) :

```bash
cd worker-go
PGHOST=localhost PGUSER=client_user PGPASSWORD=client_password PGDATABASE=client_db \
  go test -tags=integration -run Integration -v ./...
```

### Faire une release

```bash
git tag v1.0.0
git push origin v1.0.0
```

### Lancer les images de production

Avec les variables de `.env` remplies (et `docker login ghcr.io` tant que les images sont privées) :

```bash
TAG=1.0.0 docker compose -f docker-compose.prod.yml up -d
```

Pour revenir à une version précédente, relancer avec son numéro (`TAG=0.9.0`). Les migrations Prisma, elles, ne reviennent pas en arrière.

### Mettre en place les notifications Google Chat (une seule fois)

1. Dans Google Chat, ouvrir l'espace qui doit recevoir les notifications → **Applications et intégrations** → **Webhooks** → **Ajouter un webhook**, puis copier l'URL.
2. L'enregistrer comme secret du dépôt : `gh secret set GOOGLE_CHAT_WEBHOOK` (puis coller l'URL).

Sans ce secret, le job `notify` affiche un avertissement et la CI continue.
