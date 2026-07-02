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

### Exemple de retry réel via Queueable

Capturer explicitement `UNABLE_TO_LOCK_ROW` et renvoyer le traitement vers un Queueable qui retente après un court délai, avec un nombre d'essais borné.

```apex
public class AccountLockRetryQueueable implements Queueable {
    private List<Id> accountIds;
    private Integer attempt;
    private static final Integer MAX_ATTEMPTS = 3;

    public AccountLockRetryQueueable(List<Id> accountIds, Integer attempt) {
        this.accountIds = accountIds;
        this.attempt = attempt;
    }

    public void execute(QueueableContext ctx) {
        List<Account> accounts = [
            SELECT Id, Custom_Count__c
            FROM Account
            WHERE Id IN :accountIds
        ];
        for (Account acc : accounts) {
            acc.Custom_Count__c = (acc.Custom_Count__c == null ? 0 : acc.Custom_Count__c) + 1;
        }

        try {
            update accounts;
        } catch (DmlException e) {
            if (e.getMessage().contains('UNABLE_TO_LOCK_ROW') && attempt < MAX_ATTEMPTS) {
                // Backoff simple : le nouveau Queueable est planifié après l'exécution
                // courante, laissant le temps au verrou concurrent de se libérer.
                System.enqueueJob(new AccountLockRetryQueueable(accountIds, attempt + 1));
            } else {
                for (Integer i = 0; i < e.getNumDml(); i++) {
                    System.debug(LoggingLevel.ERROR, 'Echec DML définitif : ' + e.getDmlMessage(i));
                }
                throw e;
            }
        }
    }
}
```

Anti-pattern : boucler immédiatement en synchrone sur l'erreur (`while` avec `update` répété dans la même transaction) sans délai ni asynchronisme — cela ne fait qu'aggraver la contention car le verrou du concurrent n'a pas eu le temps de se libérer.

### Gestion des échecs partiels en masse

Pour des mises à jour concurrentes en masse, ne pas laisser un seul enregistrement verrouillé faire échouer tout le lot. Utiliser `Database.update` avec `allOrNone = false` et parcourir les `Database.SaveResult` pour isoler et journaliser les échecs individuels (typiquement pour les rejouer plus tard via un retry ciblé).

```apex
Database.DMLOptions dmlOptions = new Database.DMLOptions();
dmlOptions.optAllOrNone = false;

List<Database.SaveResult> results = Database.update(accountsToUpdate, dmlOptions);

List<Id> failedLockedIds = new List<Id>();
for (Integer i = 0; i < results.size(); i++) {
    Database.SaveResult sr = results[i];
    if (!sr.isSuccess()) {
        for (Database.Error err : sr.getErrors()) {
            System.debug(LoggingLevel.ERROR, 'Echec sur ' + accountsToUpdate[i].Id + ' : ' + err.getStatusCode() + ' - ' + err.getMessage());
            if (err.getStatusCode() == StatusCode.UNABLE_TO_LOCK_ROW) {
                failedLockedIds.add(accountsToUpdate[i].Id);
            }
        }
    }
}

if (!failedLockedIds.isEmpty() && !System.isFuture() && !System.isBatch()) {
    System.enqueueJob(new AccountLockRetryQueueable(failedLockedIds, 1));
}
```

## Data skew

Surveiller les cas où beaucoup d’enregistrements enfants pointent vers le même parent, ou lorsqu’un propriétaire possède un volume très élevé d’enregistrements. Cela peut amplifier les conflits de verrouillage et les problèmes de performance.

## Ordre d'exécution Salesforce comme cause de lock

Une part importante des `UNABLE_TO_LOCK_ROW` sur des comptes très partagés ne vient pas du code métier lui-même mais de l'ordre d'exécution Salesforce : recalcul de sharing déclenché en tâche de fond, recalculs de roll-up summary asynchrones, ou changement de `OwnerId` qui déclenche un recalcul de partage sur l'enregistrement et ses enfants. Ces traitements internes posent des verrous que le code Apex ne contrôle pas directement. Voir [`../principes/02_CONTEXTE_EXECUTION.md`](../principes/02_CONTEXTE_EXECUTION.md) pour le détail de l'ordre d'exécution.

## Bulkification par lot de parent

Pour limiter le nombre de verrous simultanés posés sur un même parent, regrouper les DML par clé de parent (par exemple par `AccountId`) plutôt que de traiter les enregistrements dans un ordre arbitraire. Traiter chaque groupe en un seul DML réduit le nombre d'accès concurrents au même verrou parent. Voir [`../principes/05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md) pour les patterns de regroupement.

## Checklist IA

- [ ] Le code met-il à jour des parents communs ?
- [ ] Les updates inutiles sont-ils évités ?
- [ ] Le risque `UNABLE_TO_LOCK_ROW` est-il identifié ?
- [ ] Un traitement asynchrone/retry est-il nécessaire ?
- [ ] Le retry est-il borné, journalisé et asynchrone (pas de boucle synchrone) ?
- [ ] Les mises à jour en masse utilisent-elles `allOrNone = false` avec parcours des `SaveResult` ?
- [ ] Les gros volumes par parent/propriétaire sont-ils pris en compte ?
- [ ] Les DML sont-ils regroupés par parent pour limiter les verrous simultanés ?
- [ ] Le rôle du recalcul de sharing/roll-up asynchrone dans les locks a-t-il été considéré ?

## Voir aussi

- [`10_CONFIGURATION.md`](10_CONFIGURATION.md)
- [`../principes/02_CONTEXTE_EXECUTION.md`](../principes/02_CONTEXTE_EXECUTION.md)
- [`../principes/05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md)
