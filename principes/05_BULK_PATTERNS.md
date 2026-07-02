# Bulk patterns Apex

## Règle principale

Tout code Apex doit être conçu pour traiter plusieurs enregistrements. Même si l’écran ou le besoin fonctionnel semble unitaire, Salesforce peut exécuter le code en masse via import, API, Flow, Batch, Data Loader ou intégration.

## Pattern standard

1. Collecter les IDs.
2. Faire les requêtes nécessaires en une fois.
3. Construire des Maps.
4. Traiter en mémoire.
5. Faire les DML groupés.

```apex
public class ContactService {
    public static void updateRelatedAccounts(List<Contact> contacts) {
        Set<Id> accountIds = new Set<Id>();
        for (Contact c : contacts) {
            if (c.AccountId != null) accountIds.add(c.AccountId);
        }
        if (accountIds.isEmpty()) return;

        Map<Id, Account> accountsById = new Map<Id, Account>([
            SELECT Id, Description
            FROM Account
            WHERE Id IN :accountIds
        ]);

        List<Account> accountsToUpdate = new List<Account>();
        for (Account a : accountsById.values()) {
            a.Description = 'Compte lié à des contacts mis à jour';
            accountsToUpdate.add(a);
        }

        if (!accountsToUpdate.isEmpty()) update accountsToUpdate;
    }
}
```

## Anti-pattern

```apex
for (Contact c : Trigger.new) {
    Account a = [SELECT Id FROM Account WHERE Id = :c.AccountId];
    update a;
}
```

## Méthodes unitaires et bulk

Préférer :

```apex
public static void applyRules(List<Account> accounts)
```

Éviter :

```apex
public static void applyRule(Account account)
```

sauf si cette méthode délègue immédiatement à une méthode bulk.

## Checklist IA

- [ ] La méthode publique accepte-t-elle une collection ?
- [ ] Les données liées sont-elles préchargées en Map ?
- [ ] Le DML est-il groupé ?
- [ ] Le code fonctionne-t-il avec 0, 1 et 200 enregistrements ?
- [ ] Les doublons d’Id sont-ils évités ?
