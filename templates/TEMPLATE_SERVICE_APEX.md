# Template service Apex bulkifié

La requête SOQL est déléguée à un Selector (voir [`TEMPLATE_SELECTOR_APEX.md`](TEMPLATE_SELECTOR_APEX.md)) : le Service ne construit jamais lui-même de `SELECT`. Le Selector applique `WITH USER_MODE` pour l'enforcement CRUD/FLS ; le Service n'a donc pas à revérifier les droits d'accès sur les champs lus — cette responsabilité est entièrement portée par le Selector. Le Service reste responsable de la règle métier et de la gestion des échecs DML.

```apex
/**
 * Service applicant la règle de mise à jour du statut des comptes.
 * Ne contient aucune requête SOQL inline : la lecture des données est
 * déléguée à AccountSelector, qui applique l'enforcement FLS/CRUD
 * via WITH USER_MODE.
 */
public inherited sharing class ExampleService {

    /**
     * Traite un seul compte en déléguant à processMany (jamais l'inverse).
     * @param recordId Id du compte à traiter.
     */
    public static void processOne(Id recordId) {
        if (recordId == null) return;
        processMany(new Set<Id>{ recordId });
    }

    /**
     * Traite un lot de comptes : met à jour Statut__c à 'Actif' lorsque
     * AnnualRevenue dépasse 1 000 000, sinon à 'Standard'.
     * @param recordIds Ids des comptes à traiter.
     * @throws ExampleServiceException si des échecs DML critiques surviennent.
     */
    public static void processMany(Set<Id> recordIds) {
        if (recordIds == null || recordIds.isEmpty()) return;

        // Lecture déléguée au Selector : SOQL centralisé, WITH USER_MODE appliqué.
        List<Account> accounts = AccountSelector.selectByIds(recordIds);

        List<Account> accountsToUpdate = new List<Account>();
        for (Account accountRecord : accounts) {
            // Règle métier : le statut dépend du chiffre d'affaires annuel.
            String newStatus = (accountRecord.AnnualRevenue != null
                    && accountRecord.AnnualRevenue >= 1000000)
                ? 'Actif'
                : 'Standard';

            if (accountRecord.Statut__c != newStatus) {
                accountRecord.Statut__c = newStatus;
                accountsToUpdate.add(accountRecord);
            }
        }

        if (accountsToUpdate.isEmpty()) return;

        // allOrNone = false : un échec isolé ne doit pas bloquer tout le lot.
        Database.SaveResult[] saveResults = Database.update(accountsToUpdate, false);

        Boolean hasCriticalFailure = false;
        for (Integer i = 0; i < saveResults.size(); i++) {
            Database.SaveResult result = saveResults[i];
            if (!result.isSuccess()) {
                hasCriticalFailure = true;
                // Journalisation structurée : voir ../patterns/11_DEBUG_DIAGNOSTICS.md
                for (Database.Error error : result.getErrors()) {
                    System.debug(LoggingLevel.ERROR, String.format(
                        'Echec update Account {0} : {1} ({2})',
                        new List<Object>{
                            accountsToUpdate[i].Id,
                            error.getMessage(),
                            error.getStatusCode()
                        }
                    ));
                }
            }
        }

        if (hasCriticalFailure) {
            // Exception custom : voir TEMPLATE_EXCEPTION_CUSTOM.md
            throw new ExampleServiceException(
                'Echec de mise à jour d\'un ou plusieurs comptes dans ExampleService.processMany.'
            );
        }
    }
}
```

## Checklist IA

- [ ] Aucune requête SOQL écrite en dur dans le Service : la lecture passe par une classe Selector dédiée.
- [ ] Le Selector applique bien `WITH USER_MODE` (ou équivalent) ; le Service ne dédouble pas cet enforcement FLS/CRUD.
- [ ] Les erreurs DML sont gérées via `Database.update(..., false)` et le parcours des `Database.SaveResult`, pas via un `update` nu.
- [ ] Les échecs individuels sont journalisés et une exception custom est levée en cas d'échec critique.
- [ ] La classe et les méthodes publiques (`processOne`, `processMany`) portent un commentaire ApexDoc.
- [ ] `processOne` délègue systématiquement à `processMany` (jamais de logique dupliquée).

## Voir aussi

- [`TEMPLATE_SELECTOR_APEX.md`](TEMPLATE_SELECTOR_APEX.md)
- [`TEMPLATE_EXCEPTION_CUSTOM.md`](TEMPLATE_EXCEPTION_CUSTOM.md)
- [`../principes/05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md)
- [`../patterns/15_SECURITE_CRUD_FLS_SHARING.md`](../patterns/15_SECURITE_CRUD_FLS_SHARING.md)
