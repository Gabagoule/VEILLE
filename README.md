# Veille URCV — version web

Veille bibliographique en réanimation cardio-vasculaire, utilisable sur PC et sur téléphone depuis GitHub Pages. Tout tourne dans le navigateur ; un Worker Cloudflare synchronise les appareils.

```
index.html              l'application complète
manifest.webmanifest    installation sur l'écran d'accueil
icon.svg, icon-*.png    icônes
.nojekyll               publication telle quelle par GitHub Pages
worker/worker.js        Worker Cloudflare (synchro + relais)
worker/wrangler.toml    configuration du Worker
```

## 1. Déployer le Worker (une fois)

Depuis le dossier `worker/`, avec Node installé :

```bash
npx wrangler login
npx wrangler kv namespace create VEILLE
```

1. Copier l'`id` affiché dans `wrangler.toml` (ligne `id = "REMPLACER..."`).
2. Vérifier `ALLOWED_ORIGINS` : l'adresse de ta page, sans chemin ni `/` final (ex. `https://gabagoule.github.io`).
   - Si l'app est aussi servie sur `gabagoule.net`, ajoute cette adresse en la séparant de la première par une virgule.
3. Choisir un jeton long et le poser en secret :

```bash
npx wrangler secret put SYNC_TOKEN
npx wrangler deploy
```

Note l'adresse obtenue (`https://veille-urcv.<compte>.workers.dev`).

Même chose sans terminal, depuis le tableau de bord Cloudflare :
1. Créer un Worker et coller `worker.js`.
2. Créer un namespace KV et le lier sous le nom `VEILLE`.
3. Ajouter la variable `ALLOWED_ORIGINS` et le secret `SYNC_TOKEN`.

## 2. Publier l'app

1. Pousser le contenu du dossier dans un dépôt (ex. `VEILLE`).
2. Dans Settings, puis Pages, publier la branche `main`, dossier racine.

L'app sera sur `https://gabagoule.github.io/VEILLE/`.

Le dépôt ne contient aucune donnée ni aucune clé : le dossier `worker/` est public mais sans secret.

## 3. Configurer chaque appareil

Onglet **Réglages** :
- **Synchronisation** : adresse du Worker et jeton, puis « Enregistrer et synchroniser ». Un appareil neuf récupère alors tout : thèmes, articles, tris et historique des scans.
- **Accès aux sources** :
  - e-mail ;
  - clé NCBI (scans environ trois fois plus rapides) ;
  - clé OpenAlex (gratuite sur openalex.org ; sans elle, l'enrichissement se fait article par article).

  Les clés restent sur l'appareil : à saisir sur chacun.

**Installer sur le téléphone** :
- iPhone : dans Safari, bouton Partager, puis « Sur l'écran d'accueil ».
- Android : dans Chrome, menu, puis « Installer l'application ».

## Fonctionnement

**Thèmes** (graphe) :
- Glisser un thème sur un autre le rattache.
- Un double-clic ou un double tap dans le vide crée un thème ; le pincement zoome.
- Un thème sans terme est un dossier : il regroupe ses enfants sans être scanné.
- Types de termes : Auto (mapping PubMed), Titre/abstract, MeSH, syntaxe brute. Des suggestions MeSH sont disponibles.
- Chaque thème est croisé avec le filtre réa cardio-vasculaire (modifiable dans les Réglages, désactivable par thème).

**Exclusions** (bouton en haut) :
- Exclusions globales et préréglages (animal, pédiatrie, case reports…).
- Exclusions propres à chaque thème, dans l'inspecteur.

**Scan** :
- Les thèmes sont traités un par un.
- Premier scan d'un thème : un an en arrière (réglable). Ensuite, seules les nouveautés depuis le dernier scan, même si ce scan a été fait sur un autre appareil.
- Les citations des articles de moins de 2 ans sont rafraîchies chaque semaine.
- Garder l'app ouverte pendant le scan : l'écran reste allumé sur les navigateurs qui le permettent.
- Option de scan automatique à l'ouverture si le dernier scan est ancien.

**Score URCV** (0 à 100) : niveau de preuve, citations et tendance, fraîcheur, impact du journal, priorité du thème. Les poids sont réglables.

**Triage** : glisser la carte, ou utiliser le clavier.

| Geste ou touche | Effet |
|---|---|
| → | À lire |
| ← | Ignorer |
| ↑ | Pépite |
| ↓ | Moins de ce genre |
| Z | Annuler |
| O | Ouvrir dans PubMed |

Sur téléphone, dans l'abstract, seul le glissement horizontal est pris en compte (le vertical fait défiler).

**Bibliothèque** : filtres combinables, vues enregistrées, sélection multiple (sur PC).

**Lecture** : tableau À lire, Lu, Journal club, Archivé.
- Souris : glisser.
- Téléphone : appui long, puis glisser.

## Synchronisation

- Découpage : un document « core » (thèmes, exclusions, réglages, tris, scans) et 32 paquets d'articles, compressés.
- Déclenchement : à l'ouverture, 8 secondes après une modification, toutes les 5 minutes et au retour sur l'app.
- Fusion : élément par élément, la modification la plus récente l'emporte. Si deux appareils écrivent en même temps, le second recharge, fusionne et renvoie : rien n'est perdu.
- Relais : si un réseau (hôpital) bloque l'accès direct à PubMed, iCite ou OpenAlex, l'app passe automatiquement par le Worker.
- Quotas KV gratuits (1 000 écritures par jour) : une synchro n'écrit que les documents modifiés, donc un usage normal reste très en dessous.

## Sauvegarde et migration

Réglages > **Exporter** télécharge un JSON complet.

**Importer** accepte :
- ce format ;
- l'export de la version Python (`data/exports/`), converti automatiquement.

## Dépannage

| Symptôme | Cause probable |
|---|---|
| « Worker injoignable » | Adresse erronée, ou `ALLOWED_ORIGINS` ne correspond pas exactement à l'adresse de la page (protocole et domaine). |
| « Jeton refusé » | Le jeton de l'appareil diffère du secret `SYNC_TOKEN`. |
| « PubMed injoignable… » | Réseau filtrant : configurer la synchro pour activer le relais. |
| Un appareil ne voit pas la dernière modification | Le KV peut mettre quelques secondes à propager : relancer la synchro. |
