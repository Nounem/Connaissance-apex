# Sécurité Apex : CRUD, FLS et Sharing

## Enjeu

Trois mécanismes distincts, souvent confondus, protègent les données dans Apex :

- **Sharing** : visibilité des **lignes** (quels enregistrements un utilisateur peut voir selon OWD, règles de partage, rôles).
- **CRUD/FLS** : droits sur les **objets** (Create/Read/Update/Delete) et sur les **champs** (Field-Level Security), définis par profil/permission set.
- **Injection SOQL/SOSL** : intégrité de la requête elle-même face à une entrée utilisateur non maîtrisée.

Apex s'exécute par défaut en mode système : sans précaution explicite, une classe Apex ignore le sharing et les droits CRUD/FLS de l'utilisateur courant. C'est le sujet de sécurité le plus critique du langage : l'IA doit systématiquement l'appliquer, même si la demande fonctionnelle ne le mentionne pas.

## Règle IA

Toute classe Apex générée doit déclarer explicitement son mode de sharing, et tout accès aux données exposé à un utilisateur doit appliquer CRUD/FLS via `USER_MODE` (SOQL et DML) ou `Security.stripInaccessible`. Aucune exception sans justification écrite dans le code.

## Sharing : `with sharing`, `without sharing`, `inherited sharing`

### `with sharing` (défaut recommandé)

Applique les règles de partage de l'utilisateur courant à toutes les requêtes et DML de la classe.

```apex
public with sharing class OpportunityService {
    public static List<Opportunity> getOpenOpportunities() {
        // Ne retourne que les opportunités visibles par l'utilisateur courant.
        return [SELECT Id, Name, Amount FROM Opportunity WHERE IsClosed = false];
    }
}
```

Doit être le choix par défaut pour toute nouvelle classe, sauf raison métier explicite.

### `without sharing` (exception justifiée uniquement)

Ignore le sharing de l'utilisateur courant : la classe s'exécute avec une visibilité complète des lignes, quel que soit l'appelant. Cas légitimes limités : traitement système cross-utilisateur explicitement voulu (ex. un batch de réconciliation qui doit lire tous les comptes indépendamment du propriétaire), jamais un simple raccourci pour éviter un problème de visibilité non compris.

```apex
// JUSTIFICATION OBLIGATOIRE : ce service de réconciliation nocturne doit
// pouvoir mettre à jour des comptes appartenant à n'importe quel commercial,
// indépendamment du sharing, car il est déclenché par un batch système
// (cf. US-4821, validé par l'équipe sécurité le 2026-05-10).
public without sharing class AccountReconciliationBatchService {
    public static void reconcileAll(List<Account> accountsToFix) {
        update accountsToFix;
    }
}
```

Ne jamais utiliser `without sharing` sans ce commentaire de justification. Une classe `without sharing` ne doit jamais être directement invocable par un Guest User (voir section dédiée).

### `inherited sharing` (classe utilitaire polyvalente)

Hérite du mode de sharing du contexte appelant : `with sharing` si appelée depuis une classe `with sharing`, `without sharing` si appelée depuis une classe `without sharing`. Utile pour une classe utilitaire réutilisée dans des contextes variés, où imposer un mode fixe serait incorrect. Par défaut (si appelée directement, sans appelant `with sharing` explicite, par ex. depuis un test ou un contexte anonyme), elle se comporte comme `with sharing`.

```apex
public inherited sharing class AddressFormatterService {
    public static String formatBillingAddress(Account acc) {
        return acc.BillingStreet + ', ' + acc.BillingCity;
    }
}
```

## CRUD/FLS : trois approches

Le sharing ne protège pas les champs ni les objets. Un utilisateur peut voir la ligne `Account` mais ne pas avoir le droit de lire `AnnualRevenue`. Trois approches, de la plus récente à la plus ancienne.

### (a) `WITH USER_MODE` en SOQL (API 56+, approche recommandée)

Force l'enforcement CRUD/FLS et sharing de l'utilisateur courant directement au niveau de la requête, y compris pour une classe `without sharing`.

```apex
List<Contact> contacts = [
    SELECT Id, Name, Email, Phone
    FROM Contact
    WHERE AccountId = :accountId
    WITH USER_MODE
];
```

Une requête `WITH USER_MODE` lève une `System.QueryException` aussi bien si l'utilisateur n'a pas le droit `isAccessible()` sur l'objet que s'il n'a pas accès à un ou plusieurs champs référencés dans la requête : contrairement à une intuition répandue, les champs inaccessibles ne sont **pas** silencieusement omis du résultat. La méthode `getInaccessibleFields()` de `QueryException` permet de récupérer d'un coup l'ensemble des champs en cause. C'est l'approche à utiliser en priorité pour tout nouveau code.

### (b) `Security.stripInaccessible` pour filtrage explicite

Filtre une liste d'enregistrements pour retirer les champs non accessibles avant de les exposer (ex. en réponse à un composant LWC), ou avant un DML pour retirer les champs non créables/modifiables.

```apex
// Lecture : retirer les champs non lisibles avant exposition à l'UI.
List<Contact> contacts = [SELECT Id, Name, Email, SSN__c FROM Contact WHERE AccountId = :accountId];
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, contacts);
List<Contact> safeContacts = decision.getRecords();

// Ecriture : retirer les champs non créables avant insert.
List<Contact> newContacts = buildContactsFromImport();
SObjectAccessDecision createDecision = Security.stripInaccessible(AccessType.CREATABLE, newContacts);
insert createDecision.getRecords();

// Mise à jour : retirer les champs non modifiables avant update.
SObjectAccessDecision updateDecision = Security.stripInaccessible(AccessType.UPDATABLE, contactsToUpdate);
update updateDecision.getRecords();
```

Avec la signature à deux paramètres utilisée ci-dessus, `Security.stripInaccessible` ne vérifie pas le droit objet (Create/Read/Update/Delete global) mais uniquement les champs ; il doit être combiné avec une vérification objet (`isAccessible()`, `isCreateable()`) ou avec `USER_MODE` sur la requête/DML pour une couverture complète. Il existe une signature à trois paramètres, `Security.stripInaccessible(accessType, records, enforceRootObjectCRUD)`, qui active explicitement ce contrôle objet (et lève une exception si l'utilisateur n'a pas le droit CRUD correspondant) ; elle n'est pas illustrée ici, la combinaison avec `USER_MODE` restant l'approche la plus simple à généraliser.

### (c) `WITH SECURITY_ENFORCED` (approche plus ancienne)

Lève une exception si l'utilisateur n'a pas accès à un champ ou objet référencé dans la requête. Limites connues : comportement incohérent avec les sous-requêtes (relations enfants) et certains agrégats/fonctions, ce qui en limite l'usage fiable.

```apex
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WHERE Id = :accountId
    WITH SECURITY_ENFORCED
];
```

À mentionner comme existant dans du code legacy, mais **ne pas générer de nouveau code avec `WITH SECURITY_ENFORCED`** : préférer `WITH USER_MODE`. Depuis l'API 67.0 (Winter '26), Salesforce a retiré cette clause : une classe Apex compilée avec cette version d'API (ou une version ultérieure) ne compile plus si elle contient `WITH SECURITY_ENFORCED`. La migration vers `WITH USER_MODE` est donc désormais obligatoire pour tout code portant sa version d'API à 67+, pas seulement une bonne pratique.

## DML en mode utilisateur

`Database.insert`/`update`/`delete`/`upsert` acceptent un paramètre `AccessLevel.USER_MODE` qui applique automatiquement l'enforcement CRUD/FLS sur l'opération, sans vérification manuelle champ par champ.

```apex
List<Contact> contactsToInsert = buildContacts();

try {
    Database.insert(contactsToInsert, AccessLevel.USER_MODE);
} catch (System.DmlException e) {
    // L'utilisateur n'a pas le droit de créer un ou plusieurs champs/l'objet
    // (échec CRUD/FLS en AccessLevel.USER_MODE : DmlException, pas NoAccessException,
    // qui elle est levée par Security.stripInaccessible ou en contexte Visualforce).
    throw new AuraHandledException('Vous n\'avez pas les droits nécessaires pour cette opération.');
}
```

C'est l'alternative moderne à la vérification manuelle historique :

```apex
// Approche manuelle plus ancienne, toujours valide mais plus verbeuse.
if (!Schema.sObjectType.Contact.fields.Email.isCreateable()) {
    throw new SecurityException('Champ Email non accessible en création.');
}
```

`AccessLevel.USER_MODE` doit être préféré par défaut sur tout DML exposé, y compris dans une classe `with sharing` (le sharing ne couvre pas le CRUD/FLS).

## Injection SOQL/SOSL

Toute concaténation directe d'une entrée utilisateur dans une requête dynamique est une injection potentielle.

### Vulnérable

```apex
public static List<Account> search(String userInput) {
    // DANGER : userInput peut contenir du SOQL arbitraire
    // (ex. "' OR Name != '") et modifier la logique de la requête.
    String query = 'SELECT Id, Name FROM Account WHERE Name = \'' + userInput + '\'';
    return Database.query(query);
}
```

### Corrigé (bind variable, toujours préférée)

```apex
public static List<Account> search(String userInput) {
    String query = 'SELECT Id, Name FROM Account WHERE Name = :userInput WITH USER_MODE';
    return Database.query(query);
}
```

### Corrigé (escaping, uniquement si la concaténation est réellement inévitable)

```apex
public static List<Account> searchByPattern(String userInput) {
    String safeInput = String.escapeSingleQuotes(userInput);
    // Concaténation nécessaire ici uniquement si le motif ne peut pas être
    // exprimé avec un bind (ex. construction dynamique du LIKE avec wildcard).
    String query = 'SELECT Id, Name FROM Account WHERE Name LIKE \'%' + safeInput + '%\' WITH USER_MODE';
    return Database.query(query);
}
```

Les bind variables (`:variable`) restent toujours la solution à privilégier ; `String.escapeSingleQuotes()` n'est qu'un filet de sécurité pour les cas où la structure de la requête (nom de champ, opérateur, clause dynamique) doit elle-même être construite dynamiquement et ne peut pas être bindée.

## `System.runAs()` en test

Pour valider qu'une classe respecte réellement le sharing et le CRUD/FLS selon le profil de l'utilisateur, les tests doivent exécuter le code sous l'identité d'un utilisateur de test avec `System.runAs()`, plutôt que de tester uniquement en contexte système. Voir [`12_TESTS_UNITAIRES.md`](12_TESTS_UNITAIRES.md) pour le détail du pattern (création d'utilisateur de test, permission sets, assertions sur l'accès refusé).

## Guest User / Experience Cloud

Le contexte Guest User (accès anonyme via un site Experience Cloud) est la surface d'attaque la plus sensible : l'utilisateur n'est pas authentifié et le profil Guest a historiquement des droits par défaut trop larges s'ils ne sont pas restreints. CRUD/FLS y est d'autant plus critique :

- Ne jamais exposer une classe `without sharing` à un flux accessible par un Guest User sans revue de sécurité approfondie.
- Toute méthode `@AuraEnabled` ou Apex REST accessible sans authentification doit appliquer `WITH USER_MODE`/`AccessLevel.USER_MODE` de façon systématique, sans exception.
- Vérifier explicitement les droits du profil Guest sur les objets/champs exposés plutôt que de supposer qu'ils sont restreints par défaut.

## Shield Platform Encryption (alerte)

Si des champs sont chiffrés via Shield Platform Encryption, certaines opérations SOQL sont limitées ou dégradées sur ces champs : filtrage (`WHERE`), tri (`ORDER BY`) et certaines fonctions d'agrégat peuvent ne pas fonctionner comme sur un champ en clair, selon le type de chiffrement (déterministe vs probabiliste) et l'index dédié disponible. Ce point doit être vérifié au cas par cas avant de construire une requête filtrant ou triant sur un champ potentiellement chiffré ; ce document ne couvre pas le sujet en détail.

## Anti-patterns

```apex
// Anti-pattern : pas de mode de sharing déclaré (comportement système implicite).
public class LeadService {
    public static void updateLeads(List<Lead> leads) {
        update leads; // Aucun contrôle CRUD/FLS, aucun sharing.
    }
}
```

```apex
// Anti-pattern : without sharing sans justification écrite.
public without sharing class ReportService {
    public static List<Opportunity> getAll() {
        return [SELECT Id, Amount FROM Opportunity];
    }
}
```

```apex
// Anti-pattern : injection SOQL par concaténation directe.
String query = 'SELECT Id FROM Contact WHERE LastName = \'' + lastName + '\'';
Database.query(query);
```

```apex
// Anti-pattern : DML exposé à l'UI sans enforcement CRUD/FLS.
@AuraEnabled
public static void saveContact(Contact c) {
    update c; // Pas de USER_MODE, pas de stripInaccessible.
}
```

## Checklist IA

- [ ] La classe déclare-t-elle explicitement `with sharing`, `without sharing` ou `inherited sharing` ?
- [ ] Tout `without sharing` porte-t-il un commentaire de justification métier ?
- [ ] Les requêtes SOQL exposées utilisent-elles `WITH USER_MODE` (ou à défaut `Security.stripInaccessible`) ?
- [ ] Les DML exposés utilisent-ils `AccessLevel.USER_MODE` ou une vérification CRUD/FLS explicite ?
- [ ] Aucune requête dynamique ne concatène directement une entrée utilisateur sans bind variable ?
- [ ] Si l'escaping est utilisé, est-ce uniquement parce que le bind est réellement impossible ?
- [ ] Les tests utilisent-ils `System.runAs()` pour valider le comportement multi-profil ?
- [ ] Les méthodes accessibles par un Guest User ont-elles reçu une revue de sécurité spécifique ?
- [ ] Un champ potentiellement chiffré (Shield) est-il utilisé dans un `WHERE`/`ORDER BY` sans vérification préalable ?

## Voir aussi

- [`../principes/02_CONTEXTE_EXECUTION.md`](../principes/02_CONTEXTE_EXECUTION.md)
- [`16_SOQL_SOSL_AVANCE.md`](16_SOQL_SOSL_AVANCE.md)
- [`../checklists/CHECKLIST_SECURITE_APEX.md`](../checklists/CHECKLIST_SECURITE_APEX.md)
- [`12_TESTS_UNITAIRES.md`](12_TESTS_UNITAIRES.md)
- [`18_INTEGRATION_LWC_APEX.md`](18_INTEGRATION_LWC_APEX.md)
