# Tests unitaires Apex

## Objectif

Les tests ne doivent pas seulement viser la couverture. Ils doivent prouver que le code fonctionne dans les cas réels : bulk, absence de données, erreurs, sécurité, asynchrone et limites.

## Règles IA

- Ne pas dépendre des données réelles sauf justification exceptionnelle.
- Créer les données de test nécessaires.
- Utiliser `Test.startTest()` et `Test.stopTest()` pour isoler l’exécution testée et déclencher l’asynchrone.
- Tester 0, 1 et plusieurs enregistrements.
- Tester les erreurs importantes.
- Utiliser des assertions précises.

## Structure recommandée

```apex
@IsTest
private class AccountServiceTest {
    @IsTest
    static void shouldProcessAccountsInBulk() {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Test ' + i));
        }
        insert accounts;

        Test.startTest();
        AccountService.process(new Map<Id, Account>(accounts).keySet());
        Test.stopTest();

        List<Account> updated = [
            SELECT Id, Description
            FROM Account
            WHERE Id IN :accounts
        ];
        System.assertEquals(200, updated.size());
    }
}
```

## Tester les exceptions difficiles

Utiliser `@TestVisible` pour forcer certains comportements, sans exposer ces contrôles en production.

```apex
@TestVisible
private static Boolean forceException = false;
```

## À éviter

- Tests sans assertion.
- Tests qui ne vérifient que la couverture.
- Tests dépendants de l’ordre des records sans `ORDER BY`.
- `SeeAllData=true` par défaut.
- Tester seulement un enregistrement.

## Checklist IA

- [ ] Les tests créent leurs données ?
- [ ] Les tests couvrent bulk ?
- [ ] Les assertions vérifient le résultat métier ?
- [ ] L’asynchrone est déclenché avec `Test.stopTest()` ?
- [ ] Les erreurs importantes sont couvertes ?
