# Bulk patterns Apex

## Règle principale

Tout code Apex doit être conçu pour traiter plusieurs enregistrements. Même si l’écran ou le besoin fonctionnel semble unitaire, Salesforce peut exécuter le code en masse via import, API, Flow, Batch, Data Loader ou intégration.

## Pattern standard

1. Collecter les IDs.
2. Faire les requêtes nécessaires en une fois.
3. Construire des Maps.
4. Traiter en mémoire.
5. Faire les DML groupés.

Exemple canonique (Contact -> Account) : à réutiliser comme référence pour tout traitement bulk de relation 1-1.

```apex
public class ContactService {
    public static void updateRelatedAccounts(List<Contact> contacts) {
        // 1. Collecter les IDs sans doublon.
        Set<Id> accountIds = new Set<Id>();
        for (Contact c : contacts) {
            if (c.AccountId != null) accountIds.add(c.AccountId);
        }
        if (accountIds.isEmpty()) return;

        // 2-3. Une seule requête, résultat directement indexé en Map<Id, Account>.
        Map<Id, Account> accountsById = new Map<Id, Account>([
            SELECT Id, Description
            FROM Account
            WHERE Id IN :accountIds
        ]);

        // 4. Traitement en mémoire, aucune requête ni DML dans la boucle.
        List<Account> accountsToUpdate = new List<Account>();
        for (Account a : accountsById.values()) {
            a.Description = 'Compte lié à des contacts mis à jour';
            accountsToUpdate.add(a);
        }

        // 5. DML unique et groupé, avec garde sur liste vide.
        if (!accountsToUpdate.isEmpty()) {
            Database.update(accountsToUpdate, false);
        }
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

## Bulkification et trigger framework

La bulkification n’est efficace que si le trigger lui-même ne l’empêche pas. Un trigger par objet qui délègue immédiatement à un handler (pattern one-trigger-per-object) garantit que tous les enregistrements de `Trigger.new`/`Trigger.old` transitent par un seul point d’entrée bulk, au lieu d’être traités enregistrement par enregistrement dans plusieurs triggers dispersés. Sans ce framework, même un service bulkifié peut se retrouver appelé en boucle depuis plusieurs triggers non coordonnés.

Le détail du trigger framework (structure du trigger, handler, méthodes `beforeInsert`/`afterUpdate`, etc.) est documenté dans [`../patterns/07_TRIGGERS.md`](../patterns/07_TRIGGERS.md) et dans le squelette prêt à l’emploi [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md). Ne pas dupliquer cette structure ici : s’y référer directement.

## Pattern delta avec `Trigger.newMap` / `Trigger.oldMap`

Ne traiter que les enregistrements dont un champ précis a réellement changé, pour éviter des requêtes, DML ou callouts inutiles sur des lignes non pertinentes.

```apex
public class AccountService {
    public static void syncOnStatusChange(List<Account> newAccounts, Map<Id, Account> oldMap) {
        List<Account> changedAccounts = new List<Account>();

        for (Account acc : newAccounts) {
            Account oldAcc = oldMap.get(acc.Id);
            if (oldAcc != null && acc.Status__c != oldAcc.Status__c) {
                changedAccounts.add(acc);
            }
        }

        if (changedAccounts.isEmpty()) return;

        // Suite du traitement bulk uniquement sur les comptes réellement modifiés.
        ExternalSyncService.sync(changedAccounts);
    }
}
```

## Collections imbriquées pour relations 1-N

Pour une relation 1-1 (un Contact -> un Account), `Map<Id, Account>` suffit. Pour une relation 1-N (un Account -> plusieurs Opportunities), regrouper dans une `Map<Id, List<SObject>>`.

```apex
public class OpportunityService {
    public static Map<Id, List<Opportunity>> groupByAccountId(List<Opportunity> opportunities) {
        Map<Id, List<Opportunity>> oppsByAccountId = new Map<Id, List<Opportunity>>();

        for (Opportunity opp : opportunities) {
            if (opp.AccountId == null) continue;

            if (!oppsByAccountId.containsKey(opp.AccountId)) {
                oppsByAccountId.put(opp.AccountId, new List<Opportunity>());
            }
            oppsByAccountId.get(opp.AccountId).add(opp);
        }

        return oppsByAccountId;
    }
}
```

## Bulkification des callouts HTTP

Les callouts synchrones sont plafonnés à 100 par transaction, sans possibilité de les regrouper en une seule requête comme pour SOQL/DML. Un traitement qui déclencherait un callout par enregistrement pour une liste de plus de 100 éléments doit être repensé (agrégation côté endpoint, pagination, ou bascule en asynchrone via Queueable). Voir [`../patterns/17_CALLOUTS_INTEGRATIONS.md`](../patterns/17_CALLOUTS_INTEGRATIONS.md) pour les patterns de bulkification et d’asynchronisation des callouts.

## Risque de cascade et DML

Un trigger bulkifié individuellement peut tout de même faire dépasser la limite de 150 DML statements si plusieurs triggers s’enchaînent en cascade (trigger A met à jour un objet B, ce qui déclenche le trigger de B, qui met à jour un objet C, etc.). La bulkification réduit le nombre de lignes traitées par DML, mais ne réduit pas le nombre de DML statements déclenchés par une chaîne de triggers. Voir [`02_CONTEXTE_EXECUTION.md`](02_CONTEXTE_EXECUTION.md) pour les patterns de garde anti-récursion et de maîtrise des cascades.

## Checklist IA

- [ ] La méthode publique accepte-t-elle une collection ?
- [ ] Les données liées sont-elles préchargées en Map ?
- [ ] Le DML est-il groupé ?
- [ ] Le code fonctionne-t-il avec 0, 1 et 200 enregistrements ?
- [ ] Les doublons d’Id sont-ils évités ?
- [ ] Le traitement delta s’appuie-t-il sur `Trigger.oldMap` plutôt que de retraiter tous les enregistrements ?
- [ ] Une relation 1-N utilise-t-elle `Map<Id, List<SObject>>` plutôt que des requêtes répétées ?
- [ ] Les callouts sont-ils bulkifiés ou basculés en asynchrone au-delà de la limite ?
- [ ] Le risque de cascade de DML entre triggers a-t-il été évalué ?

## Voir aussi

- [`04_LIMITES_GOUVERNEUR.md`](04_LIMITES_GOUVERNEUR.md)
- [`../patterns/07_TRIGGERS.md`](../patterns/07_TRIGGERS.md)
- [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md)
- [`../patterns/06_COLLECTIONS.md`](../patterns/06_COLLECTIONS.md)
