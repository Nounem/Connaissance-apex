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

## Pattern Selector

Centraliser tout le SOQL d’un objet dans une classe Selector dédiée. Jamais de `SELECT *`, jamais de champ implicite : lister explicitement les champs nécessaires. Utiliser `WITH USER_MODE` pour appliquer automatiquement le FLS et le partage.

```apex
public with sharing class AccountSelector {

    public List<Account> selectById(Set<Id> ids) {
        return [
            SELECT Id, Name, Industry, OwnerId, BillingCountry
            FROM Account
            WHERE Id IN :ids
            WITH USER_MODE
        ];
    }

    public List<Contact> selectByAccountIds(Set<Id> accountIds) {
        return [
            SELECT Id, LastName, Email, AccountId
            FROM Contact
            WHERE AccountId IN :accountIds
            WITH USER_MODE
        ];
    }
}
```

Le Service (ou le TriggerHandler) appelle le Selector, jamais l’inverse. Template complet : [`../templates/TEMPLATE_SELECTOR_APEX.md`](../templates/TEMPLATE_SELECTOR_APEX.md).

## Dette technique Flows/Apex

Salesforce recommande les Flows pour la logique simple, ce qui pousse une partie croissante de la logique métier hors d’Apex. Risque : une même règle métier implémentée à la fois dans un Flow et dans une classe Apex (ex. calcul de remise dupliqué), qui divergent silencieusement au premier correctif appliqué d’un seul côté.

- Centraliser la logique métier complexe ou réutilisée dans le Service layer Apex.
- Si un Flow doit déclencher cette logique, exposer un `Invocable Method` plutôt que de la réécrire en éléments Flow.
- Ne jamais dupliquer une règle de validation/calcul entre Flow et Apex : une seule source de vérité.
- Documenter dans le code (ApexDoc) quels Flows appellent la méthode invocable.

```apex
public with sharing class AccountService {

    @InvocableMethod(label='Appliquer la remise standard' category='Compte')
    public static void applyStandardDiscountFromFlow(List<Id> accountIds) {
        applyStandardDiscount(new Set<Id>(accountIds));
    }

    public static void applyStandardDiscount(Set<Id> accountIds) {
        // logique métier unique, appelée par Apex ET par les Flows
    }
}
```

Rappel de contrainte : une classe Apex ne peut contenir qu’une seule méthode `@InvocableMethod`, et son unique paramètre doit être une `List` (jamais un `Set` ni un type scalaire) — d’où la conversion `new Set<Id>(accountIds)` faite à l’intérieur de la méthode plutôt qu’en signature.

## @Deprecated

Marquer une méthode ou classe obsolète pour prévenir les appelants sans casser l’existant. La compilation continue de fonctionner. Nuance importante : le blocage réel de `@Deprecated` (interdiction de créer de *nouvelles* références tout en préservant le code existant) est un mécanisme propre aux **packages managés** — il s’applique aux abonnés externes d’un package, pas à du code interne à un org standard. Dans un org classique (sans package managé), l’annotation reste un simple signal documentaire : elle n’empêche rien à la compilation et ne garantit aucun avertissement natif ; un IDE (ex. extensions VS Code) peut néanmoins l’afficher visuellement (texte barré) grâce à sa propre analyse statique, indépendamment du compilateur Apex.

```apex
public with sharing class AccountService {

    /**
     * @deprecated Utiliser applyStandardDiscount(Set<Id>) à la place. Sera supprimé en v3.
     */
    @Deprecated
    public static void applyDiscount(List<Id> accountIds) {
        applyStandardDiscount(new Set<Id>(accountIds));
    }

    public static void applyStandardDiscount(Set<Id> accountIds) {
        // implémentation actuelle
    }
}
```

## ApexDoc

Documenter chaque classe et méthode publique avec un bloc `/** ... */` : description, paramètres, valeur de retour. Indispensable pour qu’une IA générative ou un autre développeur comprenne un contrat sans lire l’implémentation.

```apex
/**
 * @description Centralise les opérations métier liées au compte.
 */
public with sharing class AccountService {

    /**
     * @description Applique la remise standard aux comptes fournis.
     * @param accountIds Identifiants des comptes à traiter.
     * @return Nombre de comptes effectivement mis à jour.
     */
    public static Integer applyStandardDiscount(Set<Id> accountIds) {
        // ...
        return accountIds.size();
    }
}
```

## Checklist IA

- [ ] Le code sera-t-il compréhensible dans 6 mois ?
- [ ] Les responsabilités sont-elles séparées ?
- [ ] Les tests protègent-ils les comportements métier ?
- [ ] Les noms sont-ils explicites ?
- [ ] Les dépendances sont-elles visibles ?
- [ ] Le SOQL est-il centralisé dans un Selector avec `WITH USER_MODE` et champs explicites ?
- [ ] Une règle métier existe-t-elle en double dans un Flow et en Apex ?
- [ ] Les méthodes obsolètes sont-elles marquées `@Deprecated` avant suppression ?
- [ ] Les classes/méthodes publiques ont-elles un bloc ApexDoc ?

## Voir aussi

- [`07_TRIGGERS.md`](07_TRIGGERS.md)
- [`../templates/TEMPLATE_SELECTOR_APEX.md`](../templates/TEMPLATE_SELECTOR_APEX.md)
- [`../templates/TEMPLATE_SERVICE_APEX.md`](../templates/TEMPLATE_SERVICE_APEX.md)
- [`13_PACKAGING_DEPLOIEMENT.md`](13_PACKAGING_DEPLOIEMENT.md)
