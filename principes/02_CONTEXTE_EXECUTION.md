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

## Checklist IA

- [ ] Le code peut-il être rappelé plusieurs fois dans la même transaction ?
- [ ] La récursion est-elle contrôlée sans bloquer des cas légitimes ?
- [ ] Les traitements asynchrones sont-ils séparés du contexte synchrone ?
- [ ] Les effets des Flows et validations sont-ils pris en compte ?
