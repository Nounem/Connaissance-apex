# Template Test Data Factory Apex

Centraliser la création de données de test dans une factory unique : chaque classe de test appelle la même méthode au lieu de dupliquer la construction d'un `Account`, ce qui évite la duplication et facilite la maintenance quand un champ obligatoire est ajouté sur l'objet.

```apex
@isTest
public class TestDataFactory {
    public static List<Account> createAccounts(Integer count) {
        return createAccounts(count, null);
    }

    // overrides : valeurs à fusionner sur chaque enregistrement généré
    // (ex: { 'Industry' => 'Banking' }). Pattern simple, sans dépendance
    // externe, suffisant pour la majorité des cas de test.
    public static List<Account> createAccounts(Integer count, Map<String, Object> overrides) {
        List<Account> accountsToInsert = new List<Account>();

        for (Integer i = 0; i < count; i++) {
            Account accountRecord = new Account(
                Name = 'Test Account ' + i,
                Industry = 'Technology',
                BillingCountry = 'FR'
            );
            applyOverrides(accountRecord, overrides);
            accountsToInsert.add(accountRecord);
        }

        insert accountsToInsert;
        return accountsToInsert;
    }

    private static void applyOverrides(SObject record, Map<String, Object> overrides) {
        if (overrides == null) return;
        for (String fieldName : overrides.keySet()) {
            record.put(fieldName, overrides.get(fieldName));
        }
    }
}
```

## Utilisation dans `@TestSetup`

```apex
@isTest
private class AccountServiceTest {
    @TestSetup
    static void makeData() {
        TestDataFactory.createAccounts(200);
    }

    @isTest
    static void testActivate_bulkAccounts_allActivated() {
        List<Account> accounts = [SELECT Id, OwnerId FROM Account];

        Test.startTest();
        AccountService.activate(accounts);
        Test.stopTest();

        // Assertions...
    }
}
```

Voir [`../patterns/12_TESTS_UNITAIRES.md`](../patterns/12_TESTS_UNITAIRES.md) pour le détail du fonctionnement de `@TestSetup` (données commitées avant chaque méthode de test, isolation entre méthodes).

## Pourquoi centraliser

- Un seul endroit à modifier quand un champ obligatoire est ajouté sur l'objet, au lieu de corriger chaque classe de test une par une.
- Les valeurs par défaut sont cohérentes entre toutes les classes de test, ce qui réduit les faux positifs liés à des données incomplètes.
- La méthode est bulkifiée dès l'origine (`count` enregistrements en un seul `insert`), ce qui habitue les tests à valider le comportement en volume plutôt qu'en enregistrement unique.

## Checklist IA

- [ ] La factory construit une liste bulkifiée et fait un seul `insert`, jamais un `insert` par enregistrement dans une boucle.
- [ ] Les valeurs par défaut sont réalistes et suffisent à passer les validations/champs obligatoires de l'objet.
- [ ] Un mécanisme d'override (`Map<String, Object>` ou méthode dédiée) permet de personnaliser un cas de test sans dupliquer la factory.
- [ ] Les classes de test utilisent `@TestSetup` pour appeler la factory une seule fois par classe.
- [ ] Aucune classe de test ne construit un `Account` "à la main" en dehors de la factory.

## Voir aussi

- [`../patterns/12_TESTS_UNITAIRES.md`](../patterns/12_TESTS_UNITAIRES.md)
- [`../principes/05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md)
