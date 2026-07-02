# Limites gouverneur Apex

## Principe

Les limites gouverneur sont une contrainte d’architecture. Elles obligent à optimiser l’accès aux données, les DML, le CPU, le heap et les traitements asynchrones.

## Règles IA

- Concevoir pour consommer moins que les limites, pas juste pour rester sous le maximum.
- Supposer que d’autres traitements partagent la même transaction.
- Éviter les opérations inutiles, même si elles semblent petites.
- Déplacer les traitements lourds vers Queueable, Batch ou Platform Event si nécessaire.

## SOQL

Bonnes pratiques :

- Une requête par objet/contexte métier lorsque possible.
- Utiliser `Set<Id>` pour filtrer.
- Charger uniquement les champs nécessaires.
- Regrouper les requêtes liées.
- Penser à la sélectivité pour les grands volumes.

```apex
Set<Id> accountIds = new Set<Id>();
for (Contact c : contacts) {
    if (c.AccountId != null) accountIds.add(c.AccountId);
}

Map<Id, Account> accountsById = new Map<Id, Account>([
    SELECT Id, Name, OwnerId
    FROM Account
    WHERE Id IN :accountIds
]);
```

## DML

- Construire une liste `recordsToUpdate`.
- Ne faire le DML que si la liste n’est pas vide.
- Utiliser `Database.update(records, false)` si une réussite partielle est acceptable.

```apex
if (!accountsToUpdate.isEmpty()) {
    Database.SaveResult[] results = Database.update(accountsToUpdate, false);
}
```

## CPU

Réduire :

- boucles imbriquées ;
- accès dynamiques inutiles ;
- sérialisations JSON répétées ;
- recalculs évitables ;
- traitements synchrones lourds.

## Heap

- Ne pas cacher de gros volumes dans des variables statiques.
- Traiter en batch si le volume est élevé.
- Ne pas charger des champs inutiles.

## Tableau des limites gouverneur principales

| Limite | Synchrone | Asynchrone |
|---|---|---|
| Requêtes SOQL | 100 | 200 |
| Lignes récupérées (total, toutes requêtes SOQL) | 50 000 | 50 000 |
| Requêtes SOSL | 20 | 20 |
| DML statements | 150 | 150 |
| Lignes traitées par DML | 10 000 | 10 000 |
| CPU time | 10 000 ms (10 s) | 60 000 ms (60 s) |
| Heap size | 6 MB | 12 MB |
| Callouts HTTP/webservice | 100 | 100 |
| Taille max de la requête OU de la réponse d’un callout (chacune, pas la somme) | 6 MB | 12 MB |
| Emails envoyés | 10 par transaction | 10 par transaction |
| Jobs Queueable mis en file (`System.enqueueJob`) | 50 par transaction Apex | 50 par transaction Apex |
| Jobs Batch Apex actifs/en file simultanément | 5 | 5 |

La limite de 50 jobs Queueable s’applique de la même façon en synchrone et en asynchrone ; elle ne doit pas être confondue avec la règle de **chaînage** : depuis l’`execute()` d’un Queueable déjà en cours, un seul niveau d’imbrication (un seul nouvel `enqueueJob`) est autorisé, sauf en contexte de test (`Test.startTest()`/`stopTest()`) où plusieurs jobs peuvent être ajoutés. Par ailleurs, au-delà des 5 jobs Batch actifs/en file, des jobs supplémentaires peuvent être soumis et patientent dans l’Apex Flex Queue (jusqu’à 100 jobs en attente), mais seuls 5 s’exécutent réellement en parallèle.

Ces valeurs évoluent selon les releases Salesforce (et parfois selon le type d’org). L’IA ne doit jamais les considérer comme figées ni les coder en dur dans un raisonnement métier : elle doit vérifier la documentation officielle Salesforce (Apex Governor Limits) ou interroger la classe `Limits` à l’exécution pour connaître la limite réelle applicable.

## Classe `Limits`

La classe `Limits` expose la consommation courante et le plafond de la transaction en cours. L’IA doit l’utiliser pour écrire du code défensif avant un traitement coûteux, plutôt que de deviner une marge.

Méthodes principales :

- `Limits.getQueries()` / `Limits.getLimitQueries()`
- `Limits.getDmlStatements()` / `Limits.getLimitDmlStatements()`
- `Limits.getCpuTime()` / `Limits.getLimitCpuTime()`
- `Limits.getHeapSize()` / `Limits.getLimitHeapSize()`
- `Limits.getQueryRows()` / `Limits.getLimitQueryRows()`

```apex
public class BatchSafeProcessor {
    public static void processIfRoomAvailable(List<Account> accounts) {
        Integer queriesLeft = Limits.getLimitQueries() - Limits.getQueries();
        Integer dmlLeft = Limits.getLimitDmlStatements() - Limits.getDmlStatements();

        if (queriesLeft < 5 || dmlLeft < 5) {
            // Marge insuffisante : reporter en asynchrone plutôt que risquer une LimitException.
            System.enqueueJob(new AccountProcessorQueueable(accounts));
            return;
        }

        AccountService.processAccounts(accounts);
    }
}
```

## Batch Apex : limites spécifiques

- `scope` maximum du `start()` : 2000 enregistrements par exécution d’`execute()` ; valeur par défaut 200 si non précisée dans `Database.executeBatch(batchInstance, scope)`.
- Maximum 5 jobs Batch Apex actifs ou en file d’attente simultanément par org.
- Chaque `execute()` est une transaction indépendante : ses limites SOQL/DML/CPU repartent à zéro à chaque scope.

## Taille de code et récursion

- Une classe ou un trigger Apex ne peut pas dépasser environ 1 000 000 caractères de code source (≈ 1 MB, commentaires et lignes vides inclus) ; au-delà, une extension exceptionnelle peut être demandée au support Salesforce, mais la bonne pratique reste de découper en classes plus petites.
- Une récursion incontrôlée (ex. trigger qui redéclenche le même trigger sans garde) fait grimper silencieusement la consommation SOQL/DML/CPU et lève une `System.LimitException` dès qu’un plafond est atteint.
- Mettre en place une garde anti-récursion (variable statique de tracking) dès qu’un trigger peut provoquer un DML sur le même objet ou un objet lié en cascade.

## Checklist IA

- [ ] SOQL hors boucle ?
- [ ] DML hors boucle ?
- [ ] Requêtes sélectives ?
- [ ] Champs limités au nécessaire ?
- [ ] Pas de boucle O(n²) évitable ?
- [ ] Traitement asynchrone envisagé si CPU/heap élevé ?
- [ ] Marge de limites vérifiée via `Limits` avant un traitement coûteux ?
- [ ] Scope Batch et jobs concurrents pris en compte ?
- [ ] Garde anti-récursion en place si trigger en cascade ?

## Voir aussi

- [`05_BULK_PATTERNS.md`](05_BULK_PATTERNS.md)
- [`../patterns/08_ASYNCHRONE.md`](../patterns/08_ASYNCHRONE.md)
- [`../patterns/16_SOQL_SOSL_AVANCE.md`](../patterns/16_SOQL_SOSL_AVANCE.md)
