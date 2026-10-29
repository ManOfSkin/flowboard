Sur **Fly.io**, l'approche Infrastructure as Code (IaC) repose sur le couple **`fly.toml`** (qui déclare l'architecture réseau, la région, la RAM, la mise en veille automatique et les variables d'environnement) et **`Dockerfile`** (qui décrit la construction de l'environnement Ruby on Rails).

---

## 1. Le fichier `fly.toml` (Déclaration de l'infrastructure)

Crée un fichier nommé `fly.toml` à la racine de ton dépôt `flowboard`. Cette configuration déploie le serveur à Paris (`cdg`), exécute automatiquement les migrations de base de données à chaque mise en ligne, et active le **Scale-to-Zero** (`auto_stop_machines`) pour mettre le serveur en veille lorsqu'aucun recruteur ne visite le site :

```toml
# fly.toml - Infrastructure as Code pour Flowboard
app = "flowboard-portfolio" # Doit être unique sur Fly.io (modifie-le si déjà pris)
primary_region = "cdg"      # Région Paris
console_command = "/rails/bin/rails console"

[build]
  dockerfile = "Dockerfile"

[deploy]
  # Exécute automatiquement les migrations (et crée la BDD si besoin) avant de basculer le trafic
  release_command = "./bin/rails db:prepare"

[env]
  PORT = "3000"
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"
  RAILS_SERVE_STATIC_FILES = "true"

[http_service]
  internal_port = 3000
  force_https = true
  # Mise en veille automatique quand personne ne visite le portfolio (économise les ressources)
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 0
  processes = ["app"]

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1

[[statics]]
  guest_path = "/rails/public"
  url_prefix = "/"

```

---

## 2. Procédure de déploiement étape par étape

1. **Installer l'outil en ligne de commande (flyctl):** Requis une seule fois sur ta machine locale.
Fly.io pilote toute son infrastructure depuis le terminal. Installe le binaire selon ton système d'exploitation, puis connecte ton compte :

```bash
# Sur macOS / Linux :
curl -L https://fly.io/install.sh | sh

# Sur Windows (PowerShell) :
pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"

# Connexion à ton compte Fly.io :
fly auth login

```


2. **Générer le Dockerfile optimisé pour Rails:** Uniquement si ton projet n'a pas encore de Dockerfile à la racine.
Si ton projet utilise Rails 7.1 ou supérieur, un `Dockerfile` est déjà présent à la racine. S'il n'existe pas (versions antérieures de Rails), génère-le automatiquement avec la gem officielle de Fly.io :

```bash
bundle add dockerfile-rails --group development
bin/rails generate dockerfile

```


3. **Initialiser l'application sans la déployer:**
Place-toi à la racine de ton projet `flowboard` et enregistre l'application auprès de Fly.io en conservant le fichier `fly.toml` créé plus haut :

```bash
fly launch --copy-config --no-deploy

```

*Lorsque le terminal te demande si tu souhaites écraser le fichier `fly.toml` ou configurer des services additionnels via l'assistant interactif, réponds `N` (Non) pour garder le contrôle total via ton fichier IaC, ou accepte uniquement la création de la base PostgreSQL.*


4. **Provisionner et attacher la base de données PostgreSQL:**
Crée une instance PostgreSQL et lie-la à ton application Rails. Fly.io injectera automatiquement la variable secrète `DATABASE_URL` dans ton conteneur :

```bash
# 1. Créer le cluster Postgres à Paris (cdg)
fly postgres create --name flowboard-db --region cdg --initial-cluster-size 1 --vm-size shared-cpu-1x --volume-size 1

# 2. Attacher la base de données à l'application Rails
fly postgres attach flowboard-db --app flowboard-portfolio

```

*Alternative 100 % gratuite sans carte bancaire : tu peux créer une base PostgreSQL sur **Neon.tech** ou **Supabase**, puis passer directement son URL à l'étape suivante.*


5. **Injecter les secrets de production:**
Les clés sensibles ne doivent jamais apparaître en clair dans `fly.toml`. Envoie ta clé `RAILS_MASTER_KEY` dans le coffre-fort chiffré de Fly.io :

```bash
# Sur Linux / macOS :
fly secrets set RAILS_MASTER_KEY=$(cat config/master.key)

# Sur Windows (ou manuellement), remplace par le contenu de config/master.key :
fly secrets set RAILS_MASTER_KEY=ta_cle_master_ici

```

*(Si tu utilises une base de données externe comme Neon ou Supabase au lieu de Fly Postgres, ajoute `DATABASE_URL="postgres://..."` à cette même commande).*


6. **Déployer l'infrastructure:**
Lance la construction de l'image Docker, l'exécution des migrations (`db:prepare`) et la mise en ligne :

```bash
fly deploy

```

Une fois terminé, ouvre ton projet en ligne avec :

```bash
fly open

```


7. **Automatiser l'IaC avec GitHub Actions (GitOps):** Optionnel — Déploie automatiquement à chaque git push.
Pour que chaque modification poussée sur GitHub (`fly.toml` ou code Rails) mette à jour ton serveur automatiquement :

1. Génère un jeton de déploiement dans ton terminal :

```bash
fly tokens create deploy -x 999999h

```

2. Sur ton dépôt GitHub (`ManOfSkin/flowboard`), va dans **Settings → Secrets and variables → Actions** et crée un *Repository secret* nommé `FLY_API_TOKEN` contenant ce jeton.
3. Crée le fichier `.github/workflows/fly-deploy.yml` dans ton projet :

```yaml
name: Fly Deploy
on:
  push:
    branches:
      - main
      - master

jobs:
  deploy:
    name: Deploy app
    runs-on: ubuntu-latest
    concurrency: deploy-group
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

```