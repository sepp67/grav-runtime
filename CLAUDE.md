# Instructions locales — grav-runtime

Avant toute action, lire le fichier `../CLAUDE.md`.

Ce dépôt fournit le runtime Grav générique utilisé comme base par les images
applicatives.

## Invariants locaux

- préserver la compatibilité générique du runtime ;
- conserver un démarrage et un bootstrap idempotents ;
- ne jamais introduire de thème, de contenu ou de comportement propre à une
  application ;
- ne jamais intégrer de secret dans l'image ;
- tester le comportement réel du conteneur, pas seulement la présence des
  fichiers ;
- conserver la séparation entre runtime, application, données persistantes et
  déploiement;
- préserver un contrat générique de persistance documenté et tester toute
  modification des points de montage, de l'initialisation ou des droits
  attendus ;
- la publication éventuelle de `latest` ne doit jamais être présentée comme
  une version auditée ou une référence de déploiement immuable ;

Toute modification du contrat d'environnement doit être documentée, testée et
signalée comme potentiellement incompatible avec les images applicatives. La
rétrocompatibilité doit être préservée lorsque cela est raisonnablement
possible.

## Avant une modification

Consulter au minimum :

- `Dockerfile` ;
- `docker/entrypoint.sh` ;
- `docker/bootstrap-admin.sh` ;
- `docker/nginx.conf` ;
- `docker/php-fpm.conf` ;
- `test/` ;
- `README.md`.

## Contrôles spécifiques

- construire l'image de manière reproductible ;
- vérifier les versions effectives de Grav, PHP et des dépendances
  structurantes ;
- vérifier le démarrage réel du conteneur ;
- vérifier le healthcheck ;
- vérifier le bootstrap sur volume vide ;
- vérifier le redémarrage sur volume déjà initialisé ;
- signaler les tests non exécutés et les impacts possibles sur les images
  applicatives.
