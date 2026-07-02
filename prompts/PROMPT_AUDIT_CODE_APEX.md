# Prompt d’audit de code Apex

Analyse le code Apex fourni comme un reviewer senior Salesforce.

Retourne :

1. Résumé du rôle du code, avec un chiffrage estimé des limites gouverneur consommées par transaction (nombre de requêtes SOQL et d'opérations DML estimées pour le pattern analysé, voir [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md)).
2. Problèmes bloquants.
3. Problèmes de limites gouverneur.
4. Problèmes de bulkification.
5. Problèmes de sécurité CRUD/FLS/sharing.
6. Problèmes de tests.
7. Problèmes de maintenabilité.
8. Problèmes d'architecture : usage à bon escient des patterns Selector/Unit of Work/Domain (signaler aussi bien la sur-ingénierie — couche ajoutée sans besoin réel — que la sous-ingénierie — logique dupliquée ou requête non mockable faute de selector).
9. Compatibilité de version d'API si le code touche ou dépend d'un package géré existant.
10. Version corrigée ou recommandations concrètes.
11. Checklist finale, en te référant à [`../checklists/CHECKLIST_CODE_REVIEW_APEX.md`](../checklists/CHECKLIST_CODE_REVIEW_APEX.md) et [`../checklists/CHECKLIST_SECURITE_APEX.md`](../checklists/CHECKLIST_SECURITE_APEX.md) comme grille de référence.

Pour chaque problème remonté, indique une sévérité (bloquant / majeur / mineur) et la ligne ou le bloc de code concerné.

Sois strict sur : SOQL dans boucle, DML dans boucle, récursion trigger, absence d’assertions, dépendance à des données réelles, logs sensibles et classes sans sharing explicite.
