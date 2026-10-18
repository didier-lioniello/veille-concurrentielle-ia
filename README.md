# Veille concurrentielle IA avec rapport hebdomadaire (n8n)

## Objectif
Surveiller automatiquement les concurrents sur plusieurs sources (sites web, flux RSS, X, LinkedIn, offres d'emploi, pages de prix), ne garder que **ce qui a changé depuis la semaine précédente**, faire analyser ces changements par l'IA, les classer par importance, puis produire **un rapport hebdomadaire** (email et Notion). Un **signal fort** (baisse de prix, nouveau produit, levée de fonds, recrutement massif sur un poste clé…) déclenche une alerte immédiate à l'équipe interne.

## Schéma

```mermaid
flowchart TD
    CFG[(Liste des concurrents<br/>Google Sheets / Notion<br/>URLs, comptes, mots-clés)] --> S0

    subgraph COLLECTE[1. Collecte quotidienne]
        S0[Schedule Trigger<br/>chaque jour] --> L1[Loop Over Items<br/>chaque concurrent]
        L1 --> S1[Sites + pages de prix<br/>HTTP Request + HTML]
        L1 --> S2[Blogs et actus<br/>RSS Read]
        L1 --> S3[X<br/>API X via HTTP Request]
        L1 --> S4[LinkedIn<br/>source autorisée ou saisie manuelle]
        L1 --> S5[Offres d'emploi<br/>page carrière / RSS]
        S1 --> M[Merge]
        S2 --> M
        S3 --> M
        S4 --> M
        S5 --> M
        M --> F1{Source en erreur ?}
        F1 -- Oui --> F2[Log + compteur d'échecs<br/>on continue sans cette source]
        F1 -- Non --> N1[Normalisation<br/>concurrent, source, date, texte, URL]
    end

    subgraph DIFF[2. Déduplication + diff]
        N1 --> D1[Empreinte du contenu<br/>Crypto]
        D1 --> D2[Remove Duplicates<br/>déjà vu ?]
        D2 --> D3[Compare Datasets<br/>vs snapshot précédent]
        D3 --> D4{Changement réel ?}
        D4 -- Non --> D5[Ignorer]
        D4 -- Oui --> D6[(Snapshot Postgres<br/>mise à jour)]
    end

    subgraph ANALYSE[3. Analyse IA + scoring]
        D6 --> A1[LLM : résumé du changement<br/>catégorie : prix, produit, recrutement,<br/>partenariat, marketing, financement]
        A1 --> A2[Sortie JSON<br/>+ citation et lien source]
        A2 --> A3[Code : score d'importance 1 à 5<br/>règles + catégorie + concurrent]
        A3 --> A4[(Table des signaux)]
        A3 --> A5{Signal fort ?<br/>score 5}
    end

    subgraph ALERTE[4. Alerte temps réel - interne]
        A5 -- Oui --> AL1{Déjà alerté<br/>sur ce sujet ?}
        AL1 -- Non --> AL2[Slack / Telegram<br/>résumé + lien source]
        AL1 -- Oui --> AL3[Ajout au fil existant]
    end

    subgraph HEBDO[5. Rapport hebdomadaire]
        W0[Schedule Trigger<br/>chaque lundi matin] --> W1[Lire les signaux<br/>des 7 derniers jours]
        A4 -.-> W1
        W1 --> W2{Au moins un signal ?}
        W2 -- Non --> W3[Rapport court :<br/>rien de notable]
        W2 -- Oui --> W4[Aggregate par concurrent<br/>et par catégorie]
        W4 --> W5[LLM : synthèse,<br/>tendances, points d'attention]
        W5 --> W6[Validation humaine<br/>optionnelle avant diffusion]
        W6 --> W7[Page Notion du rapport]
        W6 --> W8[Email à l'équipe interne]
        W3 --> W8
    end

    W7 --> FB[Feedback équipe :<br/>signal utile / bruit]
    FB --> CFG

    ERR[Error Trigger] --> ERR2[Alerte Slack<br/>workflow + source en échec]
```

## Explication étape par étape

1. **Configuration** : la liste des concurrents vit dans un Google Sheet ou une base Notion (nom, URLs du site et des pages de prix, flux RSS, comptes X, page carrière, mots-clés). Ajouter un concurrent ne demande aucune modification du workflow.
2. **Collecte quotidienne** : un `Schedule Trigger` lance la collecte, puis `Loop Over Items` parcourt chaque concurrent.
   - **Sites et prix** : `HTTP Request` + `HTML` (extraction par sélecteur CSS) pour cibler la zone utile de la page (bloc de prix, liste de produits) plutôt que la page entière.
   - **RSS** : `RSS Read` pour les blogs, communiqués et actualités.
   - **X** : l'API X via `HTTP Request` ou le nœud `X (Formerly Twitter)`, selon l'accès API disponible.
   - **LinkedIn** : LinkedIn limite fortement la collecte automatisée ; on utilise une source autorisée (API officielle si accès, service tiers conforme) ou une saisie manuelle via formulaire. Pas de scraping de LinkedIn.
   - **Offres d'emploi** : page carrière du concurrent ou flux RSS d'offres. Un pic de recrutement sur un métier donne une indication sur sa stratégie.
3. **Tolérance aux pannes** : chaque source peut échouer (site modifié, quota API). L'échec est journalisé avec un compteur ; la collecte continue pour les autres sources. Au-delà de quelques échecs consécutifs, l'Error Trigger alerte.
4. **Normalisation** : tous les éléments passent au même format (concurrent, source, date, texte, URL).
5. **Déduplication** : une empreinte du contenu (`Crypto`) et `Remove Duplicates` écartent ce qui a déjà été vu (même article repris sur plusieurs sources).
6. **Diff vs semaine précédente** : `Compare Datasets` compare au dernier snapshot stocké dans Postgres. Seuls les vrais changements continuent (nouveau prix, nouvelle offre, nouvelle page) ; un simple changement de mise en page est ignoré.
7. **Analyse IA** : le LLM résume chaque changement, le classe (prix, produit, recrutement, partenariat, marketing, financement) et renvoie un JSON avec **la citation et le lien source**, pour que chaque affirmation soit vérifiable.
8. **Scoring d'importance** : un nœud `Code` attribue un score de 1 à 5 selon des règles fixées par l'équipe (catégorie, concurrent prioritaire, ampleur du changement). Exemple de règle, à adapter : une baisse de prix sur un produit concurrent direct = score 5.
9. **Alerte temps réel** : un score 5 déclenche immédiatement un message Slack ou Telegram **à l'équipe interne**, avec le résumé et le lien. Un sujet déjà alerté ne l'est pas une seconde fois : l'info s'ajoute au fil existant.
10. **Rapport hebdomadaire** : un second `Schedule Trigger` (par exemple le lundi matin) lit les signaux des 7 derniers jours, les regroupe (`Aggregate`) par concurrent et catégorie, puis le LLM rédige une synthèse : faits marquants, tendances, points d'attention. Si la semaine est calme, le rapport le dit simplement.
11. **Diffusion** : page Notion (historique consultable) et email à la liste interne. Une validation humaine avant diffusion est possible au démarrage.
12. **Boucle d'amélioration** : l'équipe marque les signaux utiles ou relevant du bruit ; les règles de scoring et la liste des sources sont ajustées en conséquence.

## Nœuds n8n principaux
`Schedule Trigger` → `Google Sheets` / `Notion` (config) → `Loop Over Items` → `HTTP Request` + `HTML` / `RSS Read` / `X (Formerly Twitter)` → `Merge` → `Edit Fields (Set)` → `Crypto` → `Remove Duplicates` → `Compare Datasets` → `Postgres` (snapshots, signaux) → `OpenAI` / `Basic LLM Chain` + `Structured Output Parser` → `Code` (score) → `IF` → `Slack` / `Telegram` (alertes) ; `Schedule Trigger` hebdo → `Postgres` → `Aggregate` → LLM → `Notion` + `Gmail` / `Send Email` ; `Error Trigger` → `Slack`.

## Garde-fous
- **Faits sourcés uniquement** : chaque signal du rapport porte son lien source et sa date ; l'IA résume, elle n'invente pas d'information sur un concurrent. Les déductions sont présentées comme des hypothèses.
- **Diffusion interne uniquement** : rapports et alertes vont à l'équipe, jamais à des tiers ; aucun message n'est envoyé aux concurrents ni à leurs contacts.
- **Légalité de la collecte** : respecter les conditions d'utilisation des sites et des plateformes (notamment LinkedIn et X), le fichier robots.txt et une fréquence raisonnable ; pas de contournement de connexion ou de protection.
- **RGPD** : la veille porte sur des entreprises ; éviter de stocker des données personnelles (noms d'employés dans les offres ou les publications) au-delà du nécessaire, et fixer une durée de conservation.
- **Coûts** : le diff avant l'IA est la principale économie : on n'analyse que ce qui a changé. Modèle léger pour la classification, modèle plus puissant seulement pour la synthèse hebdo ; surveiller les quotas d'API (X notamment).
- **Sécurité** : le contenu collecté est traité comme une donnée, jamais comme une instruction (une page web peut contenir une injection de prompt) ; credentials stockés dans n8n.
