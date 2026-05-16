# modeles-infoclimat-pipeline

> **Statut au 2026-05-17** :
> - ✅ Phase 0 (fork + clone) — done
> - ✅ Phase 1 (Docker pipeline local AROME France HD) — done, produit des `.om` valides
> - ⚙️ Phase 2 (cumuls custom) — spec + plan écrits ([specs/](docs/superpowers/specs/2026-05-16-cumul-postprocessor-design.md), [plans/](docs/superpowers/plans/2026-05-16-cumul-postprocessor.md)) ; Task 0 (spike `.om` format) done, Tasks 1–7 à faire.
> - ⏳ Phases 3–5 — pas commencées.
>
> **Reprendre sur une nouvelle machine** : voir la section "Reprise" en bas de ce fichier.

## But du projet

Self-host du pipeline `open-meteo/open-meteo` (Swift, AGPL-3.0) pour produire nos propres fichiers `.om` spatiaux, avec **deux apports custom Infoclimat** :

1. **Variables de cumul de précipitation** non disponibles dans le catalogue Open-Meteo :
   - `precipitation_24h_sum` (cumul glissant 24 h)
   - `precipitation_6h_sum`
   - `precipitation_run_total` (cumul depuis t=0 du run)
   - Idem pour `rain`, `showers`, `snowfall_water_equivalent`
2. (À moyen terme) modèles non couverts par Open-Meteo : AROME OM (Outre-Mer), Antilles, Réunion, modèles Infoclimat custom.

Les fichiers produits sont consommés par le sister-project [`../modeles-infoclimat/`](../modeles-infoclimat/) (SvelteKit + MapLibre, déjà déployé sur https://modele.cmer.fr/ et future sous-domaine `modeles.infoclimat.fr`).

## Architecture cible

```
   ┌─────────────────────────┐
   │  Météo-France OpenData  │  GRIB
   │  DWD ICON               │  ↓
   │  NOAA GFS               │
   └──────────┬──────────────┘
              ↓
   ┌─────────────────────────┐
   │ open-meteo/open-meteo   │  Pipeline Swift forké
   │ (this repo, conteneurisé)│  + nos patches cumuls
   └──────────┬──────────────┘
              ↓ écriture
   ┌─────────────────────────┐
   │   data_spatial/         │  Fichiers .om
   │   <domain>/<run>/*.om   │
   └──────────┬──────────────┘
              ↓ sync
   ┌─────────────────────────┐
   │   Cloudflare R2         │  Stockage final
   │   data.infoclimat.fr    │  (egress gratuit)
   └──────────┬──────────────┘
              ↓ HTTP
   ┌─────────────────────────┐
   │   modeles-infoclimat    │  Front SvelteKit
   │   (sister project)      │  → fetch directement les .om
   └─────────────────────────┘
```

## Décisions déjà prises (ne pas re-discuter)

- **Stack** : on garde le Swift d'Open-Meteo, on ne réécrit rien. Forker, patcher.
- **Modèles à activer en v1** : AROME France HD (priorité 1) + ARPEGE Europe + ICON D2.
  - Pas d'ECMWF IFS en v1 (payant en temps réel)
  - Pas de GFS en v1 (Open-Meteo l'a déjà bien couvert, on cible la France)
- **Stratégie git pour les patches** : branche `infoclimat/main` posée sur `upstream/main`, rebase à chaque pull. Pas de dossier `patches/` séparé.
- **Storage** : Cloudflare R2. Egress gratuit = killer feature vs S3. Estimation ~10 $/mois.
- **Nom domaine** : `data.infoclimat.fr` (à valider avec l'asso quand on en sera là)
- **Compatibilité front** : on respecte STRICTEMENT le layout d'URLs Open-Meteo pour qu'un simple changement de `getBaseUri()` côté front suffise. Layout réel **confirmé en Phase 1** (les `<run>` sont exprimés comme `YYYY/MM/DD/HHMMZ`, pas en un seul segment) :
  ```
  data_spatial/<domain>/latest.json
  data_spatial/<domain>/in-progress.json
  data_spatial/<domain>/YYYY/MM/DD/HHMMZ/<timestamp>.om
  ```
- **Licence** : AGPL-3.0 (héritée upstream). Notre fork doit être public. Pas un blocker — Infoclimat fait déjà de l'open-source.

## Plan d'implémentation (à exécuter)

### Phase 0 — Setup repo (~30 min)

1. **Fork sur GitHub** : `github.com/open-meteo/open-meteo` → `cmer81/open-meteo` (ou `infoclimat-org/open-meteo` si l'asso a un GitHub org)
2. **Cloner** (cf. section "Reprise" en bas pour la commande à jour) :
   ```bash
   git clone -b infoclimat/main git@github.com:cmer81/open-meteo.git
   cd open-meteo
   git remote add upstream git@github.com:open-meteo/open-meteo.git
   ```
3. Vérifier qu'on est sur Swift toolchain compatible (cf. `Package.swift` upstream).

### Phase 1 — Setup local Docker (~1-2 jours)

1. Lire `Dockerfile` upstream + `docker-compose.yml` upstream s'il existe.
2. Créer notre `docker-compose.yml` qui :
   - Lance le sync GRIB pour AROME France HD uniquement (cron interne ou `CronjobCommand.swift`)
   - Monte un volume `./data/` vers `DATA_SPATIAL_DIRECTORY`
   - Définit les env vars (cf. `Sources/App/configure.swift` pour la liste complète)
3. Premier test : faire tourner le pipeline localement pour AROME France HD, vérifier que des `.om` apparaissent dans `./data/data_spatial/meteofrance_arome_france_hd/...`
4. Vérifier qu'un `.om` produit localement est consommable par le front en pointant `getBaseUri()` vers `http://localhost:XXXX`.

### Phase 2 — Patches cumuls (~3-5 jours)

1. Lire `Sources/App/Helper/OmSpatialTimestepWriter.swift` pour comprendre comment les variables sont écrites.
2. Lire le `MeteoFranceDomain.swift` pour voir comment les variables disponibles sont enregistrées.
3. **Implémenter** : un hook qui, avant l'écriture du `.om`, calcule les variables de cumul à partir de `precipitation` horaire.
4. Enregistrer ces nouvelles variables dans le `metaJson` pour qu'elles apparaissent dans le selector du front.
5. Tester localement.

### Phase 3 — Hosting (~2-3 jours)

1. Créer un bucket Cloudflare R2.
2. Configurer les credentials S3 (R2 expose une API compatible S3).
3. Deux options :
   - **(a)** `rclone sync ./data/data_spatial/ r2:bucket/data_spatial/` en cron post-pipeline
   - **(b)** Utiliser `SyncCommand.swift` d'Open-Meteo qui parle S3 natif, configuré pour pointer vers R2
4. Configurer un custom domain `data.infoclimat.fr` (ou un domaine perso temporaire type `data.cmer.fr` pour la BETA)
5. Headers CORS critiques (cf. `../modeles-infoclimat/CLAUDE.md` section "Headers HTTP critiques") :
   - `Access-Control-Allow-Origin: *`
   - `Cross-Origin-Resource-Policy: cross-origin`

### Phase 4 — Bascule front (~5 min)

1. Dans `../modeles-infoclimat/src/lib/helpers.ts:getBaseUri()`, ajouter une condition : si domaine commence par `infoclimat_*` ou si flag dev, pointer vers `data.cmer.fr` au lieu de `map-tiles.open-meteo.com`.
2. Re-build et push l'image Docker du front.
3. Tester en prod : les variables cumulées doivent apparaître dans le selector.

### Phase 5 — Production (~3-5 jours)

1. Migrer le container sur un VPS Infoclimat ou serveur perso de cedric (~$5-10/mois si pas déjà dispo)
2. Setup monitoring : grafana/uptime-kuma/whatever pour alerter si le pipeline plante
3. Setup rétention : cleanup des `.om` > 7 jours pour rester dans les coûts R2
4. Rotation logs Docker (cf. `../modeles-infoclimat/docker-compose.yml` pour le pattern)

## Ressources

### Repos upstream à connaître
- **Pipeline principal** : https://github.com/open-meteo/open-meteo (Swift, AGPL-3.0)
  - `Sources/App/MeteoFrance/` — pipeline AROME/ARPEGE
  - `Sources/App/Helper/OmSpatialTimestepWriter.swift` — écriture des tuiles spatiales
  - `Sources/App/configure.swift` — env vars (chercher `DATA_SPATIAL_DIRECTORY`)
  - `Sources/App/Commands/CronjobCommand.swift` — orchestration cron
- **Format `.om`** : https://github.com/open-meteo/om-file-format (C/Swift)
- **Python bindings** (utile pour scripts de validation) : https://github.com/open-meteo/python-omfiles
- **AWS Open Data mirror** : https://github.com/open-meteo/open-data — documentation du layout S3, peut servir de fallback si on veut juste mirror sans transformer.

### Sister projects (lire pour contexte)
- **`../modeles-infoclimat/`** — le front SvelteKit qui consomme nos données. Lire `CLAUDE.md` complet, puis :
  - `src/lib/helpers.ts:getBaseUri()` — point d'entrée à modifier en phase 4
  - `src/lib/url.ts:getOMUrl()` — construction de l'URL `.om?variable=X&arrows=...`
  - `src/lib/metadata.ts` — fetch de `latest.json` / `meta.json`
  - `src/lib/layers.ts` — comment le front affiche raster + vector
- **`../site-infoclimat/`** — le site PHP principal d'Infoclimat. Ignorer sauf si on parle d'intégration menu.

### Doc à lire en cours de route
- README upstream : https://github.com/open-meteo/open-meteo/blob/main/README.md
- Open-Meteo Blog (annonces de breaking changes) : https://openmeteo.substack.com
- API doc Cloudflare R2 : https://developers.cloudflare.com/r2/

## Points de vigilance

- **AGPL-3.0** : on **doit** publier notre fork. Pas un problème, mais ne pas l'oublier (notice dans README).
- **Maintenance opérationnelle** : le pipeline d'Open-Meteo casse régulièrement quand les services nationaux changent leurs formats GRIB. Compter ~quelques heures/mois de maintenance réactive (lire leur changelog upstream, rebase, redéployer).
- **Volume de données** : Open-Meteo annonce ~2 To/jour pour leur déploiement global (30+ modèles). Pour 3 modèles (AROME + ARPEGE + ICON-D2) on est plutôt ~50-100 Go/jour de download GRIB, ~10-20 Go/jour de `.om` produit. Vérifier la bande passante du serveur d'hébergement.
- **CPU** : la conversion GRIB → `.om` est CPU-intensive (compression). Compter au moins 4-8 vCPU pour tenir la cadence horaire d'AROME.
- **Concurrence avec Open-Meteo** : si on demande gentiment à Open-Meteo d'ajouter `precipitation_24h_sum` à leur catalogue spatial AVANT de se lancer, on évite peut-être tout ce projet. **Toujours vérifier** s'ils n'ont pas déjà répondu favorablement à l'issue [open-meteo/open-meteo#1580](https://github.com/open-meteo/open-meteo/issues/1580) ou similaire avant de continuer.

## Contexte historique (d'où vient ce projet)

Le projet est né d'un échange forum (à insérer ici quand on aura le lien) où un utilisateur a demandé des cartes de cumul pluvio + combinaisons de variables.

La feature **combinaisons de variables** (raster + flèches vent de variables différentes) a été livrée le 2026-05-16 dans le sister project `../modeles-infoclimat/` (commit `87997e3 feat(layers): découpler la variable des flèches du raster`) sans avoir besoin de toucher au pipeline — c'était juste une question d'UI.

La feature **cumuls pluvio** ne pouvait pas être résolue côté front (Open-Meteo n'expose pas la variable). D'où ce projet de pipeline self-host.

## TODO immédiat (avant tout dev)

- [x] ~~Vérifier sur GitHub que l'issue [open-meteo/open-meteo#1580]~~ — vérifié 2026-05-16, c'est sur le *taux instantané* pas les cumuls, pas la même demande.
- [ ] Ouvrir éventuellement une issue chez Open-Meteo pour `precipitation_24h_sum` (skip pour l'instant, on intègre le calcul nous-mêmes).
- [x] ~~Forker et démarrer Phase 0~~ — done 2026-05-16, fork à `cmer81/open-meteo`, branche `infoclimat/main`.
- [ ] Décider qui héberge à terme : serveur perso cedric vs infra Infoclimat (à discuter avec l'asso, pas encore).

## Reprise (nouvelle machine)

Pour reprendre le projet sur une autre machine (e.g. Ubuntu x64) :

```bash
# 1. Cloner le fork sur la branche de travail
git clone -b infoclimat/main git@github.com:cmer81/open-meteo.git
cd open-meteo
git remote add upstream git@github.com:open-meteo/open-meteo.git

# 2. Pré-requis système
#    - Docker Engine + Compose v2 (apt install docker.io docker-compose-plugin)
#    - netcdf-bin (pour `ncdump` lors de la vérif Task 0/6 du plan : apt install netcdf-bin)
#    - (optionnel) Swift toolchain si tu veux `swift test` en local sans Docker.
#      Sinon, Tasks 1-7 peuvent toutes tourner dans Dockerfile.development.

# 3. Configurer la clé Météo-France (gitignored, à recréer)
cp .env.example .env
# puis éditer .env, mettre METEOFRANCE_API_KEY=<ta clé eyJ4NXQi...>

# 4. Sanity check Phase 1 (re-télécharge un run AROME France HD)
docker compose up    # ~4 min, produit data/data_spatial/...

# 5. Continuer Phase 2 : suivre `docs/superpowers/plans/2026-05-16-cumul-postprocessor.md`
#    Prochaine task = Task 1 (CumulMath.swift + tests TDD).
```

État git au 2026-05-17 (push fait sur `cmer81/open-meteo`) :
- `ca782e4e` — baseline Phase 1 pipeline + Phase 2 design
- `4c6ffad8` — docs(cumul): record confirmed .om spatial file structure (Task 0 spike)
