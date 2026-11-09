# Incident : promotions inaccessibles et stock indisponible

- **Date** : 9 novembre 2026, réalisation de l’atelier du 8 novembre.
- **Durée observée** : environ 24 minutes, de 19:38 à 20:02 CET (UTC+1).
- **Rédigé par** : Moustapha.

## Impact

Sur le serveur de pré-production, après la publication de novembre :
- La page des promotions et son affiche étaient inaccessibles.
- L’API du stock ne répondait plus à travers Nginx.
- La page d’accueil restait accessible avec une réponse HTTP 200.

Aucun impact sur de vrais clients n’a été établi dans cet exercice.

## Chronologie

Heures en CET (UTC+1), le 9 novembre 2026.

- 19:38 : publication avec `sudo mboa-publier`.
- 19:38 : les tests curl montrent un accueil en 200, les promotions en 403 et le stock en 502.
- 19:47 : le journal Nginx montre `Permission denied` sur promo.html. La commande ls révèle que promo.html et images/novembre.svg appartiennent à root:web, avec les droits 600.
- 19:49–19:50 : passage des deux fichiers en 664 ; les deux URL répondent ensuite 200.
- Vers 19:58 : le journal du service révèle `KeyError: 'STOCK_FICHIER'`. Les notes de version précisent la variable requise et son chemin.
- Avant 20:02 : ajout de STOCK_FICHIER dans l’unité systemd, rechargement de la configuration et relance.
- 20:02 : le service est actif et l’API répond avec la version novembre.
- 20:12 au plus tard : l’essai de publication confirme les fichiers en 664 après correction de l’umask.

Les heures sont celles des preuves disponibles ; certaines étapes sont approximatives.

## Causes racines

### Promotions en 403

Le journal Nginx indiquait une erreur d’ouverture : `Permission denied`.

Les nouveaux fichiers étaient créés en 600 sous root:web. Les workers Nginx, exécutés sous www-data, ne pouvaient donc pas les lire. Nadia ne pouvait pas non plus les modifier via le groupe web.

L’outil de publication imposait `umask 077`. Le setgid du dossier transmettait le groupe web, mais ne donnait aucun droit supplémentaire à ce groupe. L’outil ne vérifiait pas que les permissions obtenues convenaient au serveur web.

### Stock en 502

Le journal Nginx indiquait `Connection refused` vers 127.0.0.1:8090.

Le journal systemd montrait que Python s’arrêtait avec `KeyError: 'STOCK_FICHIER'`. La nouvelle API exigeait cette variable, mais l’unité du service ne la fournissait pas.

La publication avait livré le nouveau code sans adapter la configuration du service à cette nouvelle exigence. Le message de publication réussie ne prouvait pas que l’API fonctionnait après son redémarrage.

## Correction

Pour rétablir la lecture par Nginx et la modification par le groupe web :

```bash
sudo chmod 664 /srv/mboamarket/site/promo.html /srv/mboamarket/site/images/novembre.svg
