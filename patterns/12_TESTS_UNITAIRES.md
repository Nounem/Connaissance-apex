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
        Assert.areEqual(200, updated.size(), 'Tous les comptes du bulk doivent être traités');
    }
}
```

## `@TestSetup` : factoriser les données de test

Une méthode `@TestSetup` s'exécute une seule fois avant toutes les méthodes `@IsTest` de la classe. Les données créées sont commitées dans une transaction englobante, puis automatiquement rollback et redisponibles (fraîches) au début de chaque méthode de test — chaque test repart d'un état identique sans recréer les données à chaque fois.

```apex
@IsTest
private class AccountServiceTest {
    @TestSetup
    static void setup() {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Test ' + i));
        }
        insert accounts;
    }

    @IsTest
    static void shouldProcessAccountsInBulk() {
        // Les comptes créés dans setup() sont disponibles ici, requêtés à nouveau
        List<Account> accounts = [SELECT Id FROM Account];

        Test.startTest();
        AccountService.process(new Map<Id, Account>(accounts).keySet());
        Test.stopTest();

        Assert.areEqual(200, [SELECT COUNT() FROM Account WHERE Description != null]);
    }
}
```

À réserver aux données communes à plusieurs méthodes de test. Si une seule méthode a besoin d'un jeu de données spécifique, le créer localement dans cette méthode pour garder le test lisible et autonome.

## Classe `Assert` (moderne) vs `System.assert*`

Préférer la classe `Assert` (`System.Assert` implicite) à `System.assertEquals` / `System.assert` / `System.assertNotEquals` : mêmes vérifications, mais messages d'échec plus riches (nom de méthode, valeurs comparées) et API plus expressive. Les anciennes formes restent fonctionnelles (code legacy) mais ne doivent plus être utilisées dans du code nouveau.

```apex
Assert.areEqual(expected, actual, 'Message explicite en cas d’échec');
Assert.areNotEqual(oldValue, newValue, 'La valeur doit avoir changé');
Assert.isTrue(result.isSuccess(), 'Le traitement doit réussir');
Assert.isFalse(errors.isEmpty(), 'Des erreurs doivent avoir été collectées');
Assert.isNull(account.Description, 'La description doit rester vide');

try {
    AccountService.process(null);
    Assert.fail('Une exception aurait dû être levée pour un set null');
} catch (AccountService.AccountServiceException e) {
    Assert.isTrue(e.getMessage().contains('requis'), 'Message d’erreur attendu');
}
```

## Tester les exceptions difficiles

Utiliser `@TestVisible` pour forcer certains comportements, sans exposer ces contrôles en production.

```apex
@TestVisible
private static Boolean forceDmlException = false;
```

### `Test.isRunningTest()` : anti-pattern sauf exception ciblée

Ne jamais utiliser `Test.isRunningTest()` pour changer le comportement métier normal (ex. `if (Test.isRunningTest()) return fakeValue;`) : le code testé n'est alors plus le code qui tourne en production, ce qui invalide le test.

```apex
// ANTI-PATTERN : le test ne prouve plus rien sur le comportement réel
if (Test.isRunningTest()) {
    return 42;
}
return callExternalCalculation();
```

Seul usage acceptable : simuler une exception techniquement impossible à provoquer autrement dans un test (ex. `DmlException` sur un update qui ne peut pas échouer autrement), en s'appuyant sur un point d'injection dédié plutôt que sur une branche métier.

```apex
// Acceptable : point d'injection dédié, pas une branche métier
if (Test.isRunningTest() && forceDmlException) {
    throw new DmlException('Simulation pour test de robustesse');
}
```

## Mocker un callout HTTP sortant (`HttpCalloutMock`)

Un test ne doit jamais effectuer un vrai appel HTTP sortant. Implémenter `HttpCalloutMock` et l'enregistrer avec `Test.setMock()` avant `Test.startTest()`.

```apex
@IsTest
private class WeatherCalloutMock implements HttpCalloutMock {
    public HTTPResponse respond(HTTPRequest req) {
        Assert.areEqual('GET', req.getMethod());

        HttpResponse res = new HttpResponse();
        res.setHeader('Content-Type', 'application/json');
        res.setBody('{"temperature":21}');
        res.setStatusCode(200);
        return res;
    }
}

@IsTest
private class WeatherServiceTest {
    @IsTest
    static void shouldReturnTemperature() {
        Test.setMock(HttpCalloutMock.class, new WeatherCalloutMock());

        Test.startTest();
        Integer temperature = WeatherService.getCurrentTemperature();
        Test.stopTest();

        Assert.areEqual(21, temperature, 'La température doit venir de la réponse mockée');
    }
}
```

Voir [`17_CALLOUTS_INTEGRATIONS.md`](17_CALLOUTS_INTEGRATIONS.md) pour la conception du client HTTP à tester.

## Stub d'interface (`System.StubProvider`, `Test.createStub()`)

Pour mocker une dépendance non-HTTP (Selector, Service injecté via interface) sans callout, implémenter `System.StubProvider` et générer un stub avec `Test.createStub()`.

```apex
@IsTest
private class AccountSelectorStub implements System.StubProvider {
    public Object handleMethodCall(
        Object stubbedObject,
        String methodName,
        Type returnType,
        List<Type> paramTypes,
        List<String> paramNames,
        List<Object> args
    ) {
        if (methodName == 'selectById') {
            return new List<Account>{ new Account(Id = (Id) args[0], Name = 'Stub') };
        }
        return null;
    }
}

@IsTest
static void shouldUseInjectedSelector() {
    IAccountSelector stub = (IAccountSelector) Test.createStub(
        IAccountSelector.class,
        new AccountSelectorStub()
    );

    AccountService service = new AccountService(stub);

    Test.startTest();
    Account result = service.getAccount(fakeId);
    Test.stopTest();

    Assert.areEqual('Stub', result.Name);
}
```

Nécessite que la dépendance soit exposée via une interface et injectable (constructeur ou setter) dans la classe testée.

## `Test.loadData()` : jeux de données volumineux

Pour des jeux de données réalistes ou volumineux, charger un CSV déposé en Static Resource plutôt que de générer les enregistrements en boucle.

```apex
@TestSetup
static void setup() {
    // Test.loadData lit directement la Static Resource par son nom :
    // pas besoin de la requêter au préalable.
    List<sObject> accounts = Test.loadData(Account.SObjectType, 'ComptesTestCsv');
    Assert.isFalse(accounts.isEmpty(), 'Le CSV doit contenir des comptes de test');
}
```

## Tester triggers et sécurité

- Pour isoler un trigger d'un handler en le désactivant sur une partie du test (ex. tester l'action déclenchée sans relancer toute la chaîne), utiliser le mécanisme de bypass du handler — voir [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md).
- Pour valider les règles de sécurité (FLS, sharing, CRUD) selon le profil de l'utilisateur, exécuter le scénario dans `System.runAs(testUser) { ... }` avec un utilisateur de test dédié à un profil restreint — voir [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md).

```apex
@IsTest
static void shouldRespectSharingRulesForStandardUser() {
    User standardUser = TestDataFactory.createUser('Standard User');

    System.runAs(standardUser) {
        Test.startTest();
        List<Account> visible = [SELECT Id FROM Account WITH USER_MODE];
        Test.stopTest();

        Assert.isTrue(visible.isEmpty(), 'Un utilisateur standard ne doit pas voir ces comptes');
    }
}
```

## Couverture de code : un minimum, pas un objectif

Salesforce impose 75% de couverture globale (et une couverture non nulle par classe/trigger) pour autoriser un déploiement en production. Ce seuil est une contrainte technique de plateforme, pas un indicateur de qualité : une classe couverte à 100% par des tests sans assertion pertinente n'apporte aucune garantie. Toujours viser des assertions qui prouvent le comportement métier ; le pourcentage de couverture n'est qu'une conséquence de tests bien conçus, jamais la cible en soi.

## À éviter

- Tests sans assertion.
- Tests qui ne vérifient que la couverture.
- Tests dépendants de l’ordre des records sans `ORDER BY`.
- `SeeAllData=true` par défaut.
- Tester seulement un enregistrement.
- Callout réel dans un test (toujours `Test.setMock()`).
- `Test.isRunningTest()` utilisé pour modifier une branche métier.
- Viser 75% de couverture comme objectif final plutôt que comme effet de bord de bons tests.

## Checklist IA

- [ ] Les tests créent leurs données (`@TestSetup` ou locales) ?
- [ ] Les tests couvrent bulk ?
- [ ] Les assertions utilisent `Assert.*` avec message explicite ?
- [ ] L’asynchrone est déclenché avec `Test.stopTest()` ?
- [ ] Les erreurs importantes sont couvertes ?
- [ ] Les callouts sortants sont mockés via `HttpCalloutMock` ?
- [ ] Les dépendances non-HTTP injectables sont stubbées via `Test.createStub()` ?
- [ ] `Test.isRunningTest()` n’altère aucune logique métier ?
- [ ] La sécurité multi-profil est testée avec `System.runAs()` si pertinent ?

## Voir aussi

- [`../principes/03_VARIABLES_STATIQUES.md`](../principes/03_VARIABLES_STATIQUES.md)
- [`17_CALLOUTS_INTEGRATIONS.md`](17_CALLOUTS_INTEGRATIONS.md)
- [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md)
- [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md)
- [`../templates/TEMPLATE_TEST_DATA_FACTORY.md`](../templates/TEMPLATE_TEST_DATA_FACTORY.md)
