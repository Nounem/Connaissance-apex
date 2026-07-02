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
