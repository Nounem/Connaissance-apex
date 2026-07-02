# Limites gouverneur Apex

## Principe

Les limites gouverneur sont une contrainte d’architecture. Elles obligent à optimiser l’accès aux données, les DML, le CPU, le heap et les traitements asynchrones.

## Règles IA

- Concevoir pour consommer moins que les limites, pas juste pour rester sous le maximum.
- Supposer que d’autres traitements partagent la même transaction.
- Éviter les opérations inutiles, même si elles semblent petites.
- Déplacer les traitements lourds vers Queueable, Batch ou Platform Event si nécessaire.

## SOQL

Bonnes pratiques :

- Une requête par objet/contexte métier lorsque possible.
- Utiliser `Set<Id>` pour filtrer.
- Charger uniquement les champs nécessaires.
- Regrouper les requêtes liées.
- Penser à la sélectivité pour les grands volumes.

```apex
Set<Id> accountIds = new Set<Id>();
for (Contact c : contacts) {
    if (c.AccountId != null) accountIds.add(c.AccountId);
}

Map<Id, Account> accountsById = new Map<Id, Account>([
    SELECT Id, Name, OwnerId
    FROM Account
    WHERE Id IN :accountIds
]);
```

## DML

- Construire une liste `recordsToUpdate`.
- Ne faire le DML que si la liste n’est pas vide.
- Utiliser `Database.update(records, false)` si une réussite partielle est acceptable.

```apex
if (!accountsToUpdate.isEmpty()) {
    Database.SaveResult[] results = Database.update(accountsToUpdate, false);
}
```

## CPU

Réduire :

- boucles imbriquées ;
- accès dynamiques inutiles ;
- sérialisations JSON répétées ;
- recalculs évitables ;
- traitements synchrones lourds.

## Heap

- Ne pas cacher de gros volumes dans des variables statiques.
- Traiter en batch si le volume est élevé.
- Ne pas charger des champs inutiles.

## Checklist IA

- [ ] SOQL hors boucle ?
- [ ] DML hors boucle ?
- [ ] Requêtes sélectives ?
- [ ] Champs limités au nécessaire ?
- [ ] Pas de boucle O(n²) évitable ?
- [ ] Traitement asynchrone envisagé si CPU/heap élevé ?
