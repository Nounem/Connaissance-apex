# Intégration Lightning Web Components et Apex

## Rôle de l'intégration

Un composant LWC ne peut appeler Apex que via des méthodes explicitement exposées. L'IA doit traiter toute méthode `@AuraEnabled` comme une surface publique : elle doit être sécurisée, bulkifiée et renvoyer des données simples à sérialiser.

## @AuraEnabled

Annotation obligatoire pour rendre une méthode Apex visible depuis un composant LWC ou Aura. Sans elle, la méthode est invisible côté client, même si elle est `public` ou `global`.

- `@AuraEnabled(cacheable=true)` : pour les méthodes de lecture seule (aucun DML). Autorise l'utilisation avec `@wire` et permet la mise en cache côté client (Lightning Data Service).
- `@AuraEnabled` sans `cacheable` : pour les actions qui modifient des données (insert/update/delete). Ne doit jamais être utilisé avec `@wire` ; réservé à l'appel impératif.

```apex
public with sharing class AccountController {
    @AuraEnabled(cacheable=true)
    public static List<Account> getAccounts(String searchTerm) {
        return [
            SELECT Id, Name, Industry
            FROM Account
            WHERE Name LIKE :('%' + searchTerm + '%')
            WITH USER_MODE
            LIMIT 50
        ];
    }

    @AuraEnabled
    public static void updateAccountIndustry(Id accountId, String industry) {
        Account acc = new Account(Id = accountId, Industry = industry);
        update acc;
    }
}
```

Une méthode `cacheable=true` doit être `static` et ne doit contenir aucune opération DML.

## @wire vs appel impératif

`@wire` charge les données automatiquement dès l'insertion du composant et relance l'appel Apex si un paramètre réactif (préfixé `$`) change. À réserver aux méthodes `cacheable=true` de lecture. L'appel impératif (import direct + `then`/`await`) convient aux actions déclenchées par l'utilisateur (clic, soumission de formulaire) ou aux méthodes qui font du DML.

```apex
public with sharing class AccountController {
    @AuraEnabled(cacheable=true)
    public static List<Account> getAccountsByIndustry(String industry) {
        return [SELECT Id, Name FROM Account WHERE Industry = :industry WITH USER_MODE];
    }
}
```

```javascript
// Chargement réactif avec @wire
import { LightningElement, api, wire } from 'lwc';
import getAccountsByIndustry from '@salesforce/apex/AccountController.getAccountsByIndustry';

export default class AccountList extends LightningElement {
    @api industry;

    @wire(getAccountsByIndustry, { industry: '$industry' })
    accounts;
}
```

```javascript
// Appel impératif déclenché par un clic
import { LightningElement } from 'lwc';
import updateAccountIndustry from '@salesforce/apex/AccountController.updateAccountIndustry';

export default class AccountEditor extends LightningElement {
    recordId;
    industry;

    handleSave() {
        updateAccountIndustry({ accountId: this.recordId, industry: this.industry })
            .then(() => { /* succès */ })
            .catch((error) => { /* voir gestion d'erreurs */ });
    }
}
```

## Sérialisation

Les valeurs retournées à un LWC sont sérialisées en JSON. L'IA doit :

- Retourner des types primitifs, des `List`/`Map` de types primitifs, des sObjects ou des classes Apex avec des champs `public`/`@AuraEnabled` explicites.
- Éviter les références circulaires entre objets (ex : deux classes qui se pointent mutuellement), qui font échouer la sérialisation.
- Surveiller la taille du payload : éviter de renvoyer des sObjects avec des champs inutiles ou des relations imbriquées volumineuses (préférer une requête SOQL ciblée sur les champs nécessaires plutôt que `SELECT *`-like).
- Privilégier une wrapper class dédiée pour agréger plusieurs sources de données plutôt que renvoyer des structures Apex complexes non prévues pour l'exposition (voir [`06_COLLECTIONS.md`](06_COLLECTIONS.md), section wrapper class pour l'UI).

## Gestion d'erreurs côté LWC

Une exception Apex standard (`DmlException`, `QueryException`, etc.) qui remonte jusqu'au client voit son message masqué pour des raisons de sécurité ; le LWC ne reçoit qu'un message générique. Pour transmettre un message utile, l'IA doit intercepter l'exception et lever une `AuraHandledException` avec un message explicite.

```apex
@AuraEnabled
public static void createOrder(Id accountId) {
    try {
        Order ord = new Order(AccountId = accountId, Status = 'Draft', EffectiveDate = Date.today());
        insert ord;
    } catch (DmlException e) {
        AuraHandledException ex = new AuraHandledException(
            'Impossible de créer la commande : ' + e.getDmlMessage(0)
        );
        ex.setMessage('Impossible de créer la commande : ' + e.getDmlMessage(0));
        throw ex;
    }
}
```

Note : `AuraHandledException` réinitialise son propre message via `getMessage()` uniquement après un appel explicite à `setMessage()` (le constructeur seul ne suffit pas toujours selon le contexte) ; l'IA doit systématiquement appeler `setMessage()` avant de lever l'exception.

```javascript
import createOrder from '@salesforce/apex/OrderController.createOrder';

handleCreateOrder() {
    createOrder({ accountId: this.recordId })
        .then(() => {
            // succès : afficher un toast
        })
        .catch((error) => {
            const message = error.body?.message || 'Erreur inconnue';
            // afficher le message dans un toast ou un message d'erreur inline
        });
}
```

## CRUD/FLS

Toute méthode `@AuraEnabled` est une porte d'entrée potentiellement accessible par n'importe quel utilisateur ayant accès au composant qui l'appelle, indépendamment des contrôles d'affichage côté UI. L'IA doit donc appliquer strictement les vérifications CRUD/FLS et le partage (`with sharing`) sur ces méthodes, exactement comme pour tout point d'entrée public. Voir [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md) pour le détail des mécanismes (`WITH USER_MODE`, `Security.stripInaccessible`, `with sharing`).

## Bulkification côté LWC

L'IA ne doit jamais appeler une méthode Apex à l'intérieur d'une boucle JavaScript (`forEach`, `for`, `map`) sur une liste d'éléments : cela génère un appel serveur par élément, multiplie la latence et peut consommer les limites de gouverneur côté serveur. Il faut toujours concevoir la méthode Apex pour accepter une collection en paramètre et traiter l'ensemble en un seul appel.

```apex
@AuraEnabled
public static void updateOpportunityStages(List<Id> opportunityIds, String newStage) {
    List<Opportunity> oppsToUpdate = new List<Opportunity>();
    for (Id oppId : opportunityIds) {
        oppsToUpdate.add(new Opportunity(Id = oppId, StageName = newStage));
    }
    update oppsToUpdate;
}
```

```javascript
// Un seul appel pour toute la sélection, pas un appel par ligne
updateOpportunityStages({ opportunityIds: this.selectedIds, newStage: 'Closed Won' })
    .then(() => refreshApex(this.wiredOpportunities));
```

## Refresh du cache

Après une action impérative qui modifie des données lues par un `@wire`, l'IA doit appeler `refreshApex()` sur la propriété wired (en conservant sa référence complète, pas seulement `.data`) pour invalider le cache et relancer la méthode Apex.

```javascript
import { refreshApex } from '@salesforce/apex';

export default class OpportunityList extends LightningElement {
    wiredOpportunitiesResult;

    @wire(getOpenOpportunities)
    wiredOpportunities(result) {
        this.wiredOpportunitiesResult = result;
    }

    handleAfterUpdate() {
        return refreshApex(this.wiredOpportunitiesResult);
    }
}
```

## Anti-patterns

- Exposer une méthode qui fait du DML avec `cacheable=true`.
- Appeler une méthode Apex dans une boucle JavaScript au lieu de passer une collection.
- Laisser remonter une exception Apex standard sans la convertir en `AuraHandledException`.
- Retourner des sObjects avec des champs ou relations non nécessaires au composant.
- Oublier `with sharing` ou les contrôles FLS sur une méthode `@AuraEnabled`.
- Modifier directement `wiredProperty.data` au lieu d'utiliser `refreshApex()`.

## Règles IA

- Toujours annoter les méthodes exposées à un LWC avec `@AuraEnabled`, en choisissant `cacheable=true` uniquement pour la lecture pure.
- Toujours utiliser `@wire` pour un chargement automatique et réactif, et l'appel impératif pour toute action déclenchée par l'utilisateur ou toute méthode avec DML.
- Toujours retourner des types simples, des sObjects ciblés ou des wrapper classes dédiées, sans référence circulaire.
- Toujours convertir les exceptions internes en `AuraHandledException` avec un message utile avant de les laisser remonter au client.
- Toujours appliquer CRUD/FLS et `with sharing` sur les méthodes `@AuraEnabled`.
- Toujours bulkifier les appels Apex depuis le JavaScript : une collection en un seul appel, jamais un appel par élément.
- Toujours utiliser `refreshApex()` pour resynchroniser un `@wire` après une modification impérative.

## Checklist IA

- [ ] La méthode est-elle annotée `@AuraEnabled`, avec `cacheable=true` seulement si elle est en lecture seule ?
- [ ] Le choix entre `@wire` et appel impératif correspond-il à la nature de l'action (lecture réactive vs action utilisateur/DML) ?
- [ ] Les données retournées sont-elles sérialisables simplement (pas de référence circulaire, payload raisonnable) ?
- [ ] Les exceptions sont-elles converties en `AuraHandledException` avec un message exploitable côté LWC ?
- [ ] CRUD/FLS et `with sharing` sont-ils respectés sur la méthode exposée ?
- [ ] Le JavaScript évite-t-il tout appel Apex en boucle au profit d'un appel unique avec une collection ?
- [ ] `refreshApex()` est-il appelé après une modification impactant un `@wire` ?

## Voir aussi

- [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md)
- [`06_COLLECTIONS.md`](06_COLLECTIONS.md)
- [`../templates/TEMPLATE_SERVICE_APEX.md`](../templates/TEMPLATE_SERVICE_APEX.md)
- [`11_DEBUG_DIAGNOSTICS.md`](11_DEBUG_DIAGNOSTICS.md)
