# Variables statiques en Apex

## Principe

En Apex, une variable statique n’est pas une variable globale permanente. Elle existe uniquement pendant le contexte d’exécution courant.

## Usages recommandés

### 1. Cache transactionnel

```apex
public class CurrentUserInfo {
    private static User cachedUser;

    public static User getUser() {
        if (cachedUser == null) {
            cachedUser = [
                SELECT Id, ProfileId, TimeZoneSidKey
                FROM User
                WHERE Id = :UserInfo.getUserId()
                LIMIT 1
            ];
        }
        return cachedUser;
    }
}
```

### 2. Guard anti-récursion ciblé

Utiliser une clé métier précise, pas un simple booléen global.

### 3. Contrôle de tests

```apex
@TestVisible
private static Boolean forceFailureForTest = false;
```

Usage en test : le modificateur `@TestVisible` rend un membre `private` (ou `protected`) accessible depuis une méthode `@isTest` déclarée dans une **autre classe** (typiquement une classe de test dans un fichier `.cls` distinct), sans l’exposer publiquement au reste du code applicatif. C’est tout l’intérêt de l’annotation : un membre `private` est déjà visible à l’intérieur de sa propre classe sans elle ; `@TestVisible` sert précisément à franchir la frontière de classe/fichier, mais uniquement pour du code de test.

```apex
public class PaymentProcessor {
    @TestVisible
    private static Boolean forceFailureForTest = false;

    public static void process(Payment__c pmt) {
        if (forceFailureForTest) {
            throw new PaymentException('Échec simulé pour test.');
        }
        // logique métier réelle
    }
}

@isTest
private class PaymentProcessorTest {
    @isTest
    static void testFailurePathIsHandled() {
        PaymentProcessor.forceFailureForTest = true;

        Payment__c pmt = new Payment__c();
        Boolean exceptionThrown = false;
        try {
            PaymentProcessor.process(pmt);
        } catch (PaymentException e) {
            exceptionThrown = true;
        }

        Assert.isTrue(exceptionThrown, 'Le chemin d’échec doit être déclenché.');
    }
}
```

### 4. Constantes (`static final`)

Un `static final` est un usage légitime et distinct des statics mutables : il ne pose pas de risque de fuite d’état entre exécutions puisqu’il est immuable. L’IA doit l’utiliser pour toute valeur métier fixe plutôt que de dupliquer des littéraux dans le code.

```apex
public class OpportunityConstants {
    public static final String STAGE_CLOSED_WON = 'Closed Won';
    public static final Decimal DISCOUNT_THRESHOLD = 0.20;
    public static final Set<String> BLOCKED_STAGES = new Set<String>{
        'Closed Lost', 'Closed Won'
    };
}
```

## Portée : par classe et par transaction

Une variable statique est attachée à sa classe et à la transaction Apex en cours — elle n’est jamais partagée entre deux classes non liées, même exécutées dans la même transaction. Deux classes distinctes ayant chacune leur propre variable statique du même nom conservent des états totalement indépendants. Seules les sous-classes d’une même hiérarchie ou les classes internes partagent l’état statique de leur classe englobante commune.

## `Test.startTest()` / `Test.stopTest()` : ce qui est réinitialisé (et ce qui ne l’est pas)

`Test.startTest()` réinitialise les **limites gouverneur** (nombre de requêtes SOQL, DML, etc.) pour isoler le code testé de la préparation des données. Il ne réinitialise **pas** les variables statiques : une valeur déjà mise en cache avant `startTest()` reste en mémoire après. Les statiques ne sont remis à zéro que si un **nouveau contexte d’exécution** est démarré, ce qui se produit typiquement quand `stopTest()` force l’exécution synchrone de jobs asynchrones enqueués (Queueable, Batch, Future) : ce nouveau contexte reçoit des statics vierges pour sa propre classe.

```apex
public class CurrentUserInfo {
    private static Integer callCount = 0;

    public static Integer incrementAndGet() {
        callCount++;
        return callCount;
    }
}

@isTest
private class CurrentUserInfoTest {
    @isTest
    static void testStaticSurvivesStartTest() {
        CurrentUserInfo.incrementAndGet(); // callCount = 1, avant startTest()

        Test.startTest();
        // Piège : callCount n’est PAS réinitialisé à 0 ici.
        Integer result = CurrentUserInfo.incrementAndGet(); // callCount = 2, pas 1
        Test.stopTest();

        Assert.areEqual(2, result,
            'La variable statique persiste à travers Test.startTest(), ' +
            'contrairement aux limites gouverneur.');
    }
}
```

L’IA ne doit jamais supposer qu’un `Test.startTest()` remet à zéro un cache statique métier : si le test dépend d’un état initial propre, il doit le réinitialiser explicitement ou instancier un nouveau contexte asynchrone.

## Batch Apex : le cache statique ne survit pas entre `execute()`

Chaque scope/chunk d’un `Batchable` (`start` → `execute` → `execute` → ... → `finish`) peut s’exécuter dans un contexte serialisé/désérialisé distinct en fonction du volume et du parallélisme : une variable statique de classe initialisée dans un `execute()` n’est pas garantie disponible dans l’`execute()` suivant. Ne jamais utiliser un static de classe pour accumuler un état entre chunks.

Pour conserver un état entre les appels à `execute()`, utiliser `Database.Stateful` sur la classe Batch — cela concerne les **variables d’instance**, pas les variables statiques :

```apex
public class AccountBatch implements Database.Batchable<SObject>, Database.Stateful {
    // Variable d'instance stateful : conservée entre les execute().
    public Integer totalProcessed = 0;

    // Variable statique : NE PAS compter dessus entre execute().
    private static Integer unreliableStaticCounter = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([SELECT Id FROM Account]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        totalProcessed += scope.size(); // Fiable grâce à Database.Stateful.
        unreliableStaticCounter += scope.size(); // Ne pas s’y fier.
    }

    public void finish(Database.BatchableContext bc) {
        System.debug('Total traité : ' + totalProcessed);
    }
}
```

## Usages interdits

- Stocker des données entre transactions.
- Simuler une session utilisateur.
- Remplacer Custom Metadata, Custom Settings, Platform Cache ou un objet persistant.
- Utiliser un booléen global qui désactive tout un trigger sans distinction.

## Risques

- Une variable statique trop large peut masquer des traitements nécessaires.
- Un cache transactionnel trop volumineux peut consommer le heap.
- Une variable de test mal protégée peut introduire un comportement non souhaité.

## Checklist IA

- [ ] La variable statique est-elle limitée à la transaction ?
- [ ] Le nom indique-t-il clairement son usage ?
- [ ] Le cache réduit-il vraiment SOQL ou CPU ?
- [ ] Le cache ne consomme-t-il pas trop de heap ?
- [ ] Les variables de test sont-elles `@TestVisible` et contrôlées ?
- [ ] Le test ne suppose-t-il pas à tort que `Test.startTest()` réinitialise les statics ?
- [ ] Dans un Batch, l’état entre `execute()` repose-t-il sur `Database.Stateful` (instance) plutôt que sur un static de classe ?

## Voir aussi

- [`02_CONTEXTE_EXECUTION.md`](02_CONTEXTE_EXECUTION.md)
- [`../patterns/08_ASYNCHRONE.md`](../patterns/08_ASYNCHRONE.md)
- [`../patterns/12_TESTS_UNITAIRES.md`](../patterns/12_TESTS_UNITAIRES.md)
