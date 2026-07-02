# SOQL et SOSL avancés

## Requêtes relationnelles parent-to-child

Une sous-requête permet de récupérer les enregistrements enfants d'un parent en une seule requête, via le nom de la relation enfant (relationship name, généralement au pluriel).

```apex
List<Account> accountsWithOpps = [
    SELECT Id, Name,
        (SELECT Id, Amount, StageName FROM Opportunities WHERE IsClosed = false)
    FROM Account
    WHERE Industry = 'Energy'
];

for (Account acc : accountsWithOpps) {
    for (Opportunity opp : acc.Opportunities) {
        System.debug(opp.Amount);
    }
}
```

Limite : maximum 20 sous-requêtes enfants distinctes par requête parente.

## Requêtes relationnelles child-to-parent

La notation par points remonte vers un champ du parent sans requête séparée.

```apex
List<Contact> contacts = [
    SELECT Id, Name, Account.Name, Account.Owner.Email
    FROM Contact
    WHERE Account.Industry = 'Energy'
];
```

Limite : maximum 5 niveaux de profondeur pour une seule chaîne de relations child-to-parent (ex. `Contact.Account.Owner.Manager.Manager`), et maximum 55 relations child-to-parent distinctes cumulées sur l'ensemble d'une requête (un champ polymorphe compte pour plusieurs relations, une par type d'objet référencé). Au-delà, restructurer en plusieurs requêtes avec jointure en mémoire (voir [`06_COLLECTIONS.md`](06_COLLECTIONS.md)).

## Agrégats : GROUP BY, HAVING, fonctions

Les fonctions d'agrégation (`COUNT()`, `COUNT(champ)`, `SUM()`, `AVG()`, `MAX()`, `MIN()`) s'utilisent avec `GROUP BY` pour produire des synthèses côté base plutôt qu'en boucle Apex.

```apex
List<AggregateResult> results = [
    SELECT AccountId, COUNT(Id) nbOpps, SUM(Amount) totalAmount
    FROM Opportunity
    WHERE IsClosed = false
    GROUP BY AccountId
    HAVING SUM(Amount) > 100000
];

for (AggregateResult ar : results) {
    Id accId = (Id) ar.get('AccountId');
    Decimal total = (Decimal) ar.get('totalAmount');
}
```

Un `AggregateResult` n'est pas un SObject classique : il n'a ni `Id` stable ni possibilité de DML direct, et l'accès aux colonnes se fait par `get('alias')` (l'alias est obligatoire pour les colonnes calculées, `AccountId` reste accessible par son nom de champ). Une requête d'agrégation est plafonnée à 2000 lignes de résultats groupés ; au-delà, vérifier la documentation officielle Salesforce pour la limite en vigueur et envisager un `GROUP BY` plus sélectif ou un traitement par lots.

## Database.getQueryLocator() vs requête directe

Une requête SOQL directe dans du code synchrone est plafonnée à 50 000 lignes récupérées par transaction (limite cumulée sur toutes les requêtes). `Database.getQueryLocator()` lève cette limite jusqu'à 50 millions de lignes, mais uniquement dans le contexte `start()` d'un Batch Apex, où les enregistrements sont ensuite traités par scopes successifs.

```apex
public class RecalculAccountBatch implements Database.Batchable<SObject> {
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, AnnualRevenue
            FROM Account
            WHERE LastActivityDate < LAST_N_DAYS:365
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        for (Account acc : scope) {
            acc.AnnualRevenue = 0;
        }
        update scope;
    }

    public void finish(Database.BatchableContext bc) {
    }
}
```

En dehors d'un Batch Apex, une requête classique avec `WHERE` sélectif et éventuellement `LIMIT` reste la bonne approche ; ne pas utiliser `getQueryLocator()` comme contournement de la limite de 50 000 lignes hors contexte batch, ce n'est pas son usage.

## Injection SOQL

Une requête dynamique concaténant une entrée utilisateur sans échappement ni bind variable expose à l'injection SOQL. Voir [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md) pour le détail des règles et des exemples.

## Sélectivité des requêtes

Un `WHERE` filtre efficacement seulement s'il s'appuie sur un champ indexé (index standard sur `Id`, `Name`, `OwnerId`, champs externes, ou index custom déclaré via un ticket de support pour les champs à forte cardinalité). Sur un objet à gros volume de données (Large Data Volumes, plusieurs millions de lignes, ou Big Object), une requête non sélective déclenche un scan de table complet, avec des temps de réponse dégradés voire un `QueryException` en cas de timeout.

L'outil de diagnostic est le Query Plan (menu "Query Plan" dans l'éditeur SOQL de la Developer Console, ou endpoint `/query?explain=` de l'API REST/Tooling). Il indique le coût relatif de chaque plan d'exécution possible et le champ de sélectivité (`sObjectCardinality`, `relativeCost`) permettant de choisir le meilleur index disponible avant de figer une requête critique en production.

## SOSL vs SOQL

SOQL interroge un seul type d'objet à la fois. SOSL (`FIND ... IN ... FIELDS`) recherche un terme texte sur plusieurs objets et plusieurs champs simultanément, typiquement pour une barre de recherche globale.

```apex
List<List<SObject>> searchResults = [
    FIND 'Capitole*' IN ALL FIELDS
    RETURNING
        Account (Id, Name WHERE Industry = 'Energy'),
        Contact (Id, Name, Email),
        Opportunity (Id, Name, StageName)
];

List<Account> foundAccounts = (List<Account>) searchResults[0];
List<Contact> foundContacts = (List<Contact>) searchResults[1];
List<Opportunity> foundOpportunities = (List<Opportunity>) searchResults[2];
```

Limite : maximum 20 requêtes SOSL par transaction (contre 100 requêtes SOQL). Utiliser SOSL quand le besoin est une recherche textuelle transverse à plusieurs objets ; utiliser SOQL dès que le filtre porte sur un objet connu à l'avance avec des critères structurés (dates, montants, relations).

## FOR UPDATE

La clause `FOR UPDATE` verrouille les lignes retournées pour éviter les mises à jour concurrentes conflictuelles. Le détail (comportement, limites, cas d'usage) est traité dans [`09_CONCURRENCE_VERROUILLAGE.md`](09_CONCURRENCE_VERROUILLAGE.md) ; ne pas dupliquer ici.

## Requêtes dynamiques

`Database.query()` (et `Database.countQuery()`) construit une requête à partir d'une chaîne, utile pour des requêtes génériques dont les champs ou filtres ne sont connus qu'à l'exécution. Toujours utiliser des bind variables (`:variable`) plutôt que de la concaténation de chaînes pour les valeurs.

```apex
public class GenericQueryService {
    public static List<SObject> findByField(String objectApiName, String fieldApiName, Object value) {
        String soql = 'SELECT Id FROM ' + String.escapeSingleQuotes(objectApiName) +
                       ' WHERE ' + String.escapeSingleQuotes(fieldApiName) + ' = :value';
        return Database.query(soql);
    }
}
```

Pour construire des requêtes réellement génériques (écran de recherche paramétrable, outil d'export), s'appuyer sur `Schema.getGlobalDescribe()` pour valider dynamiquement l'existence de l'objet et du champ avant de les insérer dans la chaîne SOQL, ce qui évite à la fois les erreurs runtime et les vecteurs d'injection sur les noms d'objets/champs.

```apex
Map<String, Schema.SObjectType> globalDescribe = Schema.getGlobalDescribe();
if (!globalDescribe.containsKey(objectApiName)) {
    throw new IllegalArgumentException('Objet inconnu : ' + objectApiName);
}

Schema.DescribeSObjectResult describeResult = globalDescribe.get(objectApiName).getDescribe();
Map<String, Schema.SObjectField> fieldsMap = describeResult.fields.getMap();
if (!fieldsMap.containsKey(fieldApiName)) {
    throw new IllegalArgumentException('Champ inconnu : ' + fieldApiName);
}
```

## Anti-patterns

- Requête SOQL ou SOSL dans une boucle plutôt qu'avant celle-ci.
- Concaténer une valeur utilisateur directement dans une chaîne SOQL sans bind variable ni `escapeSingleQuotes`.
- Utiliser `Database.getQueryLocator()` hors contexte Batch Apex pour contourner la limite de 50 000 lignes.
- Ignorer un `AggregateResult` en tentant d'appeler `update` dessus comme sur un SObject standard.
- Filtrer sur un champ non indexé sur un objet à plusieurs millions de lignes sans avoir vérifié le Query Plan.
- Utiliser SOSL pour une recherche structurée sur un seul objet alors qu'une requête SOQL sélective suffirait.
- Enchaîner des relations parent au-delà de 5 niveaux au lieu de requêtes séparées jointes en mémoire.

## Règles IA

- Toujours filtrer une sous-requête enfant ou un `WHERE` sur un champ indexé quand l'objet est à fort volume.
- Toujours utiliser des bind variables (`:var`) dans les requêtes dynamiques, jamais de concaténation de valeur brute.
- Toujours valider objet et champ via `Schema.getGlobalDescribe()` avant de les insérer dans une requête dynamique construite à partir d'une entrée externe.
- Préférer un agrégat SOQL (`GROUP BY`/`SUM`/`COUNT`) à une boucle Apex qui recalcule la même chose en mémoire.
- Réserver `Database.getQueryLocator()` au `start()` d'un Batch Apex.
- Utiliser SOSL uniquement pour une recherche textuelle transverse à plusieurs objets, pas comme substitut systématique à SOQL.

## Checklist IA

- [ ] Les sous-requêtes enfants et les traversées parent restent-elles dans les limites (20 sous-requêtes, 5 niveaux) ?
- [ ] Les agrégats utilisent-ils `GROUP BY`/`HAVING` plutôt qu'une boucle Apex ?
- [ ] Le nombre de lignes agrégées attendu reste-t-il sous la limite en vigueur (vérifier la doc si volumétrie forte) ?
- [ ] `Database.getQueryLocator()` n'est-il utilisé que dans un Batch Apex ?
- [ ] Une requête dynamique utilise-t-elle des bind variables plutôt que de la concaténation ?
- [ ] Les champs/objets dynamiques sont-ils validés via `Schema.getGlobalDescribe()` avant construction du SOQL ?
- [ ] La sélectivité du `WHERE` a-t-elle été vérifiée via le Query Plan sur un objet à gros volume ?
- [ ] SOSL est-il utilisé pour une recherche multi-objets plutôt qu'à la place d'un SOQL simple ?
- [ ] Le nombre de requêtes SOSL par transaction reste-t-il sous 20 ?

## Voir aussi

- [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md)
- [`09_CONCURRENCE_VERROUILLAGE.md`](09_CONCURRENCE_VERROUILLAGE.md)
- [`06_COLLECTIONS.md`](06_COLLECTIONS.md)
- [`08_ASYNCHRONE.md`](08_ASYNCHRONE.md)
- [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md)
