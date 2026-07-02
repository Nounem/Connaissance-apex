# Checklist de revue de code Apex pour IA

## Bulkification

- [ ] Aucun SOQL dans une boucle.
- [ ] Aucun DML dans une boucle.
- [ ] Les méthodes principales acceptent des collections.
- [ ] Les IDs sont dédupliqués avec `Set<Id>`.
- [ ] Les résultats SOQL sont organisés en `Map`.

## Limites

- [ ] Nombre de SOQL minimal (rester loin des limites : 100 en synchrone / 200 en asynchrone).
- [ ] Nombre de DML minimal (150 statements, 10 000 lignes par transaction).
- [ ] Pas de boucle imbriquée évitable.
- [ ] Pas de sérialisation ou calcul coûteux répété inutilement (budget CPU 10s synchrone / 60s asynchrone).
- [ ] Traitement asynchrone envisagé si volume ou CPU élevé.
- [ ] En cas de doute sur la marge restante, code défensif avec `Limits.getQueries()` / `Limits.getDmlStatements()` plutôt qu'un chiffre en dur.
- [ ] Détail complet des limites : voir [`04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md).

## Triggers

- [ ] Un seul trigger par objet (le point d'entrée unique évite les conflits d'ordre d'exécution entre triggers multiples).
- [ ] Trigger sans logique métier lourde.
- [ ] Handler/service utilisé, idéalement via un framework de handler avec bypass (voir [`TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md)).
- [ ] `before` privilégié pour modifier le même record.
- [ ] Récursion contrôlée avec précision (guard statique, pas de simple booléen global).
- [ ] Interaction Flow/validation considérée (ordre d'exécution Salesforce, voir [`07_TRIGGERS.md`](../patterns/07_TRIGGERS.md)).
- [ ] Toutes les branches déclarées dans le trigger (`before`/`after` × `insert`/`update`/`delete`/`undelete`) sont effectivement traitées par le handler — aucune branche silencieusement ignorée.

## Sécurité

- [ ] `with sharing`, `without sharing` ou `inherited sharing` choisi volontairement et justifié en commentaire si `without sharing`.
- [ ] CRUD/FLS vérifiés si données manipulées depuis contexte utilisateur (`WITH USER_MODE`, `Security.stripInaccessible`).
- [ ] Requêtes dynamiques protégées contre l'injection SOQL (bind variables, jamais de concaténation directe d'une entrée utilisateur).
- [ ] Pas d'exposition inutile en `global` ou `public`.
- [ ] Pas de données sensibles dans les logs.
- [ ] Revue complète : voir [`CHECKLIST_SECURITE_APEX.md`](CHECKLIST_SECURITE_APEX.md) et [`15_SECURITE_CRUD_FLS_SHARING.md`](../patterns/15_SECURITE_CRUD_FLS_SHARING.md).

## Tests

- [ ] Tests avec données créées dans le test.
- [ ] Tests bulk, cas vide, cas nominal, cas erreur.
- [ ] Assertions utiles.
- [ ] Asynchrone testé avec `Test.startTest()` / `Test.stopTest()`.
- [ ] Pas de dépendance fragile à l’ordre des données.

## Maintenabilité

- [ ] Noms explicites.
- [ ] Responsabilités séparées.
- [ ] Configuration externalisée (voir [`10_CONFIGURATION.md`](../patterns/10_CONFIGURATION.md)).
- [ ] Pas d'ID hardcodé (Record Type, utilisateur, valeur de picklist) — utiliser Custom Metadata ou requête par nom développeur.
- [ ] Hypothèses documentées.
- [ ] Code lisible avant d’être clever.
- [ ] Méthodes courtes, une responsabilité par méthode.
