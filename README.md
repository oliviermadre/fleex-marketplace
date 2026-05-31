# fleex-marketplace

Marketplace officielle de **primitives agentiques** pour **Fleex** — l'ADE (Agentic Development Environment).

Ce dépôt rassemble un bootstrap prêt à l'emploi : des **personas**, **skills**, **panels** et **workflows** réutilisables, que n'importe quelle instance Fleex peut installer en une commande. L'objectif est de partir d'une base saine plutôt que d'une instance vide.

Toute la gestion (enregistrer la marketplace, importer, exporter, supprimer) se fait via la **CLI Fleex** (`~/.fleex/bin/fleex`, alias `fleex`).

---

## Qu'est-ce qu'une primitive ?

Une primitive est une brique agentique versionnée. Il en existe quatre types :

| Type | `kind` | Rôle |
|---|---|---|
| **Persona** | `persona` | Un agent avec une identité, un système de valeurs (`SOUL.md`), un modèle et un mode d'exécution. |
| **Skill** | `skill` | Une compétence déclenchable (instructions Markdown) qui produit un deliverable. |
| **Panel** | `panel` | Un collectif de personas qui délibèrent ensemble (ex. comité d'architecture, chapeaux de Bono). |
| **Workflow** | `workflow` | Un enchaînement orchestré de personas et de skills. |

Les primitives ont des **dépendances** : un panel dépend de ses personas, un workflow dépend de ses skills et personas, etc. Fleex respecte ces dépendances automatiquement à l'import comme à la suppression.

### Structure du dépôt

```
fleex-marketplace/
├── marketplace.json     # Le manifeste : liste toutes les primitives + leurs dépendances
├── personas/            # Une primitive persona par fichier .json
├── skills/              # Une primitive skill par fichier .json
├── panels/              # Une primitive panel par fichier .json
└── workflows/           # Une primitive workflow par fichier .json
```

Le `marketplace.json` est la **source de vérité**. Il référence chaque primitive par son `slug`, son `path` et ses `dependencies`. La CLI maintient ce manifeste cohérent : tu n'as jamais à l'éditer à la main.

Contenu actuel : **3 panels**, **20 personas**, **7 skills**, **1 workflow**.

---

## 1. Enregistrer la marketplace

On enregistre une marketplace en clonant son dépôt git. C'est l'étape préalable à tout import.

```bash
# Enregistrer ce dépôt comme marketplace
fleex marketplace add https://github.com/<owner>/fleex-marketplace.git

# Lui donner un nom local explicite (sinon : owner-repo)
fleex marketplace add https://github.com/<owner>/fleex-marketplace.git --name officielle

# Ré-enregistrer (re-clone) si une marketplace du même nom existe déjà
fleex marketplace add https://github.com/<owner>/fleex-marketplace.git --force
```

Gérer les marketplaces enregistrées (`mp` est l'alias de `marketplace`) :

```bash
fleex marketplace list                  # lister les marketplaces enregistrées (alias : ls)
fleex marketplace update                 # pull la dernière version de TOUTES
fleex marketplace update officielle      # pull une seule marketplace
fleex marketplace remove officielle      # désenregistrer + supprimer le cache local (alias : rm)
```

---

## 2. Importer des primitives dans Fleex

`fleex import` installe des primitives d'une marketplace enregistrée dans l'instance courante. Les **dépendances sont tirées automatiquement**.

```bash
# Tout importer depuis une marketplace
fleex import --marketplace officielle --all

# Importer des primitives ciblées (format kind:slug)
fleex import --marketplace officielle --primitive persona:jarvis skill:pr-faq

# Importer un panel : ses personas dépendantes suivent automatiquement
fleex import --marketplace officielle --primitive panel:bono
```

### Gérer les conflits

Si une primitive existe déjà localement et diffère, `--on-conflict` décide du comportement (défaut : `ask`) :

```bash
# Remplacer sans poser de question, et sauter la confirmation finale
fleex import --marketplace officielle --all --on-conflict replace -y

# Conserver les versions locales en cas de conflit
fleex import --marketplace officielle --all --on-conflict skip
```

| Mode | Effet sur une primitive locale qui diffère |
|---|---|
| `ask` *(défaut)* | Demande quoi faire, au cas par cas |
| `replace` | Écrase la version locale par celle de la marketplace |
| `skip` | Conserve la version locale, n'importe pas |

> 💡 `-y, --yes` saute uniquement la **confirmation finale**, pas les choix `ask` par conflit.

---

## 3. Exporter des primitives depuis Fleex vers le dépôt

`fleex export` fait l'inverse : il extrait des primitives de ton instance Fleex et les écrit dans une marketplace (sa copie de travail git). C'est ainsi qu'on **contribue** de nouvelles primitives à ce dépôt.

```bash
# Exporter des primitives ciblées dans ce dépôt
fleex export \
  --out ~/marketplaces/fleex-marketplace \
  --name fleex-marketplace \
  --persona jarvis catalyst \
  --skill pr-faq pr-review \
  --panel bono \
  --workflow spec-dev-qa

# Tout exporter d'un coup
fleex export --out ~/marketplaces/fleex-marketplace --name fleex-marketplace --all
```

Options de sélection (combinables) :

- `--persona <slugs...>` — personas par slug
- `--skill <commandNames...>` — skills par nom de commande
- `--panel <slugs...>` — panels par slug
- `--workflow <slugs...>` — workflows par slug
- `--all` — exporte toutes les primitives

L'export met à jour `marketplace.json` (slugs, paths, dépendances) et écrit chaque primitive dans le bon dossier. Il te reste ensuite à commit/push.

> ⚠️ **Confidentialité — `memoryMd` est exclu par défaut.** L'état personnel d'une persona (sa mémoire) n'est **pas** exporté, sauf si tu passes explicitement `--include-memory`. Garde ce flag désactivé pour des primitives partageables et portables — n'inclus la mémoire que pour une sauvegarde privée.

```bash
# À NE PAS faire pour une marketplace publique : inclut l'état personnel
fleex export --out ./fleex-marketplace --persona jarvis --include-memory
```

---

## 4. Supprimer des primitives du dépôt (unexport)

`fleex unexport` retire des primitives d'une marketplace **tout en gardant le manifeste référentiellement cohérent** — il ne te laisse jamais avec une dépendance cassée.

```bash
# Retirer une primitive isolée
fleex unexport --out ~/marketplaces/fleex-marketplace --primitive skill:playground
```

### Contrôle automatique des dépendances avec `--cascade`

Par défaut, Fleex **refuse** de retirer une primitive dont d'autres dépendent (sinon le manifeste deviendrait incohérent). `--cascade` retire la cible **et tout ce qui en dépend**.

Exemple concret avec le graphe de ce dépôt — le skill `pr-faq` est utilisé par le workflow `spec-dev-qa` :

```bash
# Sans --cascade : refusé, car spec-dev-qa dépend de pr-faq
fleex unexport --out ./fleex-marketplace --primitive skill:pr-faq

# Avec --cascade : retire pr-faq ET le workflow spec-dev-qa qui en dépend
fleex unexport --out ./fleex-marketplace --primitive skill:pr-faq --cascade -y
```

Autre exemple — retirer une persona utilisée par un panel :

```bash
# persona:arbiter est membre de plusieurs panels (archi-committee, bigtech, bono)
# --cascade retire arbiter + tous les panels qui le référencent
fleex unexport --out ./fleex-marketplace --primitive persona:arbiter --cascade
```

| Option | Effet |
|---|---|
| `--out <dir>` | Répertoire de la marketplace (copie git) |
| `--primitive <kind:slug...>` | Primitives à retirer (ex. `persona:jarvis skill:search`) |
| `--cascade` | Retire aussi toutes les primitives qui dépendent des cibles |
| `-y, --yes` | Saute les confirmations |

---

## Cycle de contribution typique

```bash
# 1. J'ai créé/amélioré des primitives dans mon instance Fleex.
#    Je les exporte dans ma copie de travail du dépôt :
fleex export --out ~/marketplaces/fleex-marketplace --name fleex-marketplace \
  --persona mon-nouvel-agent --skill mon-nouveau-skill

# 2. Je relis le diff (vérifier qu'aucune donnée perso/chemin n'a leaké), puis je commit :
cd ~/marketplaces/fleex-marketplace
git add -A && git commit -m "Add mon-nouvel-agent + mon-nouveau-skill" && git push

# 3. Les autres usagers récupèrent la mise à jour :
fleex marketplace update officielle
fleex import --marketplace officielle --primitive persona:mon-nouvel-agent
```

---

## Aide-mémoire des commandes

```bash
fleex marketplace add <git-url> [--name <n>] [--force]   # enregistrer
fleex marketplace list                                   # lister (ls)
fleex marketplace update [name]                          # mettre à jour
fleex marketplace remove <name>                          # désenregistrer (rm)

fleex import --marketplace <n> (--all | --primitive kind:slug...) [--on-conflict skip|replace|ask] [-y]
fleex export --out <dir> [--name <n>] (--all | --persona... --skill... --panel... --workflow...) [--include-memory]
fleex unexport --out <dir> --primitive kind:slug... [--cascade] [-y]
```

> Référence CLI exhaustive : `fleex documentation` (alias `fleex docs`).
