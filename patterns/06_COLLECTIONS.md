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

## Collections imbriquées à deux niveaux

Utiliser pour des agrégations multi-niveaux (ex : total des lignes de commande par compte, puis par produit). L'IA doit initialiser explicitement chaque niveau avant d'y écrire.

```apex
Map<Id, Map<Id, Decimal>> totalByAccountThenProduct = new Map<Id, Map<Id, Decimal>>();
for (OrderItem oi : orderItems) {
    Id accountId = oi.Order.AccountId;
    Id productId = oi.Product2Id;

    if (!totalByAccountThenProduct.containsKey(accountId)) {
        totalByAccountThenProduct.put(accountId, new Map<Id, Decimal>());
    }
    Map<Id, Decimal> totalByProduct = totalByAccountThenProduct.get(accountId);
    Decimal current = totalByProduct.containsKey(productId) ? totalByProduct.get(productId) : 0;
    totalByProduct.put(productId, current + oi.TotalPrice);
}
```

## Tri de collections : Comparable et Comparator

`List.sort()` trie en place. Pour des objets métier custom (wrapper classes, SObjects avec logique métier), deux approches :

- Implémenter `Comparable` sur la classe elle-même quand il n'existe qu'un seul ordre de tri naturel.
- Implémenter `Comparator<T>` (disponible à partir de l'API 59.0 / Winter '24 — indisponible sur les orgs à une version antérieure) quand plusieurs tris différents sont nécessaires sans modifier la classe triée, ou pour trier des types qu'on ne contrôle pas.

```apex
public class AccountScore implements Comparable {
    public Account acc;
    public Decimal score;

    public AccountScore(Account acc, Decimal score) {
        this.acc = acc;
        this.score = score;
    }

    public Integer compareTo(Object other) {
        AccountScore that = (AccountScore) other;
        return this.score == that.score ? 0 : (this.score > that.score ? 1 : -1);
    }
}

List<AccountScore> scores = new List<AccountScore>();
// ... peupler scores
scores.sort();
```

```apex
public class AccountScoreDescComparator implements Comparator<AccountScore> {
    public Integer compare(AccountScore a, AccountScore b) {
        return a.score == b.score ? 0 : (a.score < b.score ? 1 : -1);
    }
}

scores.sort(new AccountScoreDescComparator());
```

## Pattern wrapper class pour l'UI

Une wrapper class combine des données de plusieurs SObjects (ou des champs calculés) dans une seule structure exposée à un composant LWC/Aura via `@AuraEnabled`. L'IA doit préférer ce pattern plutôt que de renvoyer plusieurs listes distinctes au front-end. Voir [`18_INTEGRATION_LWC_APEX.md`](18_INTEGRATION_LWC_APEX.md) pour le détail des annotations et de la sérialisation.

```apex
public class AccountSummary {
    @AuraEnabled public Account acc;
    @AuraEnabled public Integer openOpportunityCount;
    @AuraEnabled public Decimal totalOrderAmount;
}
```

## Copie de collections : clone() vs copie profonde

`List.clone()` (ou `Set.clone()`, `Map.clone()`) fait une copie superficielle (shallow copy) : la nouvelle collection est indépendante, mais les objets qu'elle contient restent partagés avec l'original. Modifier un SObject issu de la liste clonée modifie donc aussi l'original. Pour une copie indépendante d'objets complexes, l'IA doit recréer manuellement chaque élément (ex : `new Account(sourceAcc)` ou copie champ par champ).

```apex
List<Account> original = new List<Account>{ new Account(Name = 'A') };
List<Account> shallow = original.clone();
shallow[0].Name = 'B';
Assert.areEqual('B', original[0].Name, 'même instance, modifiée aussi'); // shallow copy

List<Account> deepCopy = new List<Account>();
for (Account a : original) {
    deepCopy.add(a.clone(false, true)); // clone() sur SObject sans Id, avec champs
}
```

## Coût mémoire des grosses collections

Chaque élément conservé en mémoire (List, Set, Map) consomme du heap. Charger des dizaines de milliers de SObjects ou des collections imbriquées volumineuses peut dépasser la limite de heap size. L'IA doit privilégier le traitement par lots (Batch Apex, requêtes avec `LIMIT`/pagination) plutôt que de tout charger en mémoire d'un coup. Voir [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md).

## Anti-patterns

- Chercher dans une liste avec une boucle pour chaque élément d’une autre liste.
- Utiliser une List là où un Set éviterait les doublons.
- Refaire une Map déjà disponible.
- Trier une liste manuellement avec des boucles au lieu d'utiliser `List.sort()` avec `Comparable`/`Comparator`.
- Croire que `clone()` copie profondément les objets contenus dans une collection.
- Charger une collection entière en mémoire sans se soucier de la limite de heap size.

## Checklist IA

- [ ] Les IDs sont-ils collectés dans un Set ?
- [ ] Les résultats de requête sont-ils convertis en Map ?
- [ ] Les jointures sont-elles faites en mémoire ?
- [ ] Les boucles imbriquées sont-elles évitées ?
- [ ] Les tris utilisent-ils `Comparable`/`Comparator` plutôt qu'une logique manuelle ?
- [ ] Une copie profonde est-elle faite explicitement si l'original ne doit pas être modifié ?
- [ ] La volumétrie des collections en mémoire a-t-elle été évaluée par rapport à la limite de heap ?

## Voir aussi

- [`../principes/05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md)
- [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md)
- [`16_SOQL_SOSL_AVANCE.md`](16_SOQL_SOSL_AVANCE.md)
- [`18_INTEGRATION_LWC_APEX.md`](18_INTEGRATION_LWC_APEX.md)
