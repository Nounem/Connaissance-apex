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
