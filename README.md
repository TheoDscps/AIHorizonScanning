# AI Horizon Scanning — Veille Technologique IA

Pipeline automatisé de veille technologique sur l'IA pour la programmation.

**Stack :** n8n Community (Docker) + Claude Haiku (Anthropic) + Hacker News RSS
**Dashboard :** https://theodscps.github.io/AIHorizonScanning/

---

## Démarrage rapide

### 1. Cloner le projet

```bash
git clone https://github.com/TheoDscps/AIHorizonScanning.git
cd AIHorizonScanning
```

### 2. Configurer les variables d'environnement

```bash
cp .env.example .env
# Éditer .env et renseigner ta clé API Anthropic
```

### 3. Lancer n8n avec Docker Compose

```bash
docker-compose up -d
```

n8n est accessible sur **http://localhost:5678**

### 4. Importer le workflow

1. Ouvrir n8n → menu gauche → **Workflows → Import from file**
2. Sélectionner `n8n/workflow.json`
3. Dans le nœud **"Message a model"**, configurer ta credential Anthropic
4. Cliquer **"Publish"** pour activer

---

## Structure du projet

```
AIHorizonScanning/
├── n8n/
│   └── workflow.json              # Workflow n8n importable directement
├── docs/                          # Site GitHub Pages
│   ├── index.html                 # Dashboard des articles collectés
│   └── data/
│       └── veille.json            # Données de veille (à mettre à jour)
├── docker-compose.yml             # Lance n8n en un seul commande
├── .env.example                   # Template des variables d'environnement
└── guide_pipeline_vt_n8n.md       # Guide d'installation détaillé
```

---

## Mettre à jour le dashboard

Après chaque collecte n8n, exporte les données dans `docs/data/veille.json` :

```json
[
  {
    "titre": "Titre de l'article",
    "lien": "https://...",
    "date": "Mon, 24 Mar 2026 10:00:00 +0000",
    "pertinent": true,
    "score": 8,
    "raison": "Explication de Claude en une phrase",
    "categorie": "IDE"
  }
]
```

Puis :

```bash
git add docs/data/veille.json
git commit -m "Update veille data"
git push
```

GitHub Pages se met à jour automatiquement en quelques secondes.

---

## Architecture du pipeline n8n

```
RSS Feed Trigger → Message a model (Claude) → Code JS (parse) → Filter → Convert to File → Code JS (écriture CSV)
```

| Nœud | Type | Rôle |
|------|------|------|
| RSS Feed Trigger | Trigger | Récupère articles HN chaque lundi à 14h |
| Message a model | Anthropic | Analyse la pertinence avec Claude Haiku |
| Code (parser) | JavaScript | Parse la réponse JSON de Claude |
| Filter | Filter | Ne garde que les articles avec score ≥ 6 |
| Convert to File | Utilitaire | Convertit en CSV |
| Code (écriture) | JavaScript | Sauvegarde dans le fichier CSV persistant |

---

## Commandes Docker utiles

```bash
# Démarrer / arrêter
docker-compose up -d
docker-compose down

# Voir les logs
docker-compose logs -f

# Créer le dossier de stockage CSV (première fois)
docker exec -it n8n sh -c "mkdir -p /home/node/.n8n/veille && chmod 777 /home/node/.n8n/veille"

# Lire le CSV
docker exec -it n8n sh -c "cat /home/node/.n8n/veille/veille_technologique.csv"
```

---

## Prochaines étapes

- [ ] Ajouter sources RSS (Anthropic blog, OpenAI blog, r/MachineLearning)
- [ ] Synthèse hebdomadaire automatique avec Claude Sonnet
- [ ] Export automatique CSV → JSON vers `docs/data/veille.json`
- [ ] Email digest hebdomadaire
- [ ] Scoring pondéré par catégorie

---

*Bachelier IT — Veille Technologique, 2025-2026*
