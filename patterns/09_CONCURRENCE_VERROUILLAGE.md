# Concurrence, verrouillage et data skew

## Problème

Salesforce peut exécuter plusieurs transactions en parallèle. Des erreurs comme `UNABLE_TO_LOCK_ROW` peuvent survenir lorsque plusieurs transactions modifient les mêmes enregistrements ou des parents fortement partagés.

## Règles IA

- Identifier les objets parents fréquemment mis à jour.
- Éviter les mises à jour inutiles sur les mêmes parents.
- Regrouper les DML par objet et par parent.
- Prévoir une stratégie de retry uniquement pour les erreurs transitoires.

## Verrouillage pessimiste

Utiliser `FOR UPDATE` uniquement si nécessaire.

```apex
List<Account> accounts = [
    SELECT Id, Custom_Count__c
    FROM Account
    WHERE Id IN :accountIds
    FOR UPDATE
];
```

## Retry contrôlé

Un retry doit être :

- limité en nombre ;
- journalisé ;
- asynchrone si nécessaire ;
- réservé aux erreurs transitoires.

## Data skew

Surveiller les cas où beaucoup d’enregistrements enfants pointent vers le même parent, ou lorsqu’un propriétaire possède un volume très élevé d’enregistrements. Cela peut amplifier les conflits de verrouillage et les problèmes de performance.

## Checklist IA

- [ ] Le code met-il à jour des parents communs ?
- [ ] Les updates inutiles sont-ils évités ?
- [ ] Le risque `UNABLE_TO_LOCK_ROW` est-il identifié ?
- [ ] Un traitement asynchrone/retry est-il nécessaire ?
- [ ] Les gros volumes par parent/propriétaire sont-ils pris en compte ?
