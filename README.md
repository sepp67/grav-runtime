# grav-runtime

Socle Docker générique et réutilisable pour exécuter des sites [Grav CMS](https://getgrav.org/)
(Core + Admin) avec Nginx et PHP-FPM dans un même conteneur.

`grav-runtime` ne contient et ne génère **aucun élément propre à un site**. Il sert de base à
des images applicatives filles :

```text
grav-runtime
    ├── projet-gites   (thème, plugins métier, pages, config, secrets)
    └── grav-docs      (thème, plugins métier, pages, config, secrets)
```

## Ce que contient l'image

- Grav Core + Admin (téléchargé, vérifié par somme de contrôle, et vendorisé à la construction).
- PHP-FPM (`php:8.3-fpm-alpine`) avec les extensions `gd`, `zip`, `intl`, `mbstring`, `opcache`.
- Nginx, configuré pour servir Grav et transmettre le PHP à PHP-FPM.
- Les plugins techniques officiels du bundle Grav Admin : `admin2`, `login`, `form`, `email`,
  `error`, `problems`, `flex-objects`, `api`, `github-markdown-alerts`, `shortcode-core`.
- Le thème `quark2` (thème par défaut de Grav Admin — voir "Pourquoi `quark2` reste ici" plus bas).
- Un mécanisme générique d'initialisation des données persistantes à partir d'un répertoire seed.
- Un script d'entrypoint qui supervise Nginx et PHP-FPM, un bootstrap admin optionnel, et un
  healthcheck HTTP interne.

## Ce que l'image ne contient pas

Aucun thème métier, aucun plugin métier, aucune page, aucune photo, aucun nom de domaine, aucune
adresse IP, aucune configuration SMTP réelle, aucun compte, aucun secret, aucun mot de passe. Rien
qui référence `gites`, `projet-gites`, `grav-docs` ou un domaine réel n'est codé en dur ici — ces
éléments appartiennent exclusivement aux images applicatives filles.

Le répertoire `Ressources/` de ce dépôt contient les deux projets sources ayant servi à l'analyse
(`projet-gites`, `grav-docs`) ; il est exclu du contexte de build (`.dockerignore`) et ne fait
jamais partie de l'image.

## Construire l'image

```bash
docker build -t grav-runtime:test .
```

Arguments de build disponibles (voir aussi "Versionnement") :

| ARG | Défaut | Rôle |
|---|---|---|
| `PHP_VERSION` | `8.3` | Tag de l'image de base `php:<version>-fpm-alpine` |
| `GRAV_VERSION` | `2.0.11` | Version de Grav Core + Admin à vendoriser |
| `GRAV_ZIP_URL` | Release GitHub officielle pour `GRAV_VERSION` | Source du zip Grav |
| `GRAV_ZIP_SHA256` | Empreinte publiée par GitHub pour cette release | Vérification d'intégrité |

Le build échoue explicitement (et n'utilise jamais de contenu non vérifié) si le téléchargement
échoue ou si la somme de contrôle ne correspond pas.

## Lancer l'image

Sans volume (tests rapides uniquement — **non adapté à la production**, tout est perdu à la
suppression du conteneur) :

```bash
docker run -d -p 8080:80 grav-runtime:test
```

Avec des volumes persistants séparés par répertoire (modèle recommandé, voir "Répertoires
persistants") :

```bash
docker run -d -p 8080:80 \
  -v pages_data:/var/www/html/user/pages \
  -v accounts_data:/var/www/html/user/accounts \
  -v data_data:/var/www/html/user/data \
  -v images_data:/var/www/html/user/images \
  grav-runtime:test
```

Un harnais de test complet (build + volumes + seed de démonstration) est fourni dans
[`test/compose.yml`](test/compose.yml) — voir "Tests".

## Créer une image applicative fille

`grav-runtime` fournit le socle ; l'image fille ajoute exclusivement le contenu métier : thème,
plugins, pages seed, configuration versionnée.

```dockerfile
FROM ghcr.io/sepp67/grav-runtime:1.0.0

# Thème et plugins métier : livrés par changement d'image, jamais par volume.
COPY grav/user/themes/mon-theme  /var/www/html/user/themes/mon-theme
COPY grav/user/plugins/mon-plugin /var/www/html/user/plugins/mon-plugin

# Configuration non secrète, versionnée comme du code (voir "Configuration non secrète").
COPY grav/user/config /var/www/html/user/config

# Contenu initial : copié dans le volume persistant au premier démarrage seulement,
# jamais écrasé ensuite (voir "Répertoire seed").
COPY grav/user/pages    /opt/grav-seed/pages
COPY grav/user/accounts /opt/grav-seed/accounts
```

Cet exemple ne contient volontairement aucun élément du projet des gîtes ni d'aucun site réel.

## Répertoire seed

`/opt/grav-seed/` est vide dans `grav-runtime`. Une image fille y dépose son contenu initial :
`pages/`, `accounts/`, `data/`, `images/` (mêmes noms que les répertoires persistants réels).

Au démarrage, `docker/seed-init.sh` traite **chaque sous-répertoire indépendamment** :

```text
destination persistante vide     → copie du contenu seed, puis chown www-data
destination déjà peuplée          → aucune copie, aucun écrasement
seed absent (pas fourni par l'image fille) → rien à faire, log explicite
échec de la copie                 → sortie en erreur (le démarrage s'interrompt)
```

Il n'y a pas de fichier `.initialized` global : l'état de chaque répertoire est lu directement
sur le répertoire lui-même. Monter `user/pages` sans monter `user/images` fonctionne
correctement — chacun est initialisé (ou non) indépendamment des autres.

`rsync --delete` n'est jamais utilisé ; rien n'est jamais supprimé automatiquement.

## Répertoires persistants

| Chemin | Nature | Peut être monté séparément |
|---|---|---|
| `/var/www/html/user/pages` | Pages du site | Oui |
| `/var/www/html/user/accounts` | Comptes utilisateurs Grav | Oui |
| `/var/www/html/user/data` | Données des plugins (flex-objects, etc.) | Oui |
| `/var/www/html/user/images` | Médias uploadés | Oui |
| `/var/www/html/user/config` | Config + secrets — voir ci-dessous | Lecture seule pour les secrets |

`/var/www/html/user/themes` et `/var/www/html/user/plugins` **ne sont pas** des répertoires
persistants : ils sont fournis par l'image (runtime + image fille) et changent par changement
d'image, jamais par volume.

## Configuration non secrète (`user/config`)

`user/config` n'est ni "seed", ni volume générique : c'est **du code versionné**, fourni par
l'image applicative via `COPY` (thème actif, alias de la page d'accueil, etc.), au même titre que
le thème ou les plugins. Cela implique une politique GitOps : **la configuration technique ne
doit pas être modifiée depuis Grav Admin en production** — toute modification passe par un commit
et une nouvelle image.

Les véritables secrets (`security-private.php`, `email-private.php`, identifiants SMTP réels...)
ne font partie ni du runtime ni de l'image fille : ils sont montés individuellement, en lecture
seule, au déploiement (fichier monté ou secret Docker/Ansible Vault), jamais générés ni stockés
ici.

Grav n'a pas besoin d'écrire dans `user/config` pour fonctionner normalement (édition de contenu,
authentification, rendu de pages) : seule une modification de configuration depuis l'interface
Admin tenterait d'y écrire, ce qui est précisément ce que la politique GitOps ci-dessus interdit
en production. Un montage en lecture seule de `user/config` est donc compatible avec un
fonctionnement normal du site.

## Variables d'environnement

| Variable | Obligatoire | Défaut | Rôle |
|---|---|---|---|
| `GRAV_ADMIN_USER` | Non* | — | Nom d'utilisateur du compte admin à créer |
| `GRAV_ADMIN_PASSWORD` | Non* | — | Mot de passe (jamais loggé, jamais en argument CLI) |
| `GRAV_ADMIN_EMAIL` | Non* | — | Email du compte admin à créer |
| `GRAV_ADMIN_FULLNAME` | Non | `Administrator` | Nom complet (champ obligatoire côté Grav) |
| `GRAV_ADMIN_TITLE` | Non | défaut Grav | Titre du compte |
| `GRAV_ADMIN_LANGUAGE` | Non | défaut Grav (`en`) | Langue du compte |
| `GRAV_TIMEZONE` | Non | défaut de l'image PHP (UTC) | `date.timezone` PHP |

`*` : `GRAV_ADMIN_USER` / `GRAV_ADMIN_PASSWORD` / `GRAV_ADMIN_EMAIL` doivent être **soit toutes les
trois définies, soit toutes les trois absentes** — voir "Bootstrap administrateur". Aucune valeur
par défaut n'est fournie pour ces trois variables : aucun secret n'est présent dans l'image.

## Bootstrap administrateur

`docker/bootstrap-admin.sh`, exécuté en tant que `www-data` (via `su-exec`), applique une
politique stricte :

```text
aucune des 3 variables définie   → bootstrap désactivé, démarrage normal (exit 0)
les 3 variables définies         → bootstrap exécuté
variables partiellement définies → erreur bloquante, le conteneur ne démarre pas (exit ≠ 0)
```

Un compte déjà existant (`user/accounts/<user>.yaml` présent) n'est **jamais écrasé** : le
bootstrap est ignoré silencieusement (idempotent). Le mot de passe est transmis à la CLI officielle
(`bin/plugin login new-user`) uniquement via stdin, jamais en argument CLI ni en log.

Si le bootstrap est explicitement demandé (les 3 variables sont fournies) mais ne peut pas
aboutir — plugin `login` absent, échec de la CLI — le conteneur s'arrête également en erreur,
plutôt que de démarrer silencieusement sans compte administrateur alors qu'un opérateur en a
demandé un. C'est une extension délibérée de la politique décrite plus haut (qui ne couvrait
explicitement que le cas des variables partielles) ; à signaler si un comportement "continuer sans
compte" est préféré dans ce cas précis.

### Comportement par défaut vs. identifiants de test vs. production

Trois choses distinctes à ne pas confondre :

1. **Comportement sécurisé de `grav-runtime` lui-même** : ni le `Dockerfile`, ni l'entrypoint, ni
   l'image ne définissent la moindre valeur par défaut pour `GRAV_ADMIN_USER` /
   `GRAV_ADMIN_PASSWORD` / `GRAV_ADMIN_EMAIL`. Sans variable fournie, aucun compte n'est créé.
   C'est le seul comportement garanti par l'image ; tout ce qui suit est propre à l'usage qui en
   est fait, pas au runtime lui-même.
2. **`test/compose.yml`** définit volontairement des identifiants en clair
   (`admin` / `ChangeMe123` / `admin@example.com`) pour que le harnais de test fonctionne
   immédiatement, sans étape manuelle. Ces valeurs sont publiques, triviales, et documentées comme
   telles directement dans ce fichier — elles ne doivent **jamais** être réutilisées telles
   quelles, ni comme modèle, en dehors d'un test local jetable.
3. **Environnements réels (staging/production)** : les identifiants admin proviennent
   exclusivement d'un secret injecté au déploiement (Ansible Vault ou équivalent), jamais d'un
   fichier versionné comme `test/compose.yml`. Le futur rôle Ansible sera responsable de cette
   injection ; aucun fichier de ce dépôt ne doit être copié tel quel pour un déploiement réel.

## Healthcheck

`GET /healthz` traverse Nginx → PHP-FPM → un script PHP minimal (`docker/healthz.php`), **sans
passer par le contrôleur frontal de Grav ni dépendre du contenu d'un site**. Il prouve que les
deux processus du conteneur sont vivants et correctement reliés, indépendamment de toute page
métier.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 CMD ["/healthcheck.sh"]
```

**Limite assumée** : `/healthz` ne prouve pas qu'une page Grav réelle se rend correctement (thème,
plugins, contenu). Le futur rôle Ansible devra compléter ce healthcheck technique par une
validation HTTP sur une page réelle du site déployé, avec un chemin configurable (ex. `/` ou une
route dédiée), en plus — pas à la place — du `HEALTHCHECK` Docker natif décrit ici.

## Utilisateurs, permissions et opérations root

| Processus / opération | Utilisateur | Pourquoi |
|---|---|---|
| `entrypoint.sh` (PID 1) | root | Doit pouvoir `chown`/`mkdir` sur des volumes fraîchement montés (root par défaut) et lancer Nginx (bind du port 80, privilégié) |
| Nginx — process master | root (transitoire) | Standard Nginx : bind du port 80 puis fork des workers |
| Nginx — workers | `www-data` (uid/gid 82) | `user www-data;` dans `nginx.conf` — même utilisateur que PHP-FPM, pas de dépendance implicite aux bits "other" |
| PHP-FPM — process master | root (transitoire, standard des images `php-fpm` officielles) | Lancé par l'entrypoint (root), supervise ses workers |
| PHP-FPM — workers | `www-data` | `user`/`group` dans `php-fpm.conf` |
| `bin/plugin login new-user` (bootstrap admin) | `www-data` (via `su-exec`) | Évite de créer des fichiers `root:root` dans `cache/compiled/` — plus besoin d'un second passage de `chown -R` après le bootstrap, contrairement au mécanisme d'origine |
| `chown`/`mkdir` dans `fix_permissions()` | root | Seule opération root répétée à chaque démarrage, limitée aux chemins persistants réels (pages/accounts/data/images/cache/logs/assets), et sautée si la propriété est déjà correcte (idempotent) |

Aucun `chmod 0777` nulle part. Les répertoires persistants sont `u+rwX,g+rwX` (jamais de bit
"other" ajouté).

## Nginx et PHP-FPM

Nginx : sert Grav, transmet uniquement `index.php` et `/healthz` à PHP-FPM (tout autre `.php`
sous le webroot est refusé), refuse les fichiers sensibles (dotfiles, `cache/`, `bin/`, `logs/`,
`backup/`, `tests/`, `user/config`, `user/accounts`, `user/data`, `*.yaml`, `*.yml`, `*.twig`,
`*.md`, `*.sh`, `*.env`), logs vers stdout/stderr. Aucun TLS (géré par le reverse proxy externe).

PHP-FPM : pool `www` avec des valeurs modestes adaptées à un petit site (`pm.max_children = 10`).
Logs déjà routés vers stdout/stderr par `docker.conf`/`zz-docker.conf` fournis par l'image de base
`php:8.3-fpm-alpine` (non dupliqués ici). Pour surcharger ces réglages sans reconstruire l'image :
remplacer le fichier entier, par bind-mount sur `/usr/local/etc/php-fpm.d/www.conf` ou par `COPY`
dans une image dérivée — PHP-FPM ne fusionne pas plusieurs définitions partielles d'un même pool
réparties sur plusieurs fichiers.

## Arrêt et supervision des processus

`entrypoint.sh` reste PID 1 et démarre Nginx et PHP-FPM en arrière-plan : il les surveille tous
les deux, relaie `SIGTERM`/`SIGINT` (et `SIGQUIT`) aux deux, et attend leur arrêt propre. Si l'un
des deux s'arrête de façon inattendue, l'autre est arrêté et le conteneur se termine avec un code
non nul plutôt que de continuer à moitié fonctionnel.

**Point d'attention découvert en test** : l'image de base `php:8.3-fpm-alpine` déclare
`STOPSIGNAL SIGQUIT` (pertinent quand PHP-FPM est lui-même PID 1, ce qui n'est pas notre cas ici).
`grav-runtime` redéclare `STOPSIGNAL SIGTERM` dans son propre Dockerfile pour que `docker stop`
envoie le signal que l'entrypoint sait effectivement traiter.

Deux conteneurs séparés (Nginx / PHP-FPM) n'ont pas été retenus : aucune nécessité technique ne le
justifie, et cela irait à l'encontre de la simplicité opérationnelle recherchée ici.

## Pourquoi `quark2` reste dans le runtime

`quark2` est le thème par défaut livré avec le bundle officiel "Grav Admin" — ce n'est pas un
actif métier. Un thème applicatif peut en hériter au runtime par chaînage de stream (`theme://`),
comme observé dans `gites-theme.yaml` :

```yaml
streams:
  schemes:
    theme:
      paths:
        - user://themes/gites-theme
        - user://themes/quark2
```

Retirer `quark2` casserait ce mécanisme pour toute image fille qui l'utilise, sauf à ce que
chaque image fille re-vendorise elle-même Grav Admin — ce qui annulerait l'intérêt d'un runtime
partagé. Décision : `quark2` reste dans `grav-runtime`.

**Nettoyage effectué** : le zip officiel de Grav Admin embarque aussi son propre contenu de
démonstration (pages "Home"/"Typography", `site.yaml`/`system.yaml`/`media.yaml` d'exemple)
directement sous `user/pages` et `user/config` — exactement les répertoires que ce runtime traite
comme persistants/seedables. Ce contenu est supprimé à la construction de l'image (voir
`Dockerfile`) : seuls `user/themes` et `user/plugins` (le socle technique réel) sont conservés ;
`user/pages`, `user/accounts`, `user/data`, `user/config` démarrent vides.

## Tests

Tous les tests ci-dessous ont été exécutés avec succès pendant l'implémentation. Les identifiants
admin utilisés (`admin` / `ChangeMe123`) sont ceux, publics et jetables, de `test/compose.yml` —
strictement réservés à ces tests locaux, jamais à un environnement réel (voir "Comportement par
défaut vs. identifiants de test vs. production").

**Test 1 — Build**
```bash
docker build -t grav-runtime:test .
```

**Test 2 — Premier démarrage**
```bash
docker compose -f test/compose.yml up -d --build
curl http://localhost:8081/healthz        # attendu : 200
curl -L http://localhost:8081/            # attendu : contenu de test/seed/pages
docker inspect --format='{{.State.Health.Status}}' test-grav-runtime-1   # attendu : healthy
```

**Test 3 — Bootstrap administrateur**
```bash
docker run -d --name grav-admin-test -p 8082:80 \
  -e GRAV_ADMIN_USER=admin -e GRAV_ADMIN_PASSWORD=ChangeMe123 -e GRAV_ADMIN_EMAIL=admin@example.com \
  -v <mêmes volumes que le test précédent> grav-runtime:test
docker logs grav-admin-test | grep bootstrap-admin   # "created successfully"
docker restart grav-admin-test
docker logs grav-admin-test | grep bootstrap-admin   # "already exists — skipping"
```

**Test 3bis — Politique stricte (variables partielles)**
```bash
docker run --rm -e GRAV_ADMIN_USER=admin -e GRAV_ADMIN_PASSWORD=x <volumes> grav-runtime:test
echo $?   # attendu : 1, aucun service démarré
```

**Test 4 — Initialisation des données (seed)**
```bash
# volume user/pages vide + seed monté -> contenu copié au premier démarrage
docker exec test-grav-runtime-1 cat /var/www/html/user/pages/01.home/default.md
# modification + redémarrage -> modification conservée
docker exec test-grav-runtime-1 sh -c "echo modifié >> /var/www/html/user/pages/01.home/default.md"
docker compose -f test/compose.yml restart
docker exec test-grav-runtime-1 cat /var/www/html/user/pages/01.home/default.md   # modification présente
```

**Test 5 — Protection des données** : couvert par le Test 4 (la modification n'est jamais
écrasée par un redémarrage, quel que soit le contenu du seed).

**Test 6 — Permissions**
```bash
docker exec test-grav-runtime-1 stat -c '%a %U:%G %n' /var/www/html/user/pages /var/www/html/user/images
docker exec test-grav-runtime-1 su-exec www-data:www-data sh -c "touch /var/www/html/user/pages/t && rm /var/www/html/user/pages/t"
```

**Test 7 — Arrêt propre**
```bash
docker stop test-grav-runtime-1
docker inspect --format='{{.State.ExitCode}}' test-grav-runtime-1   # attendu : 0
```

**Test 8 — Arrêt inattendu de PHP-FPM (supervision PID 1)**
```bash
PHP_PID=$(docker exec test-grav-runtime-1 sh -c "ps aux | grep 'php-fpm: master' | grep -v grep | awk '{print \$1}'")
docker exec test-grav-runtime-1 kill -9 "$PHP_PID"
sleep 2
docker inspect --format='running={{.State.Running}} exitcode={{.State.ExitCode}}' test-grav-runtime-1
# attendu : running=false exitcode=1, logs montrant l'arrêt forcé de Nginx en réaction
```

## Versionnement

```dockerfile
FROM ghcr.io/sepp67/grav-runtime:1.0.0
```

Une image applicative épingle toujours une version explicite. `ghcr.io/sepp67/grav-runtime:latest`
ne doit jamais être utilisé dans un build destiné à la production.

## Contrat avec le futur rôle Ansible

Sans implémenter Ansible, voici le contrat que ce runtime expose à un futur rôle de déploiement :

- **Image à déployer** : une image applicative fille taguée explicitement, dérivée de
  `ghcr.io/sepp67/grav-runtime:<version>`.
- **Variables d'environnement acceptées** : voir "Variables d'environnement" ci-dessus.
- **Port exposé** : `80/tcp` (HTTP uniquement — le TLS est géré par le reverse proxy externe).
- **Répertoires persistants à monter** : `user/pages`, `user/accounts`, `user/data`,
  `user/images` (séparément ou ensemble).
- **Fichiers secrets montables** : sous `user/config/`, en lecture seule (ex.
  `security-private.php`, `email-private.php`) — jamais générés par l'image.
- **Healthcheck** : `HEALTHCHECK` Docker natif sur `/healthz` (technique, générique) ; le rôle
  Ansible doit compléter par une vérification HTTP sur une page réelle du site, chemin
  configurable.
- **Premier démarrage** : les répertoires persistants vides sont peuplés depuis le seed fourni
  par l'image applicative (`/opt/grav-seed/`), indépendamment par sous-répertoire.
- **Redémarrages suivants** : aucune donnée persistante n'est jamais écrasée ; le bootstrap admin
  ne recrée jamais un compte existant.

## Limitations connues

- **`php:8.3-fpm-alpine` reste un tag mouvant** : précis jusqu'à la version mineure de PHP et la
  variante Alpine, mais pas jusqu'au patch exact ni à la couche de base Alpine — ce tag se déplace
  au fil des reconstructions upstream. Pour un pinning strict, faire pointer `FROM` sur le digest
  exact (`php:8.3-fpm-alpine@sha256:...`) obtenu via `docker inspect` ou le registre.
- **Somme de contrôle Grav** : `getgrav.org` ne publie aucune somme de contrôle pour ses liens de
  téléchargement "latest". La release GitHub versionnée
  (`github.com/getgrav/grav/releases/download/<version>/grav-admin-v<version>.zip`) porte en
  revanche une empreinte SHA-256 authentique publiée par l'API GitHub Releases — c'est celle-ci
  qui est vérifiée par ce Dockerfile (`GRAV_ZIP_SHA256`), plutôt que l'absence de vérification des
  sources d'origine.
- **`/healthz` ne couvre pas le rendu réel d'un site** (thème, plugins, contenu métier) — voir
  "Healthcheck".
- **Un seul conteneur pour Nginx + PHP-FPM** : choix assumé de simplicité opérationnelle, avec une
  supervision minimale mais réelle (voir "Arrêt et supervision des processus").
