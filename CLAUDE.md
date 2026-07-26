# Instructions locales — grav-runtime

Lire d’abord le fichier `../CLAUDE.md`.

Ce dépôt fournit un runtime Grav générique.

Priorités :

- compatibilité générique ;
- démarrage idempotent ;
- bootstrap sûr ;
- aucun code applicatif spécifique ;
- aucun secret dans l’image ;
- tests du comportement réel du conteneur.

Toute modification du contrat d’environnement doit être :

- documentée ;
- testée ;
- rétrocompatible lorsque possible ;
- signalée comme impactant potentiellement les images applicatives.

Avant toute modification, consulter :

- `Dockerfile`;
- scripts d’entrée ;
- scripts de bootstrap ;
- configuration Nginx/PHP ;
- tests ;
- README.
