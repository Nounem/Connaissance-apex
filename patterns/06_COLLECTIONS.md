# Collections Apex : List, Set, Map

## Rôle des collections

Les collections sont la base de la bulkification. Une IA doit utiliser les collections pour éviter les requêtes répétées, les DML répétitifs et les boucles imbriquées coûteuses.

## List

Utiliser pour conserver un ordre ou préparer un DML groupé.

```apex
List<Account> accountsToUpdate = new List<Account>();
```

## Set

Utiliser pour dédupliquer des valeurs, surtout des `Id`.

```apex
Set<Id> accountIds = new Set<Id>();
```

## Map

Utiliser pour accéder rapidement par clé.

```apex
Map<Id, Account> accountsById = new Map<Id, Account>(accounts);
```

## Pattern de jointure en mémoire

```apex
Map<Id, List<Contact>> contactsByAccountId = new Map<Id, List<Contact>>();
for (Contact c : contacts) {
    if (c.AccountId == null) continue;
    if (!contactsByAccountId.containsKey(c.AccountId)) {
        contactsByAccountId.put(c.AccountId, new List<Contact>());
    }
    contactsByAccountId.get(c.AccountId).add(c);
}
```

## Anti-patterns

- Chercher dans une liste avec une boucle pour chaque élément d’une autre liste.
- Utiliser une List là où un Set éviterait les doublons.
- Refaire une Map déjà disponible.

## Checklist IA

- [ ] Les IDs sont-ils collectés dans un Set ?
- [ ] Les résultats de requête sont-ils convertis en Map ?
- [ ] Les jointures sont-elles faites en mémoire ?
- [ ] Les boucles imbriquées sont-elles évitées ?
