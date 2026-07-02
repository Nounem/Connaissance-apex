# Guide complet Apex Salesforce pour débutant

**Objectif du document :** aider un développeur débutant à écrire du code Apex propre, bulkifié, testable, maintenable et compatible avec la plateforme Salesforce.

Ce guide est conçu pour être utilisé à la fois par un humain et par une IA de développement. Il peut servir de document de référence dans ChatGPT, Cursor, GitHub Copilot, Claude, Windsurf ou une base RAG interne.

> Ce document est une synthèse originale en français inspirée des principes du livre *Advanced Apex Programming in Salesforce* de Dan Appleman et des bonnes pratiques officielles Salesforce. Il ne remplace pas la documentation officielle Salesforce, notamment pour les valeurs exactes des limites gouverneur, qui peuvent évoluer.

---

## Table des matières

1. [Comment utiliser ce guide](#1-comment-utiliser-ce-guide)
2. [Mentalité Apex : penser plateforme avant syntaxe](#2-mentalité-apex--penser-plateforme-avant-syntaxe)
3. [Règles d’or à respecter absolument](#3-règles-dor-à-respecter-absolument)
4. [Structure recommandée d’un projet Apex](#4-structure-recommandée-dun-projet-apex)
5. [Bases du langage Apex](#5-bases-du-langage-apex)
6. [Objets, SObjects et DML](#6-objets-sobjects-et-dml)
7. [SOQL et SOSL](#7-soql-et-sosl)
8. [Collections : List, Set, Map](#8-collections--list-set-map)
9. [Governor limits](#9-governor-limits)
10. [Bulkification](#10-bulkification)
11. [Triggers](#11-triggers)
12. [Trigger Handler Pattern](#12-trigger-handler-pattern)
13. [Service Layer](#13-service-layer)
14. [Selector / Repository Layer](#14-selector--repository-layer)
15. [Variables statiques et contexte d’exécution](#15-variables-statiques-et-contexte-dexécution)
16. [Sécurité : sharing, CRUD, FLS](#16-sécurité--sharing-crud-fls)
17. [Gestion des erreurs](#17-gestion-des-erreurs)
18. [Asynchrone : Future, Queueable, Batch, Scheduled](#18-asynchrone--future-queueable-batch-scheduled)
19. [Callouts HTTP](#19-callouts-http)
20. [Tests unitaires](#20-tests-unitaires)
21. [Factory de données de test](#21-factory-de-données-de-test)
22. [Debugging et diagnostics](#22-debugging-et-diagnostics)
23. [Qualité de code et conventions](#23-qualité-de-code-et-conventions)
24. [Checklist avant commit](#24-checklist-avant-commit)
25. [Prompts IA prêts à l’emploi](#25-prompts-ia-prêts-à-lemploi)
26. [Templates Apex réutilisables](#26-templates-apex-réutilisables)
27. [Anti-patterns à éviter](#27-anti-patterns-à-éviter)
28. [Plan d’apprentissage débutant](#28-plan-dapprentissage-débutant)

---

# 1. Comment utiliser ce guide

Ce document doit être utilisé comme une **référence pratique**. Avant de coder, lis les règles d’or. Avant de créer un trigger, lis les sections sur les triggers, la bulkification et les governor limits. Avant de livrer, utilise les checklists.

Pour une IA, donne cette instruction :

```text
Tu es un expert Apex Salesforce. Avant de générer ou modifier du code Apex, applique toutes les règles du guide : bulkification, limites gouverneur, séparation des responsabilités, tests unitaires, sécurité CRUD/FLS/sharing, gestion des erreurs et maintenabilité. Si une règle n’est pas applicable, explique pourquoi.
```

---

# 2. Mentalité Apex : penser plateforme avant syntaxe

Apex ressemble à Java ou C#, mais ce n’est pas un langage classique exécuté sur un serveur que tu contrôles entièrement. Apex tourne dans la plateforme Salesforce, dans une organisation qui peut contenir :

- des flows ;
- des validations rules ;
- des processus déclaratifs ;
- des triggers existants ;
- des packages installés ;
- des intégrations externes ;
- des règles de sécurité ;
- des limites gouverneur.

Le débutant fait souvent cette erreur : il apprend la syntaxe, puis code comme s’il était seul dans l’application. En Apex, il faut d’abord penser :

- transaction ;
- contexte d’exécution ;
- bulk ;
- limites ;
- sécurité ;
- effets de bord ;
- tests.

## Principe fondamental

> En Apex, un code correct n’est pas seulement un code qui fonctionne avec un enregistrement. C’est un code qui fonctionne avec 1, 10, 200 ou plusieurs milliers d’enregistrements, sans dépasser les limites Salesforce, tout en restant lisible et testable.

---

# 3. Règles d’or à respecter absolument

## Règles obligatoires

1. **Ne jamais faire de SOQL dans une boucle.**
2. **Ne jamais faire de DML dans une boucle.**
3. **Toujours écrire les triggers en bulk.**
4. **Un seul trigger par objet.**
5. **Pas de logique métier dans le trigger.**
6. **Utiliser un handler pour orchestrer.**
7. **Utiliser une classe service pour la logique métier.**
8. **Utiliser des collections : `List`, `Set`, `Map`.**
9. **Tester les cas bulk, pas seulement un seul enregistrement.**
10. **Ne jamais dépendre de données existantes dans l’org pour les tests.**
11. **Ne jamais hardcoder les IDs Salesforce.**
12. **Toujours gérer les erreurs importantes.**
13. **Toujours vérifier les impacts sécurité : sharing, CRUD, FLS.**
14. **Préférer `before trigger` quand on modifie le même enregistrement.**
15. **Écrire du code lisible avant d’écrire du code “intelligent”.**

## Règles pour l’IA

Quand une IA génère du code Apex, elle doit automatiquement vérifier :

- Est-ce bulkifié ?
- Est-ce testable ?
- Les SOQL/DML sont-ils hors des boucles ?
- Y a-t-il des tests pour insert/update/delete selon le besoin ?
- Le trigger délègue-t-il la logique ?
- Les exceptions sont-elles traitées proprement ?
- La sécurité est-elle prise en compte ?
- Le code est-il compréhensible par un développeur junior ?

---

# 4. Structure recommandée d’un projet Apex

Une structure simple et propre :

```text
force-app/main/default/classes/
├── account/
│   ├── AccountTriggerHandler.cls
│   ├── AccountService.cls
│   ├── AccountSelector.cls
│   └── AccountTestDataFactory.cls
├── contact/
│   ├── ContactTriggerHandler.cls
│   ├── ContactService.cls
│   └── ContactSelector.cls
├── common/
│   ├── ApplicationException.cls
│   ├── SecurityUtil.cls
│   ├── DmlUtil.cls
│   └── LogService.cls
└── tests/
    ├── AccountServiceTest.cls
    └── ContactServiceTest.cls

force-app/main/default/triggers/
├── AccountTrigger.trigger
└── ContactTrigger.trigger
```

## Rôle de chaque couche

| Couche | Rôle |
|---|---|
| Trigger | Point d’entrée uniquement |
| Handler | Détecte l’événement et appelle les services |
| Service | Contient la logique métier |
| Selector | Centralise les requêtes SOQL |
| Util | Fonctions techniques communes |
| TestDataFactory | Crée les données de test |

---

# 5. Bases du langage Apex

Apex est orienté objet. Les classes, méthodes, interfaces, listes et maps sont omniprésentes.

## Exemple de classe simple

```apex
public with sharing class AccountService {
    public static void normalizeNames(List<Account> accounts) {
        for (Account acc : accounts) {
            if (String.isNotBlank(acc.Name)) {
                acc.Name = acc.Name.trim();
            }
        }
    }
}
```

## Types courants

```apex
String name = 'Acme';
Integer count = 10;
Decimal amount = 125.50;
Boolean isActive = true;
Date todayDate = Date.today();
Datetime nowValue = Datetime.now();
Id recordId;
```

## Bonnes pratiques de nommage

- Classe : `AccountService`, `OpportunitySelector`
- Méthode : `calculateScore`, `validateAccounts`
- Variable : `accountsToUpdate`, `contactIds`
- Constante : `STATUS_ACTIVE`, `DEFAULT_LIMIT`

Évite :

```apex
String x;
List<Account> l;
Map<Id, Account> m;
```

Préférer :

```apex
String customerName;
List<Account> accountsToProcess;
Map<Id, Account> accountsById;
```

---

# 6. Objets, SObjects et DML

Les données Salesforce sont représentées par des `SObject` : `Account`, `Contact`, `Opportunity`, objets custom, etc.

## Créer un enregistrement

```apex
Account acc = new Account(Name = 'Acme');
insert acc;
```

## Mettre à jour un enregistrement

```apex
acc.Name = 'Acme France';
update acc;
```

## Supprimer un enregistrement

```apex
delete acc;
```

## DML en bulk

Mauvais :

```apex
for (Account acc : accounts) {
    update acc;
}
```

Bon :

```apex
List<Account> accountsToUpdate = new List<Account>();

for (Account acc : accounts) {
    acc.Description = 'Updated';
    accountsToUpdate.add(acc);
}

if (!accountsToUpdate.isEmpty()) {
    update accountsToUpdate;
}
```

## DML partiel avec `Database.update`

Quand tu veux continuer même si certains enregistrements échouent :

```apex
Database.SaveResult[] results = Database.update(accountsToUpdate, false);

for (Database.SaveResult result : results) {
    if (!result.isSuccess()) {
        for (Database.Error err : result.getErrors()) {
            System.debug(LoggingLevel.ERROR, err.getMessage());
        }
    }
}
```

Utilise cette approche pour les traitements robustes, les batchs et les intégrations.

---

# 7. SOQL et SOSL

SOQL sert à interroger les objets Salesforce.

## Requête simple

```apex
List<Account> accounts = [
    SELECT Id, Name, Industry
    FROM Account
    WHERE Industry = 'Technology'
];
```

## Requête avec `Set<Id>`

```apex
Set<Id> accountIds = new Set<Id>();

for (Contact c : contacts) {
    if (c.AccountId != null) {
        accountIds.add(c.AccountId);
    }
}

Map<Id, Account> accountsById = new Map<Id, Account>([
    SELECT Id, Name, OwnerId
    FROM Account
    WHERE Id IN :accountIds
]);
```

## Requête parent

```apex
List<Contact> contacts = [
    SELECT Id, FirstName, LastName, Account.Id, Account.Name
    FROM Contact
    WHERE AccountId != null
];
```

## Sous-requête enfant

```apex
List<Account> accounts = [
    SELECT Id, Name,
        (SELECT Id, FirstName, LastName FROM Contacts)
    FROM Account
    WHERE Id IN :accountIds
];
```

## Règles SOQL

- Sélectionne seulement les champs nécessaires.
- Utilise des filtres sélectifs.
- Évite les requêtes non bornées.
- Utilise `LIMIT` quand c’est logique.
- Centralise les requêtes dans des selectors.
- Ne mets jamais une requête dans une boucle.

---

# 8. Collections : List, Set, Map

Les collections sont indispensables en Apex.

## List

Une liste garde l’ordre et peut contenir des doublons.

```apex
List<Account> accounts = new List<Account>();
accounts.add(new Account(Name = 'Acme'));
```

## Set

Un set ne contient pas de doublons. Très utile pour collecter des IDs.

```apex
Set<Id> accountIds = new Set<Id>();
for (Contact c : contacts) {
    accountIds.add(c.AccountId);
}
```

## Map

Une map associe une clé à une valeur. C’est la collection la plus importante pour le bulk.

```apex
Map<Id, Account> accountsById = new Map<Id, Account>(accounts);
Account acc = accountsById.get(contact.AccountId);
```

## Pattern classique

```apex
Set<Id> accountIds = new Set<Id>();

for (Contact c : Trigger.new) {
    if (c.AccountId != null) {
        accountIds.add(c.AccountId);
    }
}

Map<Id, Account> accountsById = new Map<Id, Account>([
    SELECT Id, Name
    FROM Account
    WHERE Id IN :accountIds
]);

for (Contact c : Trigger.new) {
    Account parentAccount = accountsById.get(c.AccountId);
    if (parentAccount != null) {
        // Logique métier
    }
}
```

---

# 9. Governor limits

Salesforce impose des limites pour protéger la plateforme multi-tenant. Ces limites concernent notamment :

- nombre de requêtes SOQL ;
- nombre d’opérations DML ;
- CPU time ;
- heap size ;
- callouts ;
- jobs asynchrones ;
- records traités ;
- profondeur de pile ;
- taille des logs.

## Mentalité à adopter

Ne code pas en pensant : “je suis sous la limite”. Code en pensant : “je minimise ma consommation”.

## Mauvais réflexes

```apex
for (Account acc : Trigger.new) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
    update acc;
}
```

Ce code consomme une requête et une opération DML par compte.

## Bon réflexe

```apex
Set<Id> accountIds = new Map<Id, Account>(Trigger.new).keySet();

List<Contact> contacts = [
    SELECT Id, AccountId
    FROM Contact
    WHERE AccountId IN :accountIds
];

List<Account> accountsToUpdate = new List<Account>();
for (Account acc : Trigger.new) {
    acc.Description = 'Processed';
    accountsToUpdate.add(acc);
}

if (!accountsToUpdate.isEmpty()) {
    update accountsToUpdate;
}
```

## Outils utiles

```apex
Integer queriesUsed = Limits.getQueries();
Integer queryLimit = Limits.getLimitQueries();
Integer dmlUsed = Limits.getDmlStatements();
Integer cpuUsed = Limits.getCpuTime();
```

Ne mets pas ces mesures partout en production, mais elles sont utiles pour diagnostiquer.

---

# 10. Bulkification

La bulkification consiste à écrire du code qui traite plusieurs enregistrements à la fois.

## Pourquoi ?

Un trigger peut être appelé avec :

- 1 enregistrement depuis l’interface ;
- 200 enregistrements via Data Loader ;
- des lots dans une intégration ;
- des mises à jour en masse ;
- des automatisations internes.

## Règle principale

> Tout code Apex doit fonctionner correctement avec une liste d’enregistrements.

## Pattern bulk standard

1. Lire tous les enregistrements entrants.
2. Collecter les IDs nécessaires dans des `Set<Id>`.
3. Faire les requêtes SOQL hors boucle.
4. Stocker les résultats dans des `Map<Id, SObject>`.
5. Traiter les données en mémoire.
6. Ajouter les modifications dans des listes.
7. Faire les DML hors boucle.

## Exemple complet

Objectif : quand un contact est créé, mettre à jour le champ `Last_Contact_Created_Date__c` du compte parent.

```apex
public with sharing class ContactService {
    public static void updateAccountLastContactDate(List<Contact> newContacts) {
        Set<Id> accountIds = new Set<Id>();

        for (Contact c : newContacts) {
            if (c.AccountId != null) {
                accountIds.add(c.AccountId);
            }
        }

        if (accountIds.isEmpty()) {
            return;
        }

        List<Account> accountsToUpdate = new List<Account>();

        for (Id accountId : accountIds) {
            accountsToUpdate.add(new Account(
                Id = accountId,
                Last_Contact_Created_Date__c = Date.today()
            ));
        }

        update accountsToUpdate;
    }
}
```

## Pourquoi cet exemple est correct ?

- Pas de SOQL dans une boucle.
- Pas de DML dans une boucle.
- Utilise un `Set<Id>` pour éviter les doublons.
- Ne met à jour chaque compte qu’une seule fois.
- Fonctionne avec 1 ou 200 contacts.

---

# 11. Triggers

Un trigger permet d’exécuter du code avant ou après une opération sur un objet.

## Événements disponibles

- `before insert`
- `before update`
- `before delete`
- `after insert`
- `after update`
- `after delete`
- `after undelete`

## Quand utiliser before ?

Utilise `before` quand tu veux modifier les enregistrements du trigger eux-mêmes.

```apex
trigger AccountTrigger on Account (before insert, before update) {
    for (Account acc : Trigger.new) {
        if (String.isNotBlank(acc.Name)) {
            acc.Name = acc.Name.trim();
        }
    }
}
```

Avantage : pas besoin de faire `update Trigger.new`.

## Quand utiliser after ?

Utilise `after` quand tu as besoin :

- de l’Id généré ;
- de créer ou modifier des enregistrements liés ;
- d’appeler une logique dépendant de l’enregistrement déjà sauvegardé.

## Ce qu’il ne faut pas faire

```apex
trigger AccountTrigger on Account (after update) {
    for (Account acc : Trigger.new) {
        Contact c = [SELECT Id FROM Contact WHERE AccountId = :acc.Id LIMIT 1];
        update c;
    }
}
```

Problèmes :

- SOQL dans une boucle ;
- DML dans une boucle ;
- pas de handler ;
- pas de gestion du bulk ;
- risque de limite gouverneur.

---

# 12. Trigger Handler Pattern

Le trigger doit être très court. Il délègue au handler.

## Trigger

```apex
trigger AccountTrigger on Account (
    before insert,
    before update,
    after insert,
    after update
) {
    AccountTriggerHandler handler = new AccountTriggerHandler();

    if (Trigger.isBefore && Trigger.isInsert) {
        handler.beforeInsert(Trigger.new);
    }

    if (Trigger.isBefore && Trigger.isUpdate) {
        handler.beforeUpdate(Trigger.new, Trigger.oldMap);
    }

    if (Trigger.isAfter && Trigger.isInsert) {
        handler.afterInsert(Trigger.new);
    }

    if (Trigger.isAfter && Trigger.isUpdate) {
        handler.afterUpdate(Trigger.new, Trigger.oldMap);
    }
}
```

## Handler

```apex
public with sharing class AccountTriggerHandler {
    public void beforeInsert(List<Account> newAccounts) {
        AccountService.normalizeNames(newAccounts);
    }

    public void beforeUpdate(List<Account> newAccounts, Map<Id, Account> oldAccountsById) {
        AccountService.normalizeNames(newAccounts);
    }

    public void afterInsert(List<Account> newAccounts) {
        AccountService.createDefaultTasks(newAccounts);
    }

    public void afterUpdate(List<Account> newAccounts, Map<Id, Account> oldAccountsById) {
        AccountService.handleImportantChanges(newAccounts, oldAccountsById);
    }
}
```

## Service

```apex
public with sharing class AccountService {
    public static void normalizeNames(List<Account> accounts) {
        for (Account acc : accounts) {
            if (String.isNotBlank(acc.Name)) {
                acc.Name = acc.Name.trim();
            }
        }
    }

    public static void createDefaultTasks(List<Account> accounts) {
        List<Task> tasksToInsert = new List<Task>();

        for (Account acc : accounts) {
            tasksToInsert.add(new Task(
                WhatId = acc.Id,
                Subject = 'Suivi nouveau compte',
                Status = 'Not Started',
                Priority = 'Normal'
            ));
        }

        if (!tasksToInsert.isEmpty()) {
            insert tasksToInsert;
        }
    }

    public static void handleImportantChanges(
        List<Account> newAccounts,
        Map<Id, Account> oldAccountsById
    ) {
        for (Account acc : newAccounts) {
            Account oldAcc = oldAccountsById.get(acc.Id);
            if (oldAcc != null && acc.Industry != oldAcc.Industry) {
                // Logique métier ici
            }
        }
    }
}
```

---

# 13. Service Layer

La service layer contient la logique métier. Elle doit être :

- indépendante du trigger ;
- testable ;
- bulkifiée ;
- lisible ;
- réutilisable depuis un trigger, un batch, un contrôleur LWC ou un flow invocable.

## Mauvais exemple

```apex
trigger OpportunityTrigger on Opportunity (after update) {
    for (Opportunity opp : Trigger.new) {
        if (opp.StageName == 'Closed Won') {
            insert new Task(WhatId = opp.Id, Subject = 'Créer onboarding');
        }
    }
}
```

## Bon exemple

```apex
public with sharing class OpportunityService {
    public static void createOnboardingTasks(
        List<Opportunity> newOpportunities,
        Map<Id, Opportunity> oldOpportunitiesById
    ) {
        List<Task> tasksToInsert = new List<Task>();

        for (Opportunity opp : newOpportunities) {
            Opportunity oldOpp = oldOpportunitiesById.get(opp.Id);
            Boolean becameClosedWon = opp.StageName == 'Closed Won'
                && oldOpp != null
                && oldOpp.StageName != 'Closed Won';

            if (becameClosedWon) {
                tasksToInsert.add(new Task(
                    WhatId = opp.Id,
                    Subject = 'Créer onboarding client',
                    Status = 'Not Started',
                    Priority = 'High'
                ));
            }
        }

        if (!tasksToInsert.isEmpty()) {
            insert tasksToInsert;
        }
    }
}
```

---

# 14. Selector / Repository Layer

Un selector centralise les requêtes SOQL.

## Pourquoi ?

- Évite la duplication des requêtes.
- Facilite les changements de champs.
- Rend les services plus lisibles.
- Aide à contrôler les performances.

## Exemple

```apex
public with sharing class AccountSelector {
    public static Map<Id, Account> selectByIds(Set<Id> accountIds) {
        if (accountIds == null || accountIds.isEmpty()) {
            return new Map<Id, Account>();
        }

        return new Map<Id, Account>([
            SELECT Id, Name, OwnerId, Industry
            FROM Account
            WHERE Id IN :accountIds
        ]);
    }
}
```

## Utilisation

```apex
Map<Id, Account> accountsById = AccountSelector.selectByIds(accountIds);
```

---

# 15. Variables statiques et contexte d’exécution

Les variables statiques en Apex vivent uniquement pendant le contexte d’exécution courant. Elles sont très utiles pour :

- éviter une récursion non voulue ;
- cacher des données pendant la transaction ;
- éviter d’exécuter plusieurs fois la même opération ;
- contrôler certains comportements dans les tests.

## Important

Une variable statique ne doit pas être utilisée pour stocker une information permanente. Elle disparaît à la fin de la transaction.

## Guard anti-récursion simple

```apex
public with sharing class TriggerExecutionGuard {
    private static Set<String> executedKeys = new Set<String>();

    public static Boolean hasAlreadyRun(String key) {
        return executedKeys.contains(key);
    }

    public static void markAsRun(String key) {
        executedKeys.add(key);
    }
}
```

Utilisation :

```apex
if (TriggerExecutionGuard.hasAlreadyRun('Account_afterUpdate')) {
    return;
}
TriggerExecutionGuard.markAsRun('Account_afterUpdate');
```

## Attention

Un guard trop simple peut bloquer un traitement légitime dans une transaction complexe. Pour les traitements avancés, il vaut mieux suivre les IDs déjà traités :

```apex
public with sharing class AccountTriggerState {
    private static Set<Id> processedAccountIds = new Set<Id>();

    public static Boolean markIfNotProcessed(Id accountId) {
        if (processedAccountIds.contains(accountId)) {
            return false;
        }
        processedAccountIds.add(accountId);
        return true;
    }
}
```

---

# 16. Sécurité : sharing, CRUD, FLS

La sécurité est souvent oubliée par les débutants.

## Sharing

```apex
public with sharing class AccountService {
    // Respecte les règles de partage de l’utilisateur courant
}
```

Utilise généralement `with sharing` pour les services exposés à l’utilisateur.

## CRUD et FLS

CRUD signifie : l’utilisateur a-t-il le droit de créer, lire, modifier ou supprimer un objet ?

FLS signifie : l’utilisateur a-t-il accès à un champ ?

## Exemple de vérification simple

```apex
if (!Schema.sObjectType.Account.isUpdateable()) {
    throw new SecurityException('Vous n’avez pas le droit de modifier Account.');
}

if (!Schema.sObjectType.Account.fields.Name.isUpdateable()) {
    throw new SecurityException('Vous n’avez pas le droit de modifier Account.Name.');
}
```

## `WITH SECURITY_ENFORCED`

```apex
List<Account> accounts = [
    SELECT Id, Name
    FROM Account
    WITH SECURITY_ENFORCED
];
```

## `Security.stripInaccessible`

```apex
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.UPDATABLE,
    accountsToUpdate
);

List<SObject> safeRecords = decision.getRecords();
update safeRecords;
```

## Règle IA

Quand une IA génère du code Apex exposé à un utilisateur, elle doit préciser comment la sécurité est gérée.

---

# 17. Gestion des erreurs

## Ne pas avaler les exceptions silencieusement

Mauvais :

```apex
try {
    update accounts;
} catch (Exception e) {
}
```

Bon :

```apex
try {
    update accounts;
} catch (DmlException e) {
    System.debug(LoggingLevel.ERROR, 'Erreur DML: ' + e.getMessage());
    throw e;
}
```

## Exception custom

```apex
public class ApplicationException extends Exception {}
```

Utilisation :

```apex
if (accounts.isEmpty()) {
    throw new ApplicationException('Aucun compte à traiter.');
}
```

## Erreurs partielles

```apex
Database.SaveResult[] results = Database.insert(records, false);

for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        for (Database.Error error : results[i].getErrors()) {
            System.debug(LoggingLevel.ERROR,
                'Erreur sur record index ' + i + ': ' + error.getMessage());
        }
    }
}
```

---

# 18. Asynchrone : Future, Queueable, Batch, Scheduled

L’asynchrone permet de déplacer un traitement hors de la transaction principale.

## Quand utiliser l’asynchrone ?

- Traitement long.
- Callout HTTP après un DML.
- Traitement de gros volume.
- Découplage entre action utilisateur et traitement métier.
- Réduction du risque CPU dans un trigger.

## Future

Simple, mais limité.

```apex
public with sharing class AccountAsyncService {
    @future
    public static void processAccounts(Set<Id> accountIds) {
        // Traitement async
    }
}
```

## Queueable

Préféré pour beaucoup de cas modernes, car plus structuré.

```apex
public with sharing class AccountQueueable implements Queueable {
    private Set<Id> accountIds;

    public AccountQueueable(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext context) {
        List<Account> accounts = [
            SELECT Id, Name
            FROM Account
            WHERE Id IN :accountIds
        ];

        // Traitement
    }
}
```

Lancement :

```apex
System.enqueueJob(new AccountQueueable(accountIds));
```

## Batch Apex

À utiliser pour de gros volumes.

```apex
public with sharing class AccountBatch implements Database.Batchable<SObject> {
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, Name
            FROM Account
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        for (Account acc : scope) {
            acc.Description = 'Processed by batch';
        }
        update scope;
    }

    public void finish(Database.BatchableContext bc) {
        // Notification ou suite du traitement
    }
}
```

Lancement :

```apex
Database.executeBatch(new AccountBatch(), 200);
```

## Scheduled Apex

```apex
public with sharing class AccountScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        Database.executeBatch(new AccountBatch(), 200);
    }
}
```

## Règles de prudence

- Évite de chaîner des queueables sans garde.
- Prévois un interrupteur fonctionnel si un job peut se relancer.
- Logge les erreurs importantes.
- Ne suppose pas que l’exécution est immédiate.
- Ne transmets pas de gros objets en paramètre, transmets plutôt des IDs.

---

# 19. Callouts HTTP

## Service de callout

```apex
public with sharing class ExternalApiClient {
    public static String getData(String endpoint) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint(endpoint);
        req.setMethod('GET');
        req.setTimeout(120000);

        Http http = new Http();
        HttpResponse res = http.send(req);

        if (res.getStatusCode() >= 200 && res.getStatusCode() < 300) {
            return res.getBody();
        }

        throw new CalloutException('Erreur HTTP: ' + res.getStatusCode() + ' - ' + res.getBody());
    }
}
```

## Bonnes pratiques

- Utilise Named Credentials quand possible.
- Ne hardcode pas les URLs sensibles.
- Gère les timeouts.
- Gère les statuts HTTP non 2xx.
- Prévois des mocks pour les tests.
- Évite les callouts dans des transactions complexes avec DML.

## Mock de test

```apex
@IsTest
private class ExternalApiMock implements HttpCalloutMock {
    public HTTPResponse respond(HTTPRequest req) {
        HttpResponse res = new HttpResponse();
        res.setStatusCode(200);
        res.setBody('{"status":"ok"}');
        return res;
    }
}
```

---

# 20. Tests unitaires

Les tests Apex ne sont pas seulement une obligation de couverture. Ils doivent prouver que le code fonctionne.

## Règles de test

- Créer ses propres données de test.
- Ne pas dépendre de données existantes.
- Tester le bulk.
- Tester les cas négatifs.
- Tester les changements de valeurs.
- Utiliser `Test.startTest()` et `Test.stopTest()`.
- Utiliser des assertions.
- Tester l’asynchrone avec `Test.stopTest()`.

## Exemple simple

```apex
@IsTest
private class AccountServiceTest {
    @IsTest
    static void normalizeNames_shouldTrimAccountNames() {
        List<Account> accounts = new List<Account>{
            new Account(Name = '  Acme  '),
            new Account(Name = '  Global Corp  ')
        };

        Test.startTest();
        AccountService.normalizeNames(accounts);
        Test.stopTest();

        System.assertEquals('Acme', accounts[0].Name);
        System.assertEquals('Global Corp', accounts[1].Name);
    }
}
```

## Test bulk

```apex
@IsTest
private class ContactServiceTest {
    @IsTest
    static void updateAccountLastContactDate_shouldWorkInBulk() {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Account ' + i));
        }
        insert accounts;

        List<Contact> contacts = new List<Contact>();
        for (Account acc : accounts) {
            contacts.add(new Contact(
                LastName = 'Contact',
                AccountId = acc.Id
            ));
        }

        Test.startTest();
        insert contacts;
        Test.stopTest();

        List<Account> results = [
            SELECT Id, Last_Contact_Created_Date__c
            FROM Account
            WHERE Id IN :accounts
        ];

        for (Account acc : results) {
            System.assertEquals(Date.today(), acc.Last_Contact_Created_Date__c);
        }
    }
}
```

## Test d’exception

```apex
@IsTest
private class AccountValidationServiceTest {
    @IsTest
    static void validateAccounts_shouldThrowWhenNameIsMissing() {
        Account acc = new Account();

        try {
            AccountValidationService.validate(new List<Account>{ acc });
            System.assert(false, 'Une exception aurait dû être levée.');
        } catch (ApplicationException e) {
            System.assert(e.getMessage().contains('Name'));
        }
    }
}
```

---

# 21. Factory de données de test

Une factory évite de dupliquer la création des données dans tous les tests.

```apex
@IsTest
public class TestDataFactory {
    public static Account createAccount(Boolean doInsert) {
        Account acc = new Account(Name = 'Test Account');
        if (doInsert) {
            insert acc;
        }
        return acc;
    }

    public static Contact createContact(Id accountId, Boolean doInsert) {
        Contact c = new Contact(
            LastName = 'Test Contact',
            AccountId = accountId
        );
        if (doInsert) {
            insert c;
        }
        return c;
    }

    public static List<Account> createAccounts(Integer count, Boolean doInsert) {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < count; i++) {
            accounts.add(new Account(Name = 'Test Account ' + i));
        }
        if (doInsert) {
            insert accounts;
        }
        return accounts;
    }
}
```

Utilisation :

```apex
Account acc = TestDataFactory.createAccount(true);
Contact c = TestDataFactory.createContact(acc.Id, true);
```

---

# 22. Debugging et diagnostics

## Debug propre

```apex
System.debug(LoggingLevel.INFO, 'Début traitement AccountService');
System.debug(LoggingLevel.ERROR, 'Erreur: ' + e.getMessage());
```

## Ce qu’il faut éviter

- Logs énormes.
- Debug de données sensibles.
- Debug permanent dans du code critique.
- Messages vagues comme `System.debug('ici')`.

## Diagnostic de limites

```apex
System.debug(LoggingLevel.INFO,
    'SOQL: ' + Limits.getQueries() + '/' + Limits.getLimitQueries()
    + ', DML: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements()
    + ', CPU: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime()
);
```

---

# 23. Qualité de code et conventions

## Classe

```apex
public with sharing class InvoiceService {
    private static final String STATUS_DRAFT = 'Draft';

    public static void processInvoices(List<Invoice__c> invoices) {
        if (invoices == null || invoices.isEmpty()) {
            return;
        }

        // Logique métier
    }
}
```

## Méthodes

Une méthode doit :

- avoir un nom clair ;
- faire une seule chose principale ;
- recevoir des paramètres explicites ;
- retourner un résultat utile si nécessaire ;
- éviter les effets de bord cachés.

## Commentaires

Commente le “pourquoi”, pas le “quoi”.

Mauvais :

```apex
// On boucle sur les comptes
for (Account acc : accounts) {
}
```

Bon :

```apex
// On ignore les comptes sans Industry car le scoring métier n’est pas applicable.
for (Account acc : accounts) {
}
```

---

# 24. Checklist avant commit

## Bulk

- [ ] Aucun SOQL dans une boucle.
- [ ] Aucun DML dans une boucle.
- [ ] Le code fonctionne avec 200 enregistrements.
- [ ] Les doublons sont évités avec `Set` ou `Map`.

## Triggers

- [ ] Un seul trigger par objet.
- [ ] Le trigger ne contient presque aucune logique.
- [ ] Le handler délègue aux services.
- [ ] Les récursions sont contrôlées.

## Sécurité

- [ ] `with sharing` ou choix justifié.
- [ ] CRUD/FLS pris en compte si code exposé à l’utilisateur.
- [ ] Pas d’exposition de données sensibles dans les logs.

## Tests

- [ ] Tests avec données créées dans le test.
- [ ] Assertions présentes.
- [ ] Test bulk présent.
- [ ] Cas négatifs testés.
- [ ] Asynchrone testé avec `Test.stopTest()`.

## Maintenance

- [ ] Noms clairs.
- [ ] Méthodes raisonnablement courtes.
- [ ] Pas de hardcoded IDs.
- [ ] Logique métier centralisée.
- [ ] Requêtes SOQL centralisées si réutilisées.

---

# 25. Prompts IA prêts à l’emploi

## Générer une classe Apex

```text
Crée une classe Apex pour [besoin]. Le code doit être bulkifié, testable, respecter les governor limits, ne contenir aucun SOQL/DML dans une boucle, utiliser with sharing sauf justification contraire, et inclure une classe de test avec assertions et cas bulk.
```

## Revoir du code Apex

```text
Analyse ce code Apex. Vérifie : bulkification, SOQL/DML dans les boucles, governor limits, récursion trigger, séparation trigger/handler/service, sécurité CRUD/FLS/sharing, lisibilité, tests unitaires et maintenabilité. Donne une liste claire des problèmes et une version corrigée.
```

## Créer un trigger propre

```text
Crée un trigger Apex propre pour l’objet [Objet] sur les événements [événements]. Le trigger doit déléguer à un handler, le handler à une classe service. Le code doit être bulkifié, éviter la récursion et inclure une classe de test bulk.
```

## Créer des tests

```text
Crée une classe de test Apex complète pour cette classe. Les tests doivent créer leurs propres données, utiliser Test.startTest/Test.stopTest, inclure des assertions, tester les cas bulk et les cas d’erreur.
```

---

# 26. Templates Apex réutilisables

## Template Trigger

```apex
trigger ObjectNameTrigger on ObjectName__c (
    before insert,
    before update,
    after insert,
    after update,
    before delete,
    after delete,
    after undelete
) {
    ObjectNameTriggerHandler handler = new ObjectNameTriggerHandler();

    if (Trigger.isBefore) {
        if (Trigger.isInsert) handler.beforeInsert(Trigger.new);
        if (Trigger.isUpdate) handler.beforeUpdate(Trigger.new, Trigger.oldMap);
        if (Trigger.isDelete) handler.beforeDelete(Trigger.old, Trigger.oldMap);
    }

    if (Trigger.isAfter) {
        if (Trigger.isInsert) handler.afterInsert(Trigger.new);
        if (Trigger.isUpdate) handler.afterUpdate(Trigger.new, Trigger.oldMap);
        if (Trigger.isDelete) handler.afterDelete(Trigger.old, Trigger.oldMap);
        if (Trigger.isUndelete) handler.afterUndelete(Trigger.new);
    }
}
```

## Template Handler

```apex
public with sharing class ObjectNameTriggerHandler {
    public void beforeInsert(List<ObjectName__c> newRecords) {}

    public void beforeUpdate(
        List<ObjectName__c> newRecords,
        Map<Id, ObjectName__c> oldRecordsById
    ) {}

    public void beforeDelete(
        List<ObjectName__c> oldRecords,
        Map<Id, ObjectName__c> oldRecordsById
    ) {}

    public void afterInsert(List<ObjectName__c> newRecords) {}

    public void afterUpdate(
        List<ObjectName__c> newRecords,
        Map<Id, ObjectName__c> oldRecordsById
    ) {}

    public void afterDelete(
        List<ObjectName__c> oldRecords,
        Map<Id, ObjectName__c> oldRecordsById
    ) {}

    public void afterUndelete(List<ObjectName__c> newRecords) {}
}
```

## Template Service

```apex
public with sharing class ObjectNameService {
    public static void process(List<ObjectName__c> records) {
        if (records == null || records.isEmpty()) {
            return;
        }

        // Logique métier bulkifiée
    }
}
```

## Template Selector

```apex
public with sharing class ObjectNameSelector {
    public static Map<Id, ObjectName__c> selectByIds(Set<Id> recordIds) {
        if (recordIds == null || recordIds.isEmpty()) {
            return new Map<Id, ObjectName__c>();
        }

        return new Map<Id, ObjectName__c>([
            SELECT Id, Name
            FROM ObjectName__c
            WHERE Id IN :recordIds
        ]);
    }
}
```

## Template Test

```apex
@IsTest
private class ObjectNameServiceTest {
    @IsTest
    static void process_shouldHandleBulkRecords() {
        List<ObjectName__c> records = new List<ObjectName__c>();
        for (Integer i = 0; i < 200; i++) {
            records.add(new ObjectName__c(Name = 'Record ' + i));
        }
        insert records;

        Test.startTest();
        ObjectNameService.process(records);
        Test.stopTest();

        List<ObjectName__c> results = [
            SELECT Id, Name
            FROM ObjectName__c
            WHERE Id IN :records
        ];

        System.assertEquals(200, results.size());
    }
}
```

---

# 27. Anti-patterns à éviter

## SOQL dans une boucle

```apex
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
}
```

## DML dans une boucle

```apex
for (Contact c : contacts) {
    update c;
}
```

## Trigger trop gros

```apex
trigger AccountTrigger on Account (after update) {
    // 300 lignes de logique métier
}
```

## Hardcoded IDs

```apex
Id profileId = '00eXXXXXXXXXXXX';
```

Préférer requêter par DeveloperName ou utiliser Custom Metadata selon le cas.

## Tests sans assertions

```apex
@IsTest
static void testSomething() {
    insert new Account(Name = 'Test');
}
```

Un test sans assertion prouve très peu de choses.

## Dépendance aux données existantes

```apex
Account acc = [SELECT Id FROM Account LIMIT 1];
```

À éviter dans les tests. Crée tes propres données.

---

# 28. Plan d’apprentissage débutant

## Étape 1 : bases

- Comprendre les objets Salesforce.
- Comprendre `SObject`.
- Faire insert/update/delete.
- Écrire des classes simples.
- Écrire des tests simples.

## Étape 2 : SOQL et collections

- `List`
- `Set`
- `Map`
- SOQL parent/enfant
- Requêtes avec `WHERE Id IN :ids`

## Étape 3 : triggers propres

- Trigger minimal.
- Handler.
- Service.
- Bulkification.
- Tests bulk.

## Étape 4 : limites et performance

- Comprendre les governor limits.
- Mesurer avec `Limits`.
- Éviter CPU inutile.
- Utiliser l’asynchrone si nécessaire.

## Étape 5 : code professionnel

- Sécurité CRUD/FLS/sharing.
- Gestion des erreurs.
- Batch/Queueable.
- Logs et diagnostics.
- Revue de code.

---

# Conclusion

Un bon développeur Apex n’est pas seulement quelqu’un qui connaît la syntaxe. C’est quelqu’un qui respecte la plateforme Salesforce.

Avant de livrer du code Apex, pose toujours ces questions :

1. Est-ce bulkifié ?
2. Est-ce compatible avec les limites ?
3. Est-ce testable ?
4. Est-ce sécurisé ?
5. Est-ce maintenable ?
6. Est-ce compréhensible par un autre développeur ?

Si la réponse est oui, tu es sur la bonne voie.
