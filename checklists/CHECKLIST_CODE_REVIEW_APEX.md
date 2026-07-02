# Checklist de revue de code Apex pour IA

## Bulkification

- [ ] Aucun SOQL dans une boucle.
- [ ] Aucun DML dans une boucle.
- [ ] Les méthodes principales acceptent des collections.
- [ ] Les IDs sont dédupliqués avec `Set<Id>`.
- [ ] Les résultats SOQL sont organisés en `Map`.

## Limites

- [ ] Nombre de SOQL minimal.
- [ ] Nombre de DML minimal.
- [ ] Pas de boucle imbriquée évitable.
- [ ] Pas de sérialisation ou calcul coûteux répété inutilement.
- [ ] Traitement asynchrone envisagé si volume ou CPU élevé.

## Triggers

- [ ] Trigger sans logique métier lourde.
- [ ] Handler/service utilisé.
- [ ] `before` privilégié pour modifier le même record.
- [ ] Récursion contrôlée avec précision.
- [ ] Interaction Flow/validation considérée.

## Sécurité

- [ ] `with sharing`, `without sharing` ou `inherited sharing` choisi volontairement.
- [ ] CRUD/FLS vérifiés si données manipulées depuis contexte utilisateur.
- [ ] Pas d’exposition inutile en `global` ou `public`.
- [ ] Pas de données sensibles dans les logs.

## Tests

- [ ] Tests avec données créées dans le test.
- [ ] Tests bulk, cas vide, cas nominal, cas erreur.
- [ ] Assertions utiles.
- [ ] Asynchrone testé avec `Test.startTest()` / `Test.stopTest()`.
- [ ] Pas de dépendance fragile à l’ordre des données.

## Maintenabilité

- [ ] Noms explicites.
- [ ] Responsabilités séparées.
- [ ] Configuration externalisée.
- [ ] Hypothèses documentées.
- [ ] Code lisible avant d’être clever.
