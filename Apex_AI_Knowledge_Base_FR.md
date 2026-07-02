

---

<!-- Fichier: 00_INSTRUCTIONS_IA_APEX.md -->

# Instructions obligatoires pour IA — Projets Apex Salesforce

## Rôle

Tu es une IA experte Salesforce Apex. Ton objectif est d’aider à concevoir, générer, auditer et maintenir du code Apex professionnel.

Tu dois privilégier la robustesse, la maintenabilité, la sécurité, la bulkification et le respect des limites Salesforce plutôt que la solution la plus courte.

## Règles non négociables

1. Ne jamais écrire de SOQL dans une boucle.
2. Ne jamais écrire de DML dans une boucle.
3. Toujours concevoir les méthodes Apex pour traiter des collections, même si le besoin initial concerne un seul enregistrement.
4. Toujours supposer qu’un trigger peut recevoir jusqu’à 200 enregistrements.
5. Toujours supposer que d’autres automatisations existent : Flows, validations, autres triggers, packages installés.
6. Toujours isoler la logique métier hors des triggers.
7. Toujours réduire le nombre de requêtes, DML, appels asynchrones, allocations heap et opérations CPU coûteuses.
8. Toujours prévoir les erreurs partielles lorsque cela est pertinent.
9. Toujours écrire ou proposer des tests unitaires significatifs.
10. Toujours expliciter les hypothèses fonctionnelles et techniques.

## Comportement attendu de l’IA

Avant de générer du code, l’IA doit répondre mentalement aux questions suivantes :

- Quel est le contexte d’exécution possible ?
- Le code est-il bulk-safe ?
- Le code est-il récursif ou peut-il être rappelé dans la même transaction ?
- Les limites gouverneur sont-elles protégées ?
- Le traitement doit-il rester synchrone ou être déporté en asynchrone ?
- Les accès aux données respectent-ils le modèle de sécurité attendu ?
- Les tests couvrent-ils succès, erreurs, bulk et absence de données ?

## Format de réponse recommandé

Quand tu proposes une solution Apex, structure la réponse ainsi :

1. Hypothèses
2. Architecture proposée
3. Code Apex
4. Tests unitaires
5. Points de vigilance limites / sécurité / concurrence
6. Checklist de validation

## Ce que l’IA doit refuser ou corriger

L’IA doit signaler explicitement toute demande qui mène à :

- une requête SOQL dans une boucle ;
- un DML dans une boucle ;
- une logique métier lourde directement dans un trigger ;
- un code non testable ;
- un contournement volontaire des limites ou de la sécurité ;
- une dépendance non documentée à des données réelles en test.


---

<!-- Fichier: 01_PRINCIPES_FONDAMENTAUX.md -->

# Principes fondamentaux Apex pour IA

## Penser Apex, pas Java

Apex ressemble syntaxiquement à Java ou C#, mais son comportement réel dépend fortement de la plateforme Salesforce. L’IA doit donc raisonner en termes de transaction Salesforce, contexte d’exécution, limites gouverneur, ordre d’exécution et automatisations déclaratives.

## Les quatre concepts à vérifier systématiquement

### 1. Contexte d’exécution

Un contexte d’exécution est la transaction dans laquelle s’exécute le code Apex. Plusieurs morceaux de code peuvent partager ce même contexte : triggers, Flows, validations, workflows historiques, Process Builder, appels synchrones et packages.

### 2. Variables statiques

Les variables statiques vivent uniquement pendant le contexte d’exécution. Elles ne sont pas un stockage permanent. Elles sont utiles pour le cache transactionnel, les guards anti-récursion et les comportements de test contrôlés.

### 3. Limites gouverneur

Les limites ne sont pas un détail technique. Elles structurent l’architecture : nombre de requêtes, lignes interrogées, DML, CPU, heap, callouts, jobs asynchrones.

### 4. Bulk patterns

Tout code doit être conçu pour fonctionner sur plusieurs enregistrements. Une méthode qui accepte un seul `Id` est souvent moins robuste qu’une méthode qui accepte un `Set<Id>`.

## Règle de conception

Toujours concevoir d’abord la version bulk, puis fournir éventuellement une surcharge pour un seul enregistrement.

```apex
public static void processOne(Id recordId) {
    processMany(new Set<Id>{ recordId });
}

public static void processMany(Set<Id> recordIds) {
    if (recordIds == null || recordIds.isEmpty()) return;
    // Requête unique, traitement en mémoire, DML groupé.
}
```


---

<!-- Fichier: README.md -->

# Base de connaissance IA — Apex Salesforce

Cette base de connaissance est destinée à être utilisée comme contexte par une IA qui génère, corrige ou audite du code Apex Salesforce.

Elle ne remplace pas la documentation officielle Salesforce. Elle formalise des règles opérationnelles inspirées des pratiques avancées Apex : contexte d’exécution, variables statiques, limites gouverneur, bulkification, triggers, asynchrone, concurrence, configuration, tests, packaging et maintenance.

## Utilisation recommandée

Avant chaque demande de génération de code Apex, fournir à l’IA au minimum :

1. `00_INSTRUCTIONS_IA_APEX.md`
2. `01_PRINCIPES_FONDAMENTAUX.md`
3. le fichier thématique correspondant au besoin
4. `checklists/CHECKLIST_CODE_REVIEW_APEX.md`
5. `prompts/PROMPT_SYSTEME_IA_APEX.md`

## Règle principale

L’IA doit toujours produire du code Apex :

- bulkifié ;
- testable ;
- compatible avec les limites gouverneur ;
- lisible et maintenable ;
- robuste face aux Flows, triggers, validations et packages existants ;
- sécurisé selon le contexte CRUD/FLS/sharing.


---

<!-- Fichier: checklists/CHECKLIST_CODE_REVIEW_APEX.md -->

# Checklist de revue de code Apex pour IA

## Bulkification

- [ ] Aucun SOQL dans une boucle.
- [ ] Aucun DML dans une boucle.
- [ ] Les méthodes principales acceptent des collections.
- [ ] Les IDs sont dédupliqués avec `Set<Id>`.
- [ ] Les résultats SOQL sont organisés en `Map`.

## Limites

- [ ] Nombre de SOQL minimal.
- [ ] Nombre de DML minimal.
- [ ] Pas de boucle imbriquée évitable.
- [ ] Pas de sérialisation ou calcul coûteux répété inutilement.
- [ ] Traitement asynchrone envisagé si volume ou CPU élevé.

## Triggers

- [ ] Trigger sans logique métier lourde.
- [ ] Handler/service utilisé.
- [ ] `before` privilégié pour modifier le même record.
- [ ] Récursion contrôlée avec précision.
- [ ] Interaction Flow/validation considérée.

## Sécurité

- [ ] `with sharing`, `without sharing` ou `inherited sharing` choisi volontairement.
- [ ] CRUD/FLS vérifiés si données manipulées depuis contexte utilisateur.
- [ ] Pas d’exposition inutile en `global` ou `public`.
- [ ] Pas de données sensibles dans les logs.

## Tests

- [ ] Tests avec données créées dans le test.
- [ ] Tests bulk, cas vide, cas nominal, cas erreur.
- [ ] Assertions utiles.
- [ ] Asynchrone testé avec `Test.startTest()` / `Test.stopTest()`.
- [ ] Pas de dépendance fragile à l’ordre des données.

## Maintenabilité

- [ ] Noms explicites.
- [ ] Responsabilités séparées.
- [ ] Configuration externalisée.
- [ ] Hypothèses documentées.
- [ ] Code lisible avant d’être clever.


---

<!-- Fichier: checklists/CHECKLIST_SECURITE_APEX.md -->

# Checklist sécurité Apex

- [ ] La classe utilise un mode de sharing explicite.
- [ ] Les opérations appelées depuis LWC/Aura respectent CRUD/FLS.
- [ ] Les requêtes dynamiques protègent les injections SOQL.
- [ ] Les erreurs retournées à l’UI ne divulguent pas d’informations internes.
- [ ] Les logs ne contiennent pas de secrets ou données sensibles.
- [ ] Les endpoints REST valident les entrées.
- [ ] Les permissions nécessaires sont documentées.


---

<!-- Fichier: patterns/06_COLLECTIONS.md -->

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


---

<!-- Fichier: patterns/07_TRIGGERS.md -->

# Triggers Apex

## Règle d’architecture

Un trigger doit être un routeur minimal. Il ne doit pas contenir de logique métier complexe.

## Structure recommandée

```apex
trigger AccountTrigger on Account (
    before insert, before update,
    after insert, after update
) {
    AccountTriggerHandler.run();
}
```

```apex
public class AccountTriggerHandler {
    public static void run() {
        if (Trigger.isBefore && Trigger.isInsert) {
            AccountService.beforeInsert((List<Account>) Trigger.new);
        }
        if (Trigger.isBefore && Trigger.isUpdate) {
            AccountService.beforeUpdate(
                (List<Account>) Trigger.new,
                (Map<Id, Account>) Trigger.oldMap
            );
        }
    }
}
```

## Before vs After

Utiliser `before` lorsque l’objectif est de modifier les champs du même enregistrement. Cela évite un DML supplémentaire.

Utiliser `after` lorsque l’Id est nécessaire ou lorsqu’il faut agir sur des objets liés.

## Récursion

Ne pas utiliser un simple booléen global pour tout bloquer. Préférer une clé ciblée par opération.

## Ordre d’exécution

L’IA doit supposer que :

- d’autres triggers existent ;
- des Flows peuvent modifier les mêmes enregistrements ;
- des validations peuvent faire échouer le DML ;
- le code peut être exécuté de nouveau dans la même transaction.

## Checklist IA

- [ ] Trigger sans logique métier lourde ?
- [ ] Un seul point d’entrée par objet dans le projet ?
- [ ] Service bulkifié appelé depuis le handler ?
- [ ] `before` utilisé quand possible ?
- [ ] Récursion contrôlée précisément ?
- [ ] Tests couvrant insert/update/delete si applicable ?


---

<!-- Fichier: patterns/08_ASYNCHRONE.md -->

# Apex asynchrone

## Quand utiliser l’asynchrone ?

Utiliser l’asynchrone lorsque le traitement :

- consomme trop de CPU ;
- effectue des callouts ;
- traite un volume important ;
- peut être différé ;
- doit être isolé du contexte transactionnel courant.

## Choisir le bon mécanisme

| Besoin | Option recommandée |
|---|---|
| Traitement différé simple | Queueable |
| Callout contrôlé | Queueable avec `Database.AllowsCallouts` |
| Gros volumes | Batch Apex |
| Planification | Scheduled Apex |
| Notification événementielle | Platform Event |
| Réaction à changement de données | Change Data Capture |

## Queueable recommandé

```apex
public class AccountRecalculationJob implements Queueable {
    private Set<Id> accountIds;

    public AccountRecalculationJob(Set<Id> accountIds) {
        this.accountIds = accountIds == null ? new Set<Id>() : new Set<Id>(accountIds);
    }

    public void execute(QueueableContext context) {
        if (accountIds.isEmpty()) return;
        AccountService.recalculate(accountIds);
    }
}
```

## Attention aux appels asynchrones multiples

Regrouper les demandes dans un seul job quand possible. Ne pas créer un job par enregistrement.

## Pattern : éviter l’enchaînement incontrôlé

- stocker l’état de progression ;
- limiter explicitement la profondeur ;
- journaliser les erreurs ;
- prévoir une reprise.

## Platform Events

Utiles pour la journalisation durable de certains échecs ou pour communiquer avec d’autres composants. À utiliser avec prudence pour ne pas transformer chaque traitement en architecture événementielle inutilement complexe.

## Checklist IA

- [ ] Le traitement doit-il vraiment être synchrone ?
- [ ] Le volume justifie-t-il Batch plutôt que Queueable ?
- [ ] Les callouts sont-ils séparés du DML incompatible ?
- [ ] Un job unique regroupe-t-il plusieurs enregistrements ?
- [ ] Les erreurs asynchrones sont-elles traçables ?


---

<!-- Fichier: patterns/09_CONCURRENCE_VERROUILLAGE.md -->

# Concurrence, verrouillage et data skew

## Problème

Salesforce peut exécuter plusieurs transactions en parallèle. Des erreurs comme `UNABLE_TO_LOCK_ROW` peuvent survenir lorsque plusieurs transactions modifient les mêmes enregistrements ou des parents fortement partagés.

## Règles IA

- Identifier les objets parents fréquemment mis à jour.
- Éviter les mises à jour inutiles sur les mêmes parents.
- Regrouper les DML par objet et par parent.
- Prévoir une stratégie de retry uniquement pour les erreurs transitoires.

## Verrouillage pessimiste

Utiliser `FOR UPDATE` uniquement si nécessaire.

```apex
List<Account> accounts = [
    SELECT Id, Custom_Count__c
    FROM Account
    WHERE Id IN :accountIds
    FOR UPDATE
];
```

## Retry contrôlé

Un retry doit être :

- limité en nombre ;
- journalisé ;
- asynchrone si nécessaire ;
- réservé aux erreurs transitoires.

## Data skew

Surveiller les cas où beaucoup d’enregistrements enfants pointent vers le même parent, ou lorsqu’un propriétaire possède un volume très élevé d’enregistrements. Cela peut amplifier les conflits de verrouillage et les problèmes de performance.

## Checklist IA

- [ ] Le code met-il à jour des parents communs ?
- [ ] Les updates inutiles sont-ils évités ?
- [ ] Le risque `UNABLE_TO_LOCK_ROW` est-il identifié ?
- [ ] Un traitement asynchrone/retry est-il nécessaire ?
- [ ] Les gros volumes par parent/propriétaire sont-ils pris en compte ?


---

<!-- Fichier: patterns/10_CONFIGURATION.md -->

# Configuration applicative Apex

## Principe

Ne pas coder en dur ce qui doit varier par org, environnement, package ou client.

## Options de configuration

| Besoin | Solution |
|---|---|
| Paramètre déployable et versionnable | Custom Metadata Type |
| Paramètre modifiable par admin | Custom Metadata ou Custom Setting selon cas |
| Donnée transactionnelle | Objet custom |
| Cache temporaire partagé | Platform Cache |
| Feature flag | Custom Metadata + service de configuration |

## Service de configuration

```apex
public class AppConfig {
    private static Map<String, App_Config__mdt> cacheByName;

    public static App_Config__mdt get(String developerName) {
        if (cacheByName == null) {
            cacheByName = new Map<String, App_Config__mdt>();
            for (App_Config__mdt cfg : [
                SELECT DeveloperName, Enabled__c, Value__c
                FROM App_Config__mdt
            ]) {
                cacheByName.put(cfg.DeveloperName, cfg);
            }
        }
        return cacheByName.get(developerName);
    }
}
```

## Règles IA

- Centraliser l’accès à la configuration.
- Ne pas répéter des requêtes de configuration partout.
- Prévoir des valeurs par défaut sûres.
- Rendre les feature flags explicites.

## Checklist IA

- [ ] Aucun identifiant Salesforce codé en dur ?
- [ ] Les paramètres métier sont-ils externalisés ?
- [ ] L’accès à la configuration est-il centralisé ?
- [ ] Le comportement par défaut est-il sûr ?


---

<!-- Fichier: patterns/11_DEBUG_DIAGNOSTICS.md -->

# Debugging et diagnostics Apex

## Objectif

Le diagnostic ne consiste pas seulement à écrire `System.debug`. Il faut pouvoir comprendre un incident en production sans dépasser les limites ni exposer des données sensibles.

## Règles IA

- Prévoir une journalisation contrôlée par configuration.
- Éviter les logs volumineux dans les boucles.
- Ne jamais journaliser de secrets, tokens, mots de passe ou données sensibles inutiles.
- Prévoir des corrélations : request id, record id, job id, contexte.

## Pattern de logger minimal

```apex
public class AppLogger {
    public enum Level { ERROR, WARN, INFO, DEBUG }

    public static void error(String message, Exception ex) {
        System.debug(LoggingLevel.ERROR, message + ' - ' + ex.getMessage());
    }
}
```

## Diagnostic d’erreurs limites

Certaines erreurs, comme les dépassements CPU, peuvent interrompre brutalement la transaction. Pour les traitements critiques, envisager des mécanismes de journalisation plus résilients, par exemple événements ou objet de log dans les limites acceptables.

## Checklist IA

- [ ] Les erreurs sont-elles capturées au bon niveau ?
- [ ] Les logs sont-ils activables/désactivables ?
- [ ] Les données sensibles sont-elles protégées ?
- [ ] Les jobs asynchrones ont-ils une trace d’exécution ?


---

<!-- Fichier: patterns/12_TESTS_UNITAIRES.md -->

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


---

<!-- Fichier: patterns/13_PACKAGING_DEPLOIEMENT.md -->

# Packaging et déploiement

## Principe

Sur Salesforce, le mode de déploiement influence l’architecture. Une solution spécifique à une org et un package distribué n’ont pas les mêmes contraintes.

## Règles IA

- Demander ou expliciter le contexte : projet interne, client unique, package managé, unlocked package.
- Éviter les dépendances implicites à la configuration d’une seule org.
- Prévoir l’évolution des métadonnées.
- Limiter l’exposition `global` au strict nécessaire.
- Prévoir la compatibilité ascendante des APIs publiques.

## Package managé

Attention particulière à :

- classes `global` irréversibles ;
- signatures publiques exposées ;
- noms de champs/objets ;
- configuration par Custom Metadata ;
- tests indépendants des données client ;
- interactions avec automatisations inconnues.

## Déploiement projet interne

Attention particulière à :

- conventions d’équipe ;
- dépendances avec Flows existants ;
- données de production ;
- stratégie Git/SFDX ;
- rollback.

## Checklist IA

- [ ] Le type de déploiement est-il identifié ?
- [ ] Les dépendances sont-elles documentées ?
- [ ] Les APIs exposées sont-elles stables ?
- [ ] Les tests sont-ils portables ?
- [ ] La configuration est-elle déployable ?


---

<!-- Fichier: patterns/14_MAINTENANCE.md -->

# Maintenance Apex

## Principe

Le coût principal d’un logiciel vient de sa maintenance. L’IA doit donc générer du code compréhensible, modulaire, testable et facile à faire évoluer.

## Règles IA

- Favoriser les noms explicites.
- Séparer orchestration, logique métier, accès aux données et configuration.
- Éviter le code magique ou trop dynamique sans justification.
- Documenter les hypothèses et limites.
- Préférer les petites méthodes cohérentes.
- Ne pas optimiser prématurément au prix de la lisibilité, sauf contrainte de limites claire.

## Architecture simple recommandée

```text
force-app/main/default/classes/
├── AccountTriggerHandler.cls
├── AccountService.cls
├── AccountSelector.cls
├── AccountDomain.cls
├── AppConfig.cls
└── AppLogger.cls
```

## Responsabilités

| Classe | Responsabilité |
|---|---|
| TriggerHandler | Router le contexte trigger |
| Service | Orchestrer les cas d’usage |
| Selector | Centraliser SOQL |
| Domain | Règles métier d’un objet |
| Config | Accès configuration |
| Logger | Diagnostics |

## Checklist IA

- [ ] Le code sera-t-il compréhensible dans 6 mois ?
- [ ] Les responsabilités sont-elles séparées ?
- [ ] Les tests protègent-ils les comportements métier ?
- [ ] Les noms sont-ils explicites ?
- [ ] Les dépendances sont-elles visibles ?


---

<!-- Fichier: principes/02_CONTEXTE_EXECUTION.md -->

# Contexte d’exécution Apex

## Définition

Le contexte d’exécution est le cadre transactionnel dans lequel Apex s’exécute. Il détermine notamment :

- la durée de vie des variables statiques ;
- les limites gouverneur disponibles ;
- les interactions entre code Apex et automatisations Salesforce.

## Points clés pour IA

- Un trigger n’est jamais garanti d’être seul.
- L’ordre de plusieurs triggers sur le même objet n’est pas maîtrisable.
- Un Flow ou une règle de mise à jour peut provoquer une nouvelle exécution de trigger dans la même transaction.
- Les limites sont partagées avec d’autres traitements synchrones.
- Un appel asynchrone démarre un nouveau contexte d’exécution.

## Pattern recommandé : détecter la récursion métier

Utiliser une classe dédiée au contrôle transactionnel plutôt qu’une variable statique dans le trigger.

```apex
public class TransactionGuard {
    private static Set<String> executedKeys = new Set<String>();

    public static Boolean enter(String key) {
        if (executedKeys.contains(key)) return false;
        executedKeys.add(key);
        return true;
    }
}
```

Utilisation :

```apex
if (!TransactionGuard.enter('AccountTrigger.afterUpdate.recalculateScore')) {
    return;
}
```

## Anti-patterns

```apex
trigger AccountTrigger on Account (after update) {
    update Trigger.new; // Risque de récursion et de DML inutile.
}
```

```apex
public static Boolean alreadyRan = false;
// Trop global : peut bloquer des traitements légitimes sur d’autres lots.
```

## Checklist IA

- [ ] Le code peut-il être rappelé plusieurs fois dans la même transaction ?
- [ ] La récursion est-elle contrôlée sans bloquer des cas légitimes ?
- [ ] Les traitements asynchrones sont-ils séparés du contexte synchrone ?
- [ ] Les effets des Flows et validations sont-ils pris en compte ?


---

<!-- Fichier: principes/03_VARIABLES_STATIQUES.md -->

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


---

<!-- Fichier: principes/04_LIMITES_GOUVERNEUR.md -->

# Limites gouverneur Apex

## Principe

Les limites gouverneur sont une contrainte d’architecture. Elles obligent à optimiser l’accès aux données, les DML, le CPU, le heap et les traitements asynchrones.

## Règles IA

- Concevoir pour consommer moins que les limites, pas juste pour rester sous le maximum.
- Supposer que d’autres traitements partagent la même transaction.
- Éviter les opérations inutiles, même si elles semblent petites.
- Déplacer les traitements lourds vers Queueable, Batch ou Platform Event si nécessaire.

## SOQL

Bonnes pratiques :

- Une requête par objet/contexte métier lorsque possible.
- Utiliser `Set<Id>` pour filtrer.
- Charger uniquement les champs nécessaires.
- Regrouper les requêtes liées.
- Penser à la sélectivité pour les grands volumes.

```apex
Set<Id> accountIds = new Set<Id>();
for (Contact c : contacts) {
    if (c.AccountId != null) accountIds.add(c.AccountId);
}

Map<Id, Account> accountsById = new Map<Id, Account>([
    SELECT Id, Name, OwnerId
    FROM Account
    WHERE Id IN :accountIds
]);
```

## DML

- Construire une liste `recordsToUpdate`.
- Ne faire le DML que si la liste n’est pas vide.
- Utiliser `Database.update(records, false)` si une réussite partielle est acceptable.

```apex
if (!accountsToUpdate.isEmpty()) {
    Database.SaveResult[] results = Database.update(accountsToUpdate, false);
}
```

## CPU

Réduire :

- boucles imbriquées ;
- accès dynamiques inutiles ;
- sérialisations JSON répétées ;
- recalculs évitables ;
- traitements synchrones lourds.

## Heap

- Ne pas cacher de gros volumes dans des variables statiques.
- Traiter en batch si le volume est élevé.
- Ne pas charger des champs inutiles.

## Checklist IA

- [ ] SOQL hors boucle ?
- [ ] DML hors boucle ?
- [ ] Requêtes sélectives ?
- [ ] Champs limités au nécessaire ?
- [ ] Pas de boucle O(n²) évitable ?
- [ ] Traitement asynchrone envisagé si CPU/heap élevé ?


---

<!-- Fichier: principes/05_BULK_PATTERNS.md -->

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


---

<!-- Fichier: prompts/PROMPT_AUDIT_CODE_APEX.md -->

# Prompt d’audit de code Apex

Analyse le code Apex fourni comme un reviewer senior Salesforce.

Retourne :

1. Résumé du rôle du code.
2. Problèmes bloquants.
3. Problèmes de limites gouverneur.
4. Problèmes de bulkification.
5. Problèmes de sécurité CRUD/FLS/sharing.
6. Problèmes de tests.
7. Problèmes de maintenabilité.
8. Version corrigée ou recommandations concrètes.
9. Checklist finale.

Sois strict sur : SOQL dans boucle, DML dans boucle, récursion trigger, absence d’assertions, dépendance à des données réelles, logs sensibles et classes sans sharing explicite.


---

<!-- Fichier: prompts/PROMPT_SYSTEME_IA_APEX.md -->

# Prompt système IA Apex

Tu es un expert Salesforce Apex senior.

Tu dois générer du code Apex conforme aux règles suivantes :

- code bulkifié par défaut ;
- aucun SOQL/DML dans les boucles ;
- respect des limites gouverneur ;
- architecture claire : trigger handler, service, selector, domain si nécessaire ;
- tests unitaires avec assertions utiles ;
- prise en compte des Flows, validations, autres triggers et packages ;
- sécurité CRUD/FLS/sharing selon contexte ;
- configuration externalisée quand une valeur peut varier.

Avant de donner le code, indique les hypothèses. Après le code, donne les points de vigilance et une checklist de validation.

Si la demande utilisateur conduit à un anti-pattern Apex, explique le risque et propose une alternative correcte.


---

<!-- Fichier: templates/TEMPLATE_SERVICE_APEX.md -->

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


---

<!-- Fichier: templates/TEMPLATE_TRIGGER_HANDLER.md -->

# Template trigger + handler

```apex
trigger ExampleTrigger on Account (
    before insert, before update,
    after insert, after update
) {
    ExampleTriggerHandler.run();
}
```

```apex
public inherited sharing class ExampleTriggerHandler {
    public static void run() {
        if (Trigger.isBefore && Trigger.isInsert) {
            ExampleService.beforeInsert((List<Account>) Trigger.new);
        }
        if (Trigger.isBefore && Trigger.isUpdate) {
            ExampleService.beforeUpdate(
                (List<Account>) Trigger.new,
                (Map<Id, Account>) Trigger.oldMap
            );
        }
    }
}
```
