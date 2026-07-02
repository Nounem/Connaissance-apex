# Checklist revue de code Apex

## Bulk
- [ ] Aucun SOQL dans une boucle.
- [ ] Aucun DML dans une boucle.
- [ ] Fonctionne avec 200 enregistrements.
- [ ] Utilise Set/Map pour éviter les doublons.

## Architecture
- [ ] Un seul trigger par objet.
- [ ] Trigger minimal.
- [ ] Logique dans une classe service.
- [ ] Requêtes centralisées si réutilisées.

## Sécurité
- [ ] `with sharing` ou justification.
- [ ] CRUD/FLS vérifiés si nécessaire.
- [ ] Pas de données sensibles dans les logs.

## Tests
- [ ] Données de test créées par le test.
- [ ] Assertions présentes.
- [ ] Cas bulk testé.
- [ ] Cas d’erreur testé.
- [ ] Asynchrone testé avec `Test.stopTest()`.

## Maintenance
- [ ] Noms clairs.
- [ ] Pas de hardcoded IDs.
- [ ] Méthodes courtes.
- [ ] Commentaires utiles.
