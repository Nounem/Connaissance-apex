# Template trigger + handler

Un trigger par objet, un handler par objet basé sur une classe de base réutilisable. La base gère le bypass et l'anti-récursion une seule fois ; chaque handler concret ne fait qu'implémenter les méthodes des événements qu'il utilise réellement.

## Piège à éviter

Le trigger déclare des événements ; le handler doit traiter **chacun** d'eux. Un trigger qui déclare `after insert, after update` sans que le handler n'ait de branche `afterInsert`/`afterUpdate` compile sans erreur mais ignore silencieusement ces événements — bug fréquent et difficile à détecter en revue rapide. Avant de livrer un trigger, vérifier que la liste des événements déclarés correspond exactement aux méthodes surchargées dans le handler.

## Classe de base `TriggerHandler`

```apex
public inherited sharing virtual class TriggerHandler {
    @TestVisible
    private static Set<String> bypassedHandlers = new Set<String>();

    // Compteur par handler ET par opération (before insert, after update, ...).
    // Une clé par opération évite qu'une transaction avec plusieurs DML
    // parfaitement légitimes (before+after d'un même DML, puis un second DML
    // sur le même objet) ne partage un seul compteur global et ne déclenche
    // une fausse alerte de récursion ; seule une vraie cascade sur le MÊME
    // événement (ex: un afterUpdate qui redéclenche un afterUpdate) est comptée.
    @TestVisible
    private static Map<String, Integer> loopCountByHandler = new Map<String, Integer>();
    private static final Integer MAX_LOOP_COUNT = 5;

    public void run() {
        String handlerName = String.valueOf(this).substringBefore(':');
        if (isBypassed(handlerName)) {
            return;
        }
        String loopKey = handlerName + '.' + Trigger.operationType;
        if (!withinLoopLimit(loopKey)) {
            throw new TriggerHandlerException(
                'Boucle de récursion détectée sur ' + loopKey
            );
        }

        if (Trigger.isBefore) {
            if (Trigger.isInsert) beforeInsert();
            if (Trigger.isUpdate) beforeUpdate();
            if (Trigger.isDelete) beforeDelete();
        } else if (Trigger.isAfter) {
            if (Trigger.isInsert) afterInsert();
            if (Trigger.isUpdate) afterUpdate();
            if (Trigger.isDelete) afterDelete();
            if (Trigger.isUndelete) afterUndelete();
        }
    }

    // À surcharger uniquement pour les événements réellement utilisés.
    protected virtual void beforeInsert() {}
    protected virtual void beforeUpdate() {}
    protected virtual void beforeDelete() {}
    protected virtual void afterInsert() {}
    protected virtual void afterUpdate() {}
    protected virtual void afterDelete() {}
    protected virtual void afterUndelete() {}

    private Boolean withinLoopLimit(String loopKey) {
        Integer count = loopCountByHandler.get(loopKey);
        count = (count == null) ? 1 : count + 1;
        loopCountByHandler.put(loopKey, count);
        return count <= MAX_LOOP_COUNT;
    }

    private Boolean isBypassed(String handlerName) {
        return bypassedHandlers.contains(handlerName);
    }

    // Utilisé pour désactiver un handler lors d'une migration de données
    // ou d'une intégration en masse : TriggerHandler.bypass('AccountTriggerHandler');
    public static void bypass(String handlerName) {
        bypassedHandlers.add(handlerName);
    }

    public static void clearBypass(String handlerName) {
        bypassedHandlers.remove(handlerName);
    }

    public class TriggerHandlerException extends Exception {}
}
```

## Trigger (mince, un seul point d'entrée)

```apex
trigger AccountTrigger on Account (
    before insert, before update,
    after insert, after update
    // Ajouter before delete / after delete / after undelete uniquement en
    // même temps que les méthodes correspondantes dans le handler ci-dessous.
) {
    new AccountTriggerHandler().run();
}
```

## Handler concret

```apex
public inherited sharing class AccountTriggerHandler extends TriggerHandler {
    protected override void beforeInsert() {
        AccountService.applyDefaults((List<Account>) Trigger.new);
    }

    protected override void beforeUpdate() {
        AccountService.validateOwnershipChange(
            (List<Account>) Trigger.new,
            (Map<Id, Account>) Trigger.oldMap
        );
    }

    protected override void afterInsert() {
        AccountService.createRelatedRecords((List<Account>) Trigger.new);
    }

    protected override void afterUpdate() {
        // Ne traiter que les comptes dont le champ pertinent a changé
        // (delta pattern) pour éviter du travail bulk inutile.
        Map<Id, Account> oldMap = (Map<Id, Account>) Trigger.oldMap;
        List<Account> changedAccounts = new List<Account>();
        for (Account acc : (List<Account>) Trigger.new) {
            if (acc.OwnerId != oldMap.get(acc.Id).OwnerId) {
                changedAccounts.add(acc);
            }
        }
        if (!changedAccounts.isEmpty()) {
            AccountService.notifyOwnerChange(changedAccounts);
        }
    }
}
```

## Checklist IA

- [ ] Le trigger ne contient aucune logique métier, uniquement l'instanciation du handler.
- [ ] Chaque événement déclaré dans le trigger a une méthode correspondante surchargée dans le handler (sinon le supprimer de la déclaration du trigger).
- [ ] Le handler délègue à un service bulkifié, jamais de SOQL/DML directement dans les méthodes `before*`/`after*`.
- [ ] Le pattern delta (`Trigger.oldMap`) est utilisé pour ne traiter que les enregistrements réellement modifiés sur le champ pertinent.
- [ ] Un mécanisme de bypass existe et est documenté pour les migrations de données.
- [ ] Le guard anti-récursion est basé sur un compteur, pas un simple booléen (permet une récursion contrôlée légitime jusqu'à N niveaux).

## Voir aussi

- [`05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md) — pattern delta et collections 1-N.
- [`07_TRIGGERS.md`](../patterns/07_TRIGGERS.md) — architecture trigger et ordre d'exécution Salesforce.
- [`02_CONTEXTE_EXECUTION.md`](../principes/02_CONTEXTE_EXECUTION.md) — récursion et contexte transactionnel.
- [`TEMPLATE_SERVICE_APEX.md`](TEMPLATE_SERVICE_APEX.md) — structure du service appelé par le handler.
