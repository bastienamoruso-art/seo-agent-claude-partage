---
name: check-pogo
description: Détection de pogo sticking et signaux comportementaux SEO en croisant GSC (CTR/position) et GA4 (engagement/durée). Adaptatif au volume du site (petit/moyen/gros). Identifie 3 types de problèmes : déception avant clic (snippet), déception après clic (contenu), et test-failure (Google a promu la page mais les signaux comportementaux sont mauvais → risque de redescente imminente).
---

# Check Pogo — Détection signaux comportementaux SEO

Pipeline d'audit qui croise GSC × GA4 pour identifier 3 types de pages à risque, avec calibration auto par volume site.

## Philosophie

**Ne jamais raisonner en seuils absolus.** Toujours comparer une page à sa cohorte interne (même type de page sur le même site). Un blog techno aura toujours plus de bounce qu'une LP locale. Les benchmarks externes ne valent rien.

**Pogo sticking ≠ bounce rate.** Le vrai signal est la combinaison `position × CTR × durée × engagement`, pondérée par volume.

**Petit site ≠ gros site.** Sur < 1000 clics/mois, la lecture pogo sticking n'est pas pertinente — pivoter vers détection pages-zombies (impressions sans clics).

## Scope strict — un skill = un job

Le skill détecte **uniquement** les 3 signaux pogo (A/B/C) + zombies sur petit site. Tout autre problème détecté est **déclaré en annexe** avec proposition de skill satellite :

| Issue détectée hors scope | Action |
|---|---|
| `(not set)` GA4 > 30% | **BLOQUANT** — refuse l'analyse, propose investigation tracking |
| `(not set)` GA4 5-30% | Avertit : "fiabilité Signal B/C dégradée", continue |
| Cannibalisation slugs | Annexe : liste de paires, audit séparé recommandé |
| Pages techniques en GA4 landings | Annexe : liste à exclure |
| Hreflang cassé | Annexe : audit hreflang recommandé |
| Pages parasites indexées | Annexe : liste à noindex |

Ne JAMAIS mélanger ces issues avec le diagnostic pogo dans la sortie principale.

## Les 3 signaux

### Signal A — Déception avant clic (snippet)
- *Mesure* : CTR observé vs CTR médian site pour le même bucket de position (1-3, 4-6, 7-10, 11-20)
- *Flag* : CTR < 50% du benchmark ET impressions ≥ seuil_min (cf. calibration)
- *Cas spécial* : CTR = 0% avec impressions ≥ seuil → flag URGENT (page invisible malgré la position)
- *Action* : refonte title/meta — l'utilisateur ne clique même pas

### Signal B — Déception après clic
- *Mesure combinée* : durée moyenne ET taux d'engagement, comparés à la médiane du **type de page** sur ce site
- *Flag* : durée < 50% médiane segment ET engagement < 25e percentile segment
- *Action* : audit above-the-fold + intent match

### Signal C — Test-failure (URGENT)
- *Mesure* : delta position entre 2 périodes couplé au signal B
- *Flag* : page qui a gagné ≥ 2 points de position **ET** signaux comportementaux dans le 25e percentile bas
- *Action* : URGENT — fix avant que Google redescende (~30j de fenêtre)

Score composite = `0.4 × C + 0.3 × A + 0.3 × B`, pondéré par clicks pour priorisation.

## Calibration auto par volume

Volume calculé sur 30 jours glissants :

| Volume | Min impressions | Min sessions | Fenêtre | Mode |
|---|---|---|---|---|
| < 1 000 clics | 50 | 5 | 90j vs 90j | **petit site** (pivot hygiène) |
| 1K – 10K | 200 | 20 | 60j vs 60j | normal |
| 10K – 100K | 1 000 | 50 | 30j vs 30j | normal |
| > 100K | 5 000 | 200 | 14j vs 14j | normal |

## Cas particuliers à gérer

### 1. Home page et brand queries
La home a souvent un CTR très élevé à position moyenne (effet brand). **Exclure systématiquement `/` du calcul des baselines CTR.** Exclure aussi les requêtes de marque.

### 2. Pages avec 0 clic et 0% CTR
Cas extrême Signal A. Si impressions ≥ seuil et CTR = 0% : flag URGENT avec gain potentiel = `CTR_benchmark × impressions`. Ces pages sont "invisibles" malgré la position.

### 3. Pages avec < seuil_sessions GA4
Exclure du Signal B et C. Logger : `X pages exclues — données insuffisantes`. Ne JAMAIS calculer un signal sur 1-2 sessions.

### 4. Fragments d'ancre GSC (`/page#section`)
Agréger les fragments sur l'URL parent. Sinon dilution massive des métriques.

### 5. Mismatch URL GSC vs GA4
GSC peut contenir l'URL canonique, GA4 le path actuel. Logique de matching :
1. Strip trailing slash
2. Strip query params
3. Match exact path
4. Fuzzy match sur slug terminal
5. Si toujours rien : logger comme `unmatched`

### 6. Saisonnalité
Si une page perd des clics et que le pattern existait l'année N-1 → marquer "saisonnier suspecté". Si pas de data N-1 → marquer "saisonnalité non vérifiable".

### 7. Pages pollution / parasites (side findings)
Détecter :
- `/page-daccueil/{image}`, `/photo-{x}`, `/logo-{x}` → médias indexés (noindex)
- Pages avec impressions sans clics répétées (zombie content)
- Pages admin/tech : `/wp-login.php`, `/admin`, `/checkout`, `/cart`, `/redirect`

### 8. Cannibalisation interne (slugs dupliqués)
Pattern fréquent sur Shopify : `/x` ET `/pages/x` qui rangent tous les deux.
- Si ratio impressions > 30% sur les 2 URLs → flag cannibalisation
- Recommander redirect 301 ou canonical

### 9. CTR catastrophique à position 1-3
Si CTR < 5% à pos ≤ 3 avec impressions ≥ seuil → flag URGENT (pas flag normal). Exemple observé : pos 1.4, 0.15% CTR sur 52K impressions = ~13K clics perdus par mois.

### 10. GA4 tracking cassé — `(not set)`
TOUJOURS détecter `(not set)` dans GA4 landingPage. Si > 5% des sessions totales → flag tracking corrompu en tête de l'output.

### 11. Hreflang/multilangue
Si URLs avec `?country=` ou `?lang=` ou path `/en/` `/de/` apparaissent dans GSC :
- Séparer les buckets par langue
- Si URL étrangère rank dans le pays principal avec 0 clic → flag hreflang

### 12. Pages techniques en landing GA4
TOUJOURS exclure du scoring (mais lister en side finding) :
- `/wp-login.php`, `/wp-admin*`
- `/cart`, `/checkout`, `/confirmation-commande`
- `/dashboard*`, `/redirect`, `/merci`

## Mode petit site (< 1000 clics/mois)

Le pogo sticking est rarement détectable par manque de signal statistique. **Pivoter vers** :

1. **Pages-zombies** : impressions ≥ 50 avec CTR < 1% sur 90j
2. **Pages parasites** : pages indexées qui ne devraient pas l'être
3. **Brand vs position** : home avec CTR fort à position lointaine → opportunité
4. **Pages locales sans visibilité** : LP locale avec 0 impressions = problème indexation

Output adapté : "Hygiène technique" plutôt que "Pogo sticking".

## Pipeline d'exécution

### Étape 0 — Setup
1. Récupérer : domaine, GSC property, GA4 property ID, brand terms (depuis un fichier de contexte ou en demandant)
2. Créer un espace de travail temporaire

### Étape 1 — Volume et calibration
1. `gsc_query_pages` sur 30 derniers jours, sort by clicks
2. Calculer total clicks_30d
3. Sélectionner le bucket de calibration
4. Si bucket = "< 1 000" → activer mode petit site

### Étape 2 — Extraction GSC
1. `gsc_query_pages` sur la fenêtre principale (selon calibration), limit 100-200
2. `gsc_compare_periods` (current vs previous selon calibration), dimensions=["page"]
3. Agréger les fragments d'ancre `/url#xxx` sur `/url`
4. Exclure la home des baselines

### Étape 3 — Extraction GA4
1. `ga4_report` avec `landingPage`, métriques `sessions, engagementRate, averageSessionDuration`, limit 200
2. Normaliser les paths (strip trailing slash, strip query)
3. Joindre avec GSC sur path normalisé. Logger les unmatched.

### Étape 4 — Classification de pages
Patterns par défaut :
- `/` → home
- `/blog/.*`, `/article/.*` → article_info
- `/product/.*`, `/p/.*` → produit
- `/{categorie}/{sous}/{...}` → category_deep
- `/{slug}` → page_top_level

### Étape 5 — Calcul des baselines (cohorte interne)
Pour chaque type de page :
- CTR médian par bucket position (1-3, 4-6, 7-10, 11-20, 21+)
- Durée moyenne médiane
- Engagement rate 25e percentile

Si moins de 5 pages dans une cohorte → utiliser baseline globale avec note.

### Étape 6 — Scoring
Pour chaque page (hors home + hors brand) :
- Signal A : CTR vs benchmark cohorte
- Signal B : durée + engagement vs cohorte type
- Signal C : delta position × Signal B
- Score composite = `0.4 × C + 0.3 × A + 0.3 × B`
- Manque à gagner = `(CTR_benchmark - CTR_actuel) × impressions`

### Étape 7 — Output

```
═══ DIAGNOSTIC POGO STICKING ═══

📊 CALIBRATION
   <volume détecté + bucket + fenêtre + seuils utilisés>

⚠ FIABILITÉ
   <% (not set) GA4 + alerte si > 5%>

🔴 URGENT — Signal C (test failure imminent)
   <pages avec position gain ≥ 2 points + signaux mauvais>

🟠 SNIPPET — Signal A (refonte title/meta)
   | page | pos | CTR | vs benchmark | impressions | manque à gagner/an |

🟡 CONTENU — Signal B (post-clic faible)
   <pages avec durée + engagement bas>

📊 BASELINES (cohorte interne site)
   <baselines par bucket position et type de page>

═══ ISSUES ANNEXES (hors scope pogo) ═══

🔧 Tracking GA4
🔀 Cannibalisation détectée
🧹 Pages techniques en landings GA4
🌍 Hreflang suspects
🗑 Pages parasites indexées
🧮 Pages exclues du diagnostic
```

Exporter en Google Sheet (si MCP disponible) avec onglets :
- `Diagnostic` : top 30 pages scorées
- `Baselines` : médianes calculées
- `Side findings` : zombies, parasites, exclus

## Règles bloquantes

- **Jamais inventer de chiffres** : si une donnée manque → `n/a` explicité
- **Toujours documenter le mode** (petit/moyen/gros) en tête d'output
- **Toujours expliquer les exclusions**
- **Toujours exporter en Sheet** pour suivi temporel

## Cas où ne PAS lancer

- Site < 100 clics/mois sur 90j → trop peu de signal, refuser et expliquer
- Pas de GA4 dispo → mode dégradé Signal A seul (le dire en tête)
- Pas d'historique 60j minimum → Signal C indisponible (le dire en tête)

## Prérequis MCP

Ce skill s'appuie sur :
- **MCP Google Search Console** — pour les données GSC
- **MCP Google Analytics 4** — pour les données comportementales GA4
- **MCP Google Sheets** (optionnel) — pour l'export du rapport
