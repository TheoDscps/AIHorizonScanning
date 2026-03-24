# Guide complet — Pipeline de Veille Technologique IA avec n8n + Claude

> **Objectif** : Construire un pipeline automatisé qui collecte des articles RSS sur l'IA pour la programmation, les analyse avec Claude (Anthropic), filtre les plus pertinents et les stocke dans un fichier CSV local.
>
> **Stack** : n8n Community (Docker) + API Anthropic (Claude Haiku) + Hacker News RSS
>
> **Durée estimée** : 1h30 depuis zéro

---

## Prérequis

- Docker Desktop installé et fonctionnel
- Un compte sur [console.anthropic.com](https://console.anthropic.com)
- Accès à un terminal (PowerShell ou CMD sur Windows)

---

## Partie 1 — Installation de n8n sur Docker

### Lancer n8n pour la première fois

Dans un terminal, exécute cette commande complète (une seule fois) :

```bash
docker run -d --name n8n -p 5678:5678 \
  -v C:/n8n_data:/home/node/.n8n \
  -e N8N_RESTRICT_FILE_ACCESS_TO=/ \
  -e NODE_FUNCTION_ALLOW_BUILTIN=fs,path \
  -e NODE_FUNCTION_ALLOW_EXTERNAL=* \
  n8nio/n8n
```

**Explication des paramètres :**
- `-v C:/n8n_data:/home/node/.n8n` → monte un dossier local pour persister les données (workflows, credentials)
- `N8N_RESTRICT_FILE_ACCESS_TO=/` → autorise l'écriture de fichiers depuis les nœuds
- `NODE_FUNCTION_ALLOW_BUILTIN=fs,path` → autorise les modules Node.js `fs` et `path` dans les nœuds Code
- `NODE_FUNCTION_ALLOW_EXTERNAL=*` → autorise les modules externes

> ⚠️ Sur Windows, le dossier `C:/n8n_data` sera créé automatiquement. Tes données persistent même après redémarrage.

### Démarrer/arrêter n8n

```bash
# Démarrer (après un redémarrage PC)
docker start n8n

# Arrêter
docker stop n8n

# Vérifier que n8n tourne
docker ps
```

### Accéder à n8n

Ouvre ton navigateur sur : **http://localhost:5678**

---

## Partie 2 — Configurer la clé API Anthropic

### Étape 1 — Créer un compte API Anthropic

1. Va sur [console.anthropic.com](https://console.anthropic.com)
2. Crée un compte (distinct de claude.ai — ce sont deux produits séparés)
3. Va dans **Settings → API Keys → Create Key**
4. **Copie immédiatement la clé** — elle ne sera plus visible après fermeture
5. La clé commence toujours par `sk-ant-api03-...`
6. Va dans **Billing → Add credits** et ajoute au minimum 5€

> 💡 Pour une veille technologique légère (quelques dizaines d'articles/semaine), le coût réel est inférieur à 0,50€/mois avec le modèle Haiku.

### Étape 2 — Configurer la clé dans n8n

1. Dans n8n, accède à : **http://localhost:5678/credentials**
2. Clique sur **"Add credential"**
3. Recherche **"Anthropic"** et sélectionne-le
4. Colle ta clé dans le champ **"API Key"**
5. Clique sur **"Save"**

> ⚠️ Tu verras peut-être une erreur rouge "Couldn't connect with these settings — The resource you are requesting could not be found". C'est un bug connu du nœud natif n8n qui appelle un endpoint de test inexistant. **Ignore cette erreur et sauvegarde quand même** — la credential fonctionne parfaitement dans les workflows réels.

---

## Partie 3 — Créer le workflow de veille

### Architecture du pipeline

```
RSS Feed Trigger → Message a model (Claude) → Code JS (parse) → Filter → Convert to File → Code JS (écriture CSV)
```

### Étape 1 — Créer un nouveau workflow

1. Dans n8n, clique sur **"Workflows"** dans la navigation gauche
2. Clique sur **"+ New workflow"**
3. Renomme-le **"VT - Pipeline RSS IA"** (clique sur le nom en haut)
4. Sauvegarde avec **Ctrl+S**

### Étape 2 — Nœud 1 : RSS Feed Trigger

Ce nœud remplace à la fois le trigger planifié ET la lecture RSS.

1. Sur le canvas, clique sur **"Add first step..."**
2. Sélectionne **"On a schedule"** → puis cherche **"RSS"** → sélectionne **"RSS Feed Trigger"**
3. Configure :
   - **Mode** → `Every Week`
   - **Hour** → `14`
   - **Minute** → `0`
   - **Weekday** → `Monday`
   - **Feed URL** → `https://hnrss.org/frontpage`

> 💡 `hnrss.org/frontpage` retourne les articles de la page d'accueil de Hacker News. Claude se chargera de filtrer ceux qui concernent l'IA pour la programmation.

**Pour tester :** Clique sur **"Fetch Test Event"** — tu devrais voir des articles avec les champs `title`, `link`, `contentSnippet`, `pubDate`.

### Étape 3 — Nœud 2 : Message a model (Claude)

1. Clique sur **"+"** après le nœud RSS
2. Recherche **"Anthropic"** → sélectionne **"Message a model"**
3. Configure :
   - **Credential** → `Anthropic account` (sélectionné automatiquement)
   - **Model** → `claude-haiku-4-5-20251001`
   - **Prompt** → colle ce texte :

```
Tu es un assistant de veille technologique spécialisé en IA pour la programmation.

Analyse cet article et réponds UNIQUEMENT par un JSON avec ce format :
{
  "pertinent": true/false,
  "score": 1-10,
  "raison": "explication en une phrase",
  "categorie": "IDE / Agent CLI / LLM / Outil / Autre"
}

Est pertinent si l'article parle de : IDE AI-native (Cursor, Windsurf...), agents CLI (Claude Code, Codex...), LLMs pour le coding, outils d'automatisation dev par IA.

Titre : {{ $json.title }}
Contenu : {{ $json.contentSnippet }}
```

   - **Role** → `User`

> 💡 Les `{{ $json.title }}` et `{{ $json.contentSnippet }}` sont des variables n8n qui injectent automatiquement les données de chaque article RSS.

### Étape 4 — Nœud 3 : Code in JavaScript (parser)

Ce nœud extrait le JSON retourné par Claude et combine les données RSS + analyse.

1. Clique sur **"+"** après le nœud Anthropic
2. Recherche **"Code"** → sélectionne **"Code"**
3. Colle ce script :

```javascript
const claudeText = $input.item.json.content[0].text;

// Enlève les backticks markdown si présents (```json ... ```)
const cleaned = claudeText.replace(/```json|```/g, '').trim();

// Parse le JSON retourné par Claude
const parsed = JSON.parse(cleaned);

// Récupère les données RSS depuis le nœud source
const rss = $('RSS Feed Trigger').item.json;

return {
  json: {
    titre: rss.title ?? '',
    lien: rss.link ?? '',
    date: rss.pubDate ?? '',
    pertinent: parsed.pertinent,
    score: Number(parsed.score),
    raison: parsed.raison,
    categorie: parsed.categorie
  }
};
```

> ⚠️ Le `.replace(/```json|```/g, '')` est indispensable — Claude entoure parfois sa réponse JSON de backticks markdown. Sans ce nettoyage, le `JSON.parse` échoue.

### Étape 5 — Nœud 4 : Filter

Ce nœud ne laisse passer que les articles avec un score ≥ 6.

1. Clique sur **"+"** après le nœud Code
2. Recherche **"Filter"** → sélectionne **"Filter"**
3. Configure la condition :
   - **value1** → `{{ $json.score }}`
   - **opérateur** → `is greater than or equal to`
   - **value2** → `6`

### Étape 6 — Nœud 5 : Convert to File

1. Clique sur **"+"** après le nœud Filter (branche **"Kept"**)
2. Recherche **"Convert to File"** → sélectionne-le
3. L'opération **"Convert to CSV"** est déjà sélectionnée par défaut → **ne rien changer**

### Étape 7 — Nœud 6 : Code in JavaScript (écriture CSV)

Ce nœud écrit les données dans un fichier CSV persistant sur le serveur Docker.

1. Clique sur **"+"** après le nœud Convert to File
2. Ajoute un nouveau nœud **"Code"**
3. Colle ce script :

```javascript
const fs = require('fs');
const path = '/home/node/.n8n/veille/veille_technologique.csv';

// Récupère les données depuis le nœud Code précédent
const data = $('Code in JavaScript').item.json;

const titre = data.titre ?? 'N/A';
const lien = data.lien ?? 'N/A';
const date = data.date ?? 'N/A';
const pertinent = data.pertinent ?? false;
const score = data.score ?? 0;
const raison = (data.raison ?? 'N/A').replace(/"/g, "'");
const categorie = data.categorie ?? 'N/A';

const header = 'titre,lien,date,pertinent,score,raison,categorie\n';
const ligne = `"${titre}","${lien}","${date}",${pertinent},${score},"${raison}","${categorie}"\n`;

// Crée le fichier avec l'en-tête s'il n'existe pas, sinon ajoute une ligne
if (!fs.existsSync(path)) {
  fs.writeFileSync(path, header + ligne);
} else {
  fs.appendFileSync(path, ligne);
}

return { json: { statut: 'sauvegardé', titre: titre, score: score } };
```

> ⚠️ Le nœud s'appelle **"Code in JavaScript"** dans n8n — vérifie que `$('Code in JavaScript')` correspond bien au nom exact de ton 3ème nœud. Si tu l'as renommé, adapte ce nom dans le script.

---

## Partie 4 — Créer le dossier de stockage

Avant de lancer le workflow, crée le dossier où le CSV sera écrit :

```bash
docker exec -it n8n sh -c "mkdir -p /home/node/.n8n/veille && chmod 777 /home/node/.n8n/veille"
```

Le fichier CSV sera accessible depuis ton Windows dans :
```
C:\n8n_data\veille\veille_technologique.csv
```

---

## Partie 5 — Tester et activer le workflow

### Test manuel

1. Sur le canvas, clique sur **"Test workflow"** (bouton en haut)
2. Le workflow va s'exécuter avec un article réel de HN
3. Vérifie chaque nœud (coche verte = OK, croix rouge = erreur)
4. Ouvre `C:\n8n_data\veille\veille_technologique.csv` pour voir le résultat

### Activation en production

1. Clique sur **"Publish"** en haut à droite
2. Le workflow se déclenchera automatiquement chaque lundi à 14h
3. Seuls les articles avec score ≥ 6 seront enregistrés

---

## Structure du CSV de sortie

| Colonne | Description | Exemple |
|---------|-------------|---------|
| `titre` | Titre de l'article | "Cursor raises $100M..." |
| `lien` | URL de l'article | https://... |
| `date` | Date de publication | Tue, 24 Mar 2026 |
| `pertinent` | Jugement de Claude | true / false |
| `score` | Score de pertinence (1-10) | 9 |
| `raison` | Explication de Claude | "Article sur Cursor..." |
| `categorie` | Catégorie détectée | IDE / Agent CLI / LLM / Outil / Autre |

---

## Dépannage — Erreurs fréquentes

| Erreur | Cause | Solution |
|--------|-------|----------|
| `The file is not writable` | Permissions Docker ou variable d'env manquante | Relancer Docker avec `NODE_FUNCTION_ALLOW_BUILTIN=fs,path` |
| `Module 'fs' is disallowed` | Variable d'env manquante | Relancer Docker avec `NODE_FUNCTION_ALLOW_BUILTIN=fs,path` |
| `Status code 429` sur RSS | Trop de requêtes vers hnrss.org | Attendre 2-3 minutes et réessayer |
| `Couldn't connect` sur Anthropic credential | Bug du test de connexion n8n | Ignorer, sauvegarder quand même |
| Données `undefined` dans le CSV | Mauvais nom de nœud dans `$('Code in JavaScript')` | Vérifier le nom exact du nœud dans le script |
| JSON.parse error | Claude retourne des backticks markdown | Vérifier que le `.replace(/\`\`\`json\|\`\`\`/g, '')` est bien présent |

---

## Commandes Docker utiles

```bash
# Démarrer n8n
docker start n8n

# Arrêter n8n
docker stop n8n

# Voir les logs en temps réel
docker logs -f n8n

# Ouvrir un terminal dans le container
docker exec -it n8n sh

# Lire le CSV depuis le terminal
docker exec -it n8n sh -c "cat /home/node/.n8n/veille/veille_technologique.csv"

# Vider le CSV (garder l'en-tête)
docker exec -it n8n sh -c "echo 'titre,lien,date,pertinent,score,raison,categorie' > /home/node/.n8n/veille/veille_technologique.csv"
```

---

## Prochaines étapes (Phase 2)

- [ ] Ajouter d'autres sources RSS (blog Anthropic, blog OpenAI, r/MachineLearning...)
- [ ] Générer une synthèse hebdomadaire automatique avec Claude Sonnet
- [ ] Créer une page web dynamique alimentée par le CSV
- [ ] Envoyer un email de digest hebdomadaire
- [ ] Ajouter un scoring pondéré par catégorie

---

*Guide rédigé dans le cadre d'un projet de Veille Technologique — Bachelier IT, 2025-2026*
