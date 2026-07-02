# Triggers Apex

## Règle : un trigger par objet et par événement

Un objet ne doit avoir **qu'un seul trigger** déclaré, quel que soit le nombre d'événements (`before insert`, `after update`, etc.). C'est une règle, pas une convention de style : Salesforce ne garantit **aucun ordre d'exécution** entre plusieurs triggers actifs sur le même objet et le même événement. S'il en existe deux, leur ordre relatif peut changer d'un déploiement à l'autre, d'une org à l'autre, voire d'une exécution à l'autre — le comportement devient non déterministe et impossible à tester de façon fiable.

Le trigger unique reste un routeur minimal : aucune logique métier, uniquement l'appel au handler.

```apex
trigger AccountTrigger on Account (
    before insert, before update,
    after insert, after update
) {
    new AccountTriggerHandler().run();
}
```

Le framework handler complet (bypass, anti-récursion, dispatch avant/après pour insert/update/delete/undelete) est fourni dans [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md) — s'y référer pour le code, ne pas le réécrire ni le dupliquer dans un nouveau trigger.

## Before vs After

Utiliser `before` lorsque l'objectif est de modifier les champs du même enregistrement : cela évite un DML supplémentaire (pas d'Id requis, la modification est intégrée au DML en cours).

Utiliser `after` lorsque l'Id est nécessaire ou lorsqu'il faut agir sur des objets liés (création d'enregistrements enfants, DML sur d'autres objets, callouts asynchrones via Queueable).

## Ordre d'exécution Salesforce

Salesforce documente un ordre précis d'exécution pour un DML donné. Cet ordre évolue avec les releases (l'intégration des Flows before/after-save notamment) — **le revérifier dans la documentation officielle avant de concevoir une logique critique** qui en dépend :

1. Validation rules
2. Duplicate rules
3. Flows before-save
4. Triggers `before`
5. Validation système (types de champs, required, etc.)
6. Triggers `after`
7. Assignment rules
8. Auto-response rules
9. Workflow rules / processes (Process Builder)
10. Flows after-save
11. Recalcul des règles de partage (sharing rules)

Conséquence pour l'IA : ne jamais supposer qu'un trigger `after` voit le résultat final d'un enregistrement — des Flows after-save ou des workflow rules peuvent encore le modifier dans la même transaction, chacun pouvant redéclencher les triggers.

## Bypass et anti-récursion : le pourquoi

Un mécanisme de **bypass** (désactivation ciblée d'un handler) est nécessaire pour les scénarios où la logique métier du trigger doit être court-circuitée : migrations de données historiques, imports en masse depuis un ETL, intégrations qui écrivent déjà des données pré-calculées. Sans bypass, ces opérations déclenchent une logique conçue pour la saisie utilisateur normale, ce qui peut échouer, dupliquer des effets de bord, ou simplement ralentir inutilement un batch.

Le **guard anti-récursion doit être un compteur, jamais un simple booléen**. Un booléen bloque toute ré-entrée dès le deuxième passage, y compris quand elle est légitime (ex. : un update en cascade sur 2-3 niveaux de hiérarchie qui doit se propager). Un compteur avec seuil (`MAX_LOOP_COUNT`) laisse la récursion contrôlée se produire jusqu'à une limite raisonnable, et ne lève une exception que si ce seuil est dépassé — signe d'une vraie boucle infinie plutôt que d'une cascade normale.

Le détail d'implémentation (classe de base, `bypass()`, compteur par handler) est dans [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md).

## Outils de diagnostic contextuel

`Trigger.isExecuting` permet à une méthode partagée entre appel direct et appel trigger d'adapter son comportement (ex. : ne pas relancer un calcul déjà fait par ailleurs). `Trigger.operationType` (de type `System.TriggerOperation`) donne l'événement exact sans empiler des `if (Trigger.isBefore && Trigger.isInsert)` — utile dans du code de diagnostic ou de logging générique.

```apex
if (Trigger.operationType == TriggerOperation.BEFORE_UPDATE) {
    Logger.debug('Delta update sur ' + Trigger.new.size() + ' enregistrements');
}
```

## Piège : champs formule non recalculés en before-trigger

En `before insert`/`before update`, les champs formule (y compris les rollup summaries et les formules dépendant de champs modifiés dans le même trigger) **ne sont pas encore recalculés** : ils reflètent l'état avant le DML en cours. Ne jamais lire un champ formule en before-trigger pour prendre une décision basée sur des champs que ce même trigger vient de modifier — le résultat lu sera obsolète. Si la valeur recalculée est nécessaire, la logique doit se faire en `after` (ou être recalculée manuellement en Apex avant lecture).

## Erreurs bulk : dimensionner et cibler

Utiliser `Trigger.new.size()` pour construire des messages d'erreur qui informent sur le volume traité, et `addError()` sur le champ précis en cause plutôt que sur l'enregistrement entier, pour préserver l'expérience utilisateur en saisie unitaire comme en import bulk.

```apex
for (Account acc : (List<Account>) Trigger.new) {
    if (acc.AnnualRevenue < 0) {
        acc.AnnualRevenue.addError(
            'Le chiffre d\'affaires ne peut pas être négatif (lot de '
            + Trigger.new.size() + ' enregistrements).'
        );
    }
}
```

## Règles IA

- Un seul trigger par objet, jamais deux triggers sur le même objet même si les événements déclarés diffèrent.
- Le trigger ne contient aucune logique métier, uniquement l'instanciation du handler.
- Ne jamais réécrire le framework handler : réutiliser la classe de base de [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md).
- Ne jamais lire un champ formule en before-trigger pour une valeur que ce trigger vient de modifier.
- Ne jamais implémenter un guard anti-récursion avec un simple booléen.

## Checklist IA

- [ ] Un seul trigger existe déjà sur cet objet dans le projet (sinon fusionner) ?
- [ ] Trigger sans logique métier lourde, uniquement `new XxxTriggerHandler().run()` ?
- [ ] Service bulkifié appelé depuis le handler, pas de SOQL/DML dans les méthodes before/after ?
- [ ] `before` utilisé quand possible pour éviter un DML supplémentaire ?
- [ ] Bypass documenté pour les cas de migration/intégration en masse ?
- [ ] Récursion contrôlée par compteur, pas par booléen ?
- [ ] Aucune lecture de champ formule en before-trigger sur un champ modifié dans le même trigger ?
- [ ] `addError()` ciblé sur le champ concerné, message dimensionné avec `Trigger.new.size()` ?
- [ ] Tests couvrant insert/update/delete/undelete si applicable, y compris en bulk (200 enregistrements) ?

## Voir aussi

- [`../templates/TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md)
- [`../principes/05_BULK_PATTERNS.md`](../principes/05_BULK_PATTERNS.md)
- [`14_MAINTENANCE.md`](14_MAINTENANCE.md)
- [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md)
- [`../principes/02_CONTEXTE_EXECUTION.md`](../principes/02_CONTEXTE_EXECUTION.md)
