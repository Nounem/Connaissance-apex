# Template service Apex bulkifié

```apex
public inherited sharing class ExampleService {
    public static void processOne(Id recordId) {
        if (recordId == null) return;
        processMany(new Set<Id>{ recordId });
    }

    public static void processMany(Set<Id> recordIds) {
        if (recordIds == null || recordIds.isEmpty()) return;

        Map<Id, Account> accountsById = new Map<Id, Account>([
            SELECT Id, Name
            FROM Account
            WHERE Id IN :recordIds
        ]);

        List<Account> accountsToUpdate = new List<Account>();
        for (Account accountRecord : accountsById.values()) {
            // Appliquer règle métier ici.
            accountsToUpdate.add(accountRecord);
        }

        if (!accountsToUpdate.isEmpty()) {
            update accountsToUpdate;
        }
    }
}
```
