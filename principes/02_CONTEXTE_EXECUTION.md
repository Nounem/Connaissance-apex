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

## Sharing : `with sharing`, `without sharing`, `inherited sharing`

- `with sharing` : la classe applique les règles de partage (sharing rules, OWD, rôles) de l’utilisateur courant. Les enregistrements auxquels l’utilisateur n’a pas accès en lecture/écriture selon le modèle de partage sont exclus.
- `without sharing` : la classe s’exécute en système, sans tenir compte du partage. À réserver aux traitements batch techniques, intégrations, ou logiques devant volontairement ignorer le partage — **toujours justifié par un commentaire expliquant pourquoi**.
- `inherited sharing` : la classe hérite du mode de partage de la classe appelante. Utile pour les classes utilitaires réutilisées à la fois en contexte `with sharing` et `without sharing`. Si la classe est appelée sans contexte englobant (ex. depuis un contexte système), elle se comporte comme `with sharing`.

Règle par défaut recommandée : déclarer explicitement `with sharing` ou `inherited sharing` sur toute nouvelle classe. Ne jamais laisser une classe sans mot-clé : par défaut, une classe **sans aucun mot-clé** de partage s’exécute en `without sharing` (mode système, sharing non appliqué), **quel que soit le contexte appelant** — elle n’hérite jamais implicitement du sharing de la classe qui l’appelle. C’est précisément pour ce cas que le mot-clé `inherited sharing` a été introduit (Winter '17) : sans lui, une classe omettant le mot-clé ignore le sharing même si elle est appelée depuis une classe `with sharing`. `without sharing` doit rester l’exception documentée.

```apex
public with sharing class AccountService {
    public List<Account> getVisibleAccounts() {
        return [SELECT Id, Name FROM Account];
    }
}

// Justifié : traitement d’intégration devant lire tous les comptes
// indépendamment du partage utilisateur, appelé uniquement depuis un Batch technique.
public without sharing class AccountIntegrationSync {
    public List<Account> getAllAccountsForExport() {
        return [SELECT Id, Name FROM Account];
    }
}
```

## System mode vs user mode (CRUD/FLS)

Le sharing gère la **visibilité des lignes** (quels enregistrements). Les droits CRUD (Create/Read/Update/Delete sur l’objet) et FLS (Field-Level Security) gèrent les **droits sur les objets et les champs** — ce sont deux mécanismes distincts et cumulatifs.

- `WITH USER_MODE` (SOQL, API 56+) : exécute la requête en respectant CRUD, FLS et sharing de l’utilisateur courant, quel que soit le mode de la classe.
  ```apex
  List<Account> accs = [
      SELECT Id, Name, AnnualRevenue
      FROM Account
      WITH USER_MODE
  ];
  ```
- `Security.stripInaccessible()` : filtre a posteriori les champs/objets inaccessibles avant un DML ou après une requête, utile quand `WITH USER_MODE` n’est pas applicable (DML, Database.insert, etc.).
  ```apex
  SObjectAccessDecision decision = Security.stripInaccessible(
      AccessType.CREATABLE, new List<Account>{ newAccount }
  );
  insert decision.getRecords();
  ```
- L’IA doit vérifier CRUD/FLS séparément du sharing : une classe `with sharing` sans `WITH USER_MODE` ni `stripInaccessible()` peut toujours exposer des champs interdits à l’utilisateur.

## Ordre d’exécution Salesforce (DML)

Lors d’une insertion/mise à jour, Salesforce exécute (ordre documenté officiellement, susceptible d’évoluer selon les releases) :

1. Chargement de l’enregistrement original (update) ou initialisation des valeurs par défaut (insert).
2. Chargement des nouvelles valeurs de champs depuis la requête (écrase les anciennes).
3. Si la requête provient d’une page standard (UI uniquement, pas API/Apex) : validation système des règles de layout, champs obligatoires au niveau layout, format et longueur des champs.
4. Before-save Flows (record-triggered, si applicable).
5. Triggers `before insert` / `before update`.
6. Nouvelle validation système des champs obligatoires **et première exécution des règles de validation personnalisées** (Validation Rules) — contrairement à une lecture intuitive, elles ne s’exécutent qu’ici, **après** les before triggers, pas avant les before-save Flows.
7. Duplicate rules — exécutées **après** les before triggers et les Validation Rules, juste avant la sauvegarde. Si la règle bloque l’enregistrement, celui-ci n’est pas sauvegardé et les étapes suivantes (after triggers, workflow rules, etc.) n’ont pas lieu.
8. Sauvegarde de l’enregistrement en base (non commitée).
9. Triggers `after insert` / `after update`.
10. Assignment rules.
11. Auto-response rules.
12. Workflow rules (et leurs field updates, qui redéclenchent une nouvelle sauvegarde et un seul nouveau passage before/after triggers, quel que soit le nombre de règles).
13. Processes (Process Builder, en fin de vie chez Salesforce au profit de Flow) / After-save Flows (record-triggered).
14. Escalation rules / entitlement rules.
15. Recalcul des règles de partage **basées sur des critères** (criteria-based sharing) si des champs impactant ces règles ont changé — ce recalcul synchrone ne couvre pas tous les mécanismes de partage (le partage par propriétaire ou certains recalculs de masse peuvent être traités de façon asynchrone, hors de cette transaction).
16. Commit de la transaction (ou rollback global en cas d’erreur non gérée).

L’IA doit garder à l’esprit que des field updates de workflow/flow peuvent redéclencher les triggers des étapes 5/9, provoquant plusieurs passages dans la même transaction (une seule fois de plus, pas en boucle).

## Gestion transactionnelle : Savepoint / Rollback

```apex
Savepoint sp = Database.setSavepoint();
try {
    insert newAccount;
    update relatedContacts; // Si erreur ici, tout est annulé.
} catch (DmlException e) {
    Database.rollback(sp);
    throw new AuraHandledException('Échec de l’opération : ' + e.getMessage());
}
```

Nuances : `Database.setSavepoint()` compte lui-même comme une instruction DML (limite de 150 par transaction) — poser des savepoints en boucle peut épuiser cette limite. `Database.rollback(sp)` annule les écritures en base effectuées après le savepoint, mais **ne réinitialise pas** les limites gouverneur déjà consommées (nombre de requêtes SOQL, DML, callouts, etc.).

## `System.runAs()` en test

Permet de simuler l’exécution sous un autre utilisateur pour tester le sharing, les profils ou les permissions. Sans `System.runAs()`, les tests s’exécutent par défaut en contexte système (droits complets), masquant les problèmes de sharing/CRUD/FLS.

```apex
@isTest
static void testRestrictedUserCannotSeeAccount() {
    User restrictedUser = TestDataFactory.createUserWithProfile('Standard User');

    System.runAs(restrictedUser) {
        List<Account> accs = [SELECT Id FROM Account WITH USER_MODE];
        Assert.areEqual(0, accs.size(), 'L’utilisateur restreint ne doit rien voir.');
    }
}
```

Nuance : `System.runAs()` ne réinitialise pas les limites gouverneur de la transaction (ce n’est pas son rôle — c’est `Test.startTest()`/`Test.stopTest()` qui ouvre un nouveau jeu de limites) ; ne pas confondre les deux mécanismes. `System.runAs()` ne peut être utilisé que dans une méthode `@isTest`.

## Contexte synchrone vs asynchrone et sharing

- Un `Queueable` ou un `Batchable` **n’hérite pas automatiquement** du mode de sharing de la classe qui l’enqueue/le lance : l’exécution asynchrone démarre un nouveau contexte d’exécution, déconnecté de la pile d’appel synchrone. Sans mot-clé de sharing déclaré sur la classe Queueable/Batchable elle-même, elle s’exécute par défaut en `without sharing` (mode système), exactement comme n’importe quelle classe Apex sans déclaration explicite. Seule une classe explicitement déclarée `inherited sharing` peut se comporter différemment — et faute de contexte appelant direct au moment de l’exécution du job asynchrone, elle se comporte alors comme `with sharing`.
- Un job planifié (`Schedulable`) ou déclenché sans classe appelante explicite s’exécute en système par défaut si aucun mot-clé de sharing n’est déclaré sur sa propre classe.
- L’IA doit toujours vérifier le mot-clé de sharing sur la classe Queueable/Batch/Schedulable elle-même plutôt que de supposer un héritage.

## Checklist IA

- [ ] Le code peut-il être rappelé plusieurs fois dans la même transaction ?
- [ ] La récursion est-elle contrôlée sans bloquer des cas légitimes ?
- [ ] Les traitements asynchrones sont-ils séparés du contexte synchrone ?
- [ ] Les effets des Flows et validations sont-ils pris en compte ?
- [ ] Le mode de sharing (`with sharing` / `without sharing` / `inherited sharing`) est-il explicite et justifié ?
- [ ] CRUD et FLS sont-ils vérifiés séparément du sharing (`WITH USER_MODE` ou `Security.stripInaccessible()`) ?

## Voir aussi

- [`03_VARIABLES_STATIQUES.md`](03_VARIABLES_STATIQUES.md)
- [`../patterns/15_SECURITE_CRUD_FLS_SHARING.md`](../patterns/15_SECURITE_CRUD_FLS_SHARING.md)
- [`../patterns/07_TRIGGERS.md`](../patterns/07_TRIGGERS.md)
