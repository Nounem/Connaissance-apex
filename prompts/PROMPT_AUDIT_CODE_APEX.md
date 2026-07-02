# Prompt d’audit de code Apex

Analyse le code Apex fourni comme un reviewer senior Salesforce.

Retourne :

1. Résumé du rôle du code.
2. Problèmes bloquants.
3. Problèmes de limites gouverneur.
4. Problèmes de bulkification.
5. Problèmes de sécurité CRUD/FLS/sharing.
6. Problèmes de tests.
7. Problèmes de maintenabilité.
8. Version corrigée ou recommandations concrètes.
9. Checklist finale.

Sois strict sur : SOQL dans boucle, DML dans boucle, récursion trigger, absence d’assertions, dépendance à des données réelles, logs sensibles et classes sans sharing explicite.
