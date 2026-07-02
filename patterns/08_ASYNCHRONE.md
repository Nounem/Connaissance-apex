# Apex asynchrone

## Quand utiliser l'asynchrone ?

Utiliser l'asynchrone lorsque le traitement :

- consomme trop de CPU (limite synchrone plus basse que l'asynchrone) ;
- effectue des callouts (interdits après DML dans un contexte synchrone sans `future`) ;
- traite un volume important (au-delà de ce qu'une transaction synchrone peut absorber) ;
- peut être différé sans dégrader l'expérience utilisateur ;
- doit être isolé du contexte transactionnel courant (rollback partiel, retry indépendant).

Chaque mécanisme asynchrone a ses propres limites de gouverneur (heap, SOQL, DML, jobs en file). Ne pas les redéfinir ici : voir [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md) pour les chiffres précis (nombre de jobs Queueable chaînables, 5 jobs Batch concurrents par org, quota quotidien de jobs asynchrones, etc.). Ces chiffres évoluent avec les releases : ne jamais les coder en dur dans la logique métier ni les dupliquer dans plusieurs fichiers.

## Tableau de décision : quel mécanisme choisir ?

| Besoin | Mécanisme recommandé | Pourquoi |
|---|---|---|
| Callout simple, éventuellement chaîné | Queueable (`Database.AllowsCallouts`) | Léger, testable, paramétrable via constructeur, chaînage natif |
| Traitement différé simple sans callout | Queueable | Remplace @future dans presque tous les cas |
| Gros volume (> quelques milliers d'enregistrements) | Batch Apex | Découpage automatique en scopes, reprise possible, `Database.Stateful` pour agréger |
| Récurrence planifiée (nocturne, horaire, etc.) | Schedulable + `System.schedule()` | Cron natif, déclenche typiquement un Batch |
| Découplage / diffusion à plusieurs abonnés (pub-sub) | Platform Events | Émetteur et récepteurs indépendants, rejouable, traversant les org via événements externes |
| Callout long depuis Visualforce/Aura sans bloquer le thread synchrone | Continuation | Libère le serveur pendant l'attente de la réponse externe |
| Compatibilité avec du code existant ou cas trivial sans retour ni objet complexe | @future | Legacy uniquement — voir limitations ci-dessous |

Anti-pattern : choisir @future par habitude alors qu'un Queueable ferait la même chose avec moins de limitations. Anti-pattern : lancer un Batch pour quelques dizaines d'enregistrements — le coût de démarrage (scope, requêtes de comptage) n'est pas justifié, un Queueable suffit.

## Queueable — le mécanisme par défaut

```apex
public class AccountRecalculationJob implements Queueable, Database.AllowsCallouts {
    private Set<Id> accountIds;

    public AccountRecalculationJob(Set<Id> accountIds) {
        this.accountIds = accountIds == null ? new Set<Id>() : new Set<Id>(accountIds);
    }

    public void execute(QueueableContext context) {
        if (accountIds.isEmpty()) return;
        AccountService.recalculate(accountIds);
    }
}
```

`Database.AllowsCallouts` n'est nécessaire que si `execute()` effectue un callout HTTP. Sans elle, tout `HttpRequest` lève une exception.

### Chaînage de Queueable

`System.enqueueJob()` peut être appelé depuis l'intérieur d'un `execute()` pour enchaîner un job suivant. La règle de chaînage est simple : un job Queueable déjà en cours d'exécution ne peut ajouter qu'un seul job enfant — un second appel à `System.enqueueJob()` dans le même `execute()` échoue. Cela n'empêche pas une chaîne longue (le job A ajoute B, B ajoute C, C ajoute D, etc.) : en production et en sandbox, aucun plafond de profondeur n'est imposé à ce type de chaîne séquentielle, tant que chaque job respecte ses propres limites ; seules les org Developer Edition et les org d'essai plafonnent la profondeur totale de la chaîne à 5 jobs (job initial compris). Cette règle de chaînage est distincte de la limite de jobs Queueable ajoutables en une seule fois depuis un contexte synchrone (Trigger, contrôleur) — voir [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md). En test, le comportement est en réalité plus restrictif qu'en production, pas plus permissif : `Test.stopTest()` exécute de façon synchrone le job initial, puis un éventuel job qu'il chaîne (un seul niveau) — mais un troisième niveau (petit-enfant) ne s'exécute pas, sans faire échouer le test pour autant. Ne pas supposer que `Test.stopTest()` déroule une chaîne de plus de deux jobs.

Pattern « auto-chaînage par lots » pour traiter un gros volume sans Batch (par exemple quand l'ordre de traitement ou un état fin doit être maîtrisé) :

```apex
public class BulkProcessorJob implements Queueable {
    private List<Id> remainingIds;
    private static final Integer CHUNK_SIZE = 200;

    public BulkProcessorJob(List<Id> remainingIds) {
        this.remainingIds = remainingIds;
    }

    public void execute(QueueableContext context) {
        Integer size = Math.min(CHUNK_SIZE, remainingIds.size());
        List<Id> currentChunk = new List<Id>();
        for (Integer i = 0; i < size; i++) {
            currentChunk.add(remainingIds.remove(0));
        }

        AccountService.recalculate(new Set<Id>(currentChunk));

        if (!remainingIds.isEmpty() && !Test.isRunningTest()) {
            System.enqueueJob(new BulkProcessorJob(remainingIds));
        }
    }
}
```

Anti-pattern : enchaîner sans condition d'arrêt claire (liste qui ne diminue jamais, boucle sur une erreur systématique). Toujours borner explicitement le nombre d'itérations ou la taille restante.

### Finalizer — la bonne pratique moderne de robustesse

Un `Finalizer` s'attache à un Queueable via `System.attachFinalizer()` et s'exécute systématiquement après `execute()`, que le job ait réussi, échoué ou été annulé (`ParentJobResult.SUCCESS` / `UNHANDLED_EXCEPTION`). C'est le mécanisme recommandé pour garantir un traitement de fin (log, notification, retry) sans dépendre d'un `try/catch` interne qui pourrait lui-même échouer silencieusement.

```apex
public class AccountRecalculationJob implements Queueable, Database.AllowsCallouts {
    private Set<Id> accountIds;

    public AccountRecalculationJob(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext context) {
        System.attachFinalizer(new RecalculationFinalizer(accountIds));
        AccountService.recalculate(accountIds);
    }
}

public class RecalculationFinalizer implements Finalizer {
    private Set<Id> accountIds;

    public RecalculationFinalizer(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(FinalizerContext context) {
        if (context.getResult() == ParentJobResult.UNHANDLED_EXCEPTION) {
            ErrorLogger.log('AccountRecalculationJob', context.getException(), accountIds);
            // Retry contrôlé, borné, avec un nouveau job dédié :
            System.enqueueJob(new AccountRecalculationRetryJob(accountIds, context.getAsyncApexJobId()));
        }
    }
}
```

Un Finalizer ne peut être attaché qu'une seule fois par Queueable et ne peut pas être attaché depuis un Batch ou un Schedulable. Il s'exécute lui-même comme un mini-Queueable avec ses propres limites.

## Batch Apex

Utiliser `Database.Batchable<SObject>` pour tout traitement portant sur un grand nombre d'enregistrements, découpé automatiquement en scopes.

```apex
public class ContactCleanupBatch implements Database.Batchable<SObject>, Database.Stateful {
    private Integer totalProcessed = 0;

    public Database.QueryLocator start(Database.BatchableContext context) {
        return Database.getQueryLocator(
            'SELECT Id, Email, AccountId FROM Contact WHERE Email = null'
        );
    }

    public void execute(Database.BatchableContext context, List<SObject> scope) {
        List<Contact> contacts = (List<Contact>) scope;
        for (Contact c : contacts) {
            c.Email = ContactService.buildFallbackEmail(c);
        }
        Database.update(contacts, false);
        totalProcessed += contacts.size();
    }

    public void finish(Database.BatchableContext context) {
        EmailNotifier.notifyAdmins('ContactCleanupBatch terminé : ' + totalProcessed + ' contacts traités.');

        // Chaînage batch -> batch pour une étape suivante :
        Database.executeBatch(new ContactDuplicateMergeBatch(), 200);
    }
}
```

Points clés :

- `start()` retourne soit un `Database.QueryLocator` (jusqu'à 50 millions d'enregistrements, requête SOQL simple sans sous-requête complexe), soit un `Iterable<SObject>` pour construire la source en mémoire (liste custom, résultat de callout, agrégation) — limité par le heap disponible.
- `Database.Stateful` rend les variables d'instance persistantes d'un scope à l'autre (sinon chaque `execute()` repart d'une instance fraîche). Indispensable pour un compteur total, un log agrégé, une liste d'erreurs collectées.
- `Database.AllowsCallouts` doit être ajouté à l'interface si `execute()` effectue des callouts HTTP.
- La taille de scope par défaut est 200, personnalisable jusqu'à 2000 via le second paramètre de `Database.executeBatch(batchInstance, scopeSize)`. Un scope plus petit réduit le risque d'atteindre les limites SOQL/DML par exécution ; un scope plus grand réduit le nombre total d'exécutions.
- `finish()` est l'endroit correct pour chaîner un batch suivant (`Database.executeBatch()`) ou envoyer une notification de fin.

Anti-pattern : lancer un Batch depuis un Trigger sans garde-fou — un Trigger déclenché en masse peut tenter de lancer plusieurs Batch en parallèle et heurter la limite de 5 jobs Batch concurrents par org.

## Schedulable

Utiliser `Schedulable` pour déclencher un traitement récurrent, typiquement un Batch.

```apex
public class NightlyCleanupScheduler implements Schedulable {
    public void execute(SchedulableContext context) {
        Database.executeBatch(new ContactCleanupBatch(), 200);
    }
}
```

Planification via une expression cron (`Sec Min Heure JourMois Mois JourSemaine Année`) :

```apex
String cronExpression = '0 0 2 * * ?'; // tous les jours à 2h00
System.schedule('Nightly Contact Cleanup', cronExpression, new NightlyCleanupScheduler());
```

Un job planifié occupe un flux de la limite de jobs planifiés concurrents ; éviter de reprogrammer un nouveau schedule à chaque exécution sauf besoin explicite (cron auto-régénéré).

## @future — legacy, à éviter dans le code nouveau

`@future` reste présent dans beaucoup de bases historiques mais cumule des limitations que Queueable n'a pas :

| Contrainte | @future | Queueable |
|---|---|---|
| Paramètres acceptés | Primitifs et collections de primitifs uniquement (pas de SObject, pas d'objet custom) | N'importe quel type sérialisable via le constructeur, y compris SObject |
| Valeur de retour | Aucune (`void` obligatoire) | Aucune (même contrainte) |
| Chaînage | Impossible d'appeler une autre méthode `@future` depuis une méthode `@future` | Chaînage natif via `System.enqueueJob()` |
| Ordre d'exécution | Non garanti, même entre plusieurs appels de la même transaction | Non garanti également, mais suivi possible via `AsyncApexJob` et Finalizer |
| Suivi du job | Pas d'Id de job accessible facilement | `System.enqueueJob()` retourne un `Id` exploitable |
| Limite par transaction | 50 appels `@future` maximum | Limite de jobs en file, cf. limites de gouverneur |

```apex
public class LegacyNotifier {
    @future(callout=true)
    public static void notifyExternalSystem(Set<Id> accountIds) {
        // accountIds : uniquement des primitifs/collections, jamais de SObject en paramètre
        for (Id accId : accountIds) {
            // callout HTTP...
        }
    }
}
```

Conclusion : ne choisir @future que pour maintenir du code existant ou dans un cas trivial isolé sans besoin de chaînage ni de paramètre complexe. Dans tous les autres cas, préférer Queueable, qui couvre les mêmes usages avec moins de contraintes et une meilleure testabilité.

## Platform Events

Les Platform Events permettent de publier un événement custom et de laisser un ou plusieurs abonnés réagir de façon découplée, y compris depuis d'autres org ou des systèmes externes (via CometD/API).

Définition d'un objet événement (`Order_Shipped__e`) avec ses champs, puis publication :

```apex
public class OrderShippingService {
    public static void notifyShipped(Id orderId, String trackingNumber) {
        Order_Shipped__e event = new Order_Shipped__e(
            Order_Id__c = orderId,
            Tracking_Number__c = trackingNumber
        );
        Database.SaveResult result = EventBus.publish(event);
        if (!result.isSuccess()) {
            for (Database.Error err : result.getErrors()) {
                ErrorLogger.log('OrderShippingService.notifyShipped', err.getMessage());
            }
        }
    }
}
```

Trigger d'écoute sur l'événement (obligatoirement `after insert`) :

```apex
trigger OrderShippedEventTrigger on Order_Shipped__e (after insert) {
    for (Order_Shipped__e evt : Trigger.new) {
        ShippingNotificationHandler.handle(evt.Order_Id__c, evt.Tracking_Number__c);
    }
}
```

Différence avec Change Data Capture (CDC) : CDC capture automatiquement les changements natifs sur des objets standard/custom (insert/update/delete/undelete) sans code de publication — il suffit d'activer le canal `<ObjectName>ChangeEvent`. Les Platform Events sont custom, publiés explicitement par du code métier pour véhiculer un événement fonctionnel (pas forcément lié à un DML), et ne sont pas automatiquement synchronisés avec l'état des enregistrements.

Anti-pattern : transformer un simple appel de méthode interne en Platform Event « au cas où » — n'introduire cette architecture que lorsqu'un découplage réel entre émetteur et récepteur(s) est nécessaire.

## Continuation

La `Continuation` permet de déclencher un callout HTTP long depuis un contrôleur Visualforce ou un composant Aura sans bloquer de ressource de traitement Apex synchrone pendant l'attente de la réponse : le serveur libère le thread pendant l'appel externe puis reprend l'exécution via une méthode de callback une fois la réponse reçue. Réservé aux contrôleurs Visualforce et composants Aura : la classe `Continuation` n'est pas prise en charge en Lightning Web Components (aucun mécanisme équivalent officiel côté LWC) ; pour un traitement asynchrone backend classique, ou depuis un LWC, préférer un Queueable appelé de façon imperative, éventuellement combiné à du polling côté client ou à un Platform Event.

## Limites de gouverneur asynchrones

Ne pas dupliquer les chiffres ici : se référer systématiquement à [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md) pour le nombre maximal de jobs en file, le nombre de jobs Batch concurrents par org (5), et le quota quotidien de jobs asynchrones (Batch + Future + Queueable + Scheduled confondus). Concevoir le code pour rester très en-deçà de ces plafonds plutôt que de les approcher.

## Attention aux appels asynchrones multiples

Regrouper les demandes dans un seul job quand possible. Ne pas créer un job par enregistrement : un Trigger qui boucle sur `Trigger.new` en enfilant un Queueable par `Id` épuise rapidement la limite de jobs en file et rend le suivi illisible.

## Pattern : éviter l'enchaînement incontrôlé

- stocker l'état de progression (via `Database.Stateful` en Batch, ou paramètre porté de job en job en Queueable) ;
- limiter explicitement la profondeur ou le nombre d'itérations restantes ;
- journaliser les erreurs, idéalement via un Finalizer plutôt qu'un `try/catch` local ;
- prévoir une reprise (rejouer depuis le dernier point connu, pas depuis zéro).

## Règles IA

- Ne jamais proposer @future pour du code nouveau sans justifier explicitement pourquoi Queueable ne convient pas.
- Toujours ajouter `Database.AllowsCallouts` quand un Queueable ou un Batch effectue un callout HTTP.
- Toujours borner un auto-chaînage Queueable par une condition d'arrêt vérifiable (liste qui diminue, compteur maximal).
- Toujours envisager un Finalizer sur un Queueable qui peut échouer silencieusement (callout externe, DML partiel).
- Ne jamais dupliquer les chiffres de limites de gouverneur dans ce fichier : renvoyer vers `../principes/04_LIMITES_GOUVERNEUR.md`.
- Ne jamais lancer un Batch pour un volume qu'un Queueable unique peut traiter sans risque de limite.

## Checklist IA

- [ ] Le traitement doit-il vraiment être synchrone ?
- [ ] Le volume justifie-t-il Batch plutôt que Queueable ?
- [ ] Les callouts sont-ils séparés du DML incompatible (pas de callout après DML non commité) ?
- [ ] Un job unique regroupe-t-il plusieurs enregistrements plutôt qu'un job par enregistrement ?
- [ ] Le Queueable expose-t-il un Finalizer pour tracer les échecs et permettre un retry ?
- [ ] Le chaînage (Queueable ou Batch) a-t-il une condition d'arrêt explicite ?
- [ ] `Database.Stateful` est-il utilisé si un état doit survivre entre les scopes d'un Batch ?
- [ ] La taille de scope du Batch est-elle adaptée à la complexité du traitement par enregistrement ?
- [ ] Un Platform Event est-il réellement nécessaire, ou un appel de méthode directe suffirait-il ?
- [ ] Les erreurs asynchrones sont-elles traçables (log, notification, objet d'erreur persistant) ?

## Voir aussi

- [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md)
- [`09_CONCURRENCE_VERROUILLAGE.md`](09_CONCURRENCE_VERROUILLAGE.md)
- [`11_DEBUG_DIAGNOSTICS.md`](11_DEBUG_DIAGNOSTICS.md)
- [`12_TESTS_UNITAIRES.md`](12_TESTS_UNITAIRES.md)
- [`../principes/03_VARIABLES_STATIQUES.md`](../principes/03_VARIABLES_STATIQUES.md)
