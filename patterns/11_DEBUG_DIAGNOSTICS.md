# Debugging et diagnostics Apex

## Objectif

Le diagnostic en production ne consiste pas à écrire `System.debug` et espérer qu'un debug log soit disponible au bon moment. Il faut pouvoir reconstituer, après coup, ce qui s'est passé dans une transaction métier — y compris quand cette transaction a échoué et a fait un rollback. Un log qui disparaît avec la transaction qu'il devait expliquer ne sert à rien.

## Pourquoi `System.debug` seul est insuffisant en production

- **Pas de persistance réelle** : un debug log est limité à 2 Mo. Au-delà, Salesforce tronque le log en coupant les lignes les plus anciennes en premier — les logs les plus utiles (souvent en fin de transaction, là où l'erreur survient) peuvent survivre, mais tout le contexte qui précède peut disparaître.
- **Rétention limitée** : les debug logs ne sont conservés que quelques jours et en nombre limité par utilisateur/Trace Flag. Impossible de les utiliser comme historique d'incidents.
- **Pas de corrélation** : `System.debug` n'associe pas nativement les lignes d'une même transaction métier (un même `Id` de commande, un même job Batch, une même requête d'intégration). Retrouver le fil d'une transaction dans un debug log brut est artisanal.
- **Niveau non configurable dynamiquement** : activer un niveau `FINEST` pour un utilisateur précis pendant une fenêtre de temps nécessite de créer un Trace Flag manuellement dans Setup — ce n'est pas pilotable depuis le code métier ni activable à la volée pour un incident en cours.
- **Non interrogeable** : un debug log n'est pas requêtable en SOQL, ne peut pas alimenter un rapport, un dashboard ou une alerte.

Conclusion : `System.debug` reste utile en développement et dans les Trace Flags ciblés, mais ne constitue pas une stratégie de journalisation en production.

## Pattern recommandé : objet de log persistant

Créer un objet custom (`Log__c`) qui survit indépendamment de la durée de vie du debug log, interrogeable en SOQL, et exploitable via un rapport ou une alerte.

Champs suggérés : `Niveau__c` (picklist ERROR/WARN/INFO/DEBUG), `Message__c` (Long Text Area), `Contexte__c` (classe.méthode, Id d'enregistrement, job id), `StackTrace__c` (Long Text Area), `Timestamp__c` (DateTime), `TransactionId__c` (`Request.getCurrent().getRequestId()` pour corréler tous les logs d'une même transaction, y compris entre synchrone et asynchrone).

```apex
public inherited sharing class AppLogger {
    private static List<Log__c> buffer = new List<Log__c>();

    public static void error(String context, String message, Exception ex) {
        buffer.add(new Log__c(
            Niveau__c = 'ERROR',
            Message__c = message,
            Contexte__c = context,
            StackTrace__c = ex != null ? ex.getStackTraceString() : null,
            TransactionId__c = Request.getCurrent().getRequestId(),
            Timestamp__c = System.now()
        ));
    }

    public static void flush() {
        if (buffer.isEmpty()) return;
        // allOrNone = false : un log défaillant (champ trop long, validation)
        // ne doit jamais faire échouer le reste des logs ni la transaction appelante.
        Database.insert(buffer, false);
        buffer.clear();
    }
}
```

`Database.insert(logs, false)` est délibéré : le paramètre `allOrNone = false` garantit qu'une erreur sur un enregistrement de log (troncature de champ, validation, erreur de partage) n'empêche pas l'insertion des autres logs, et surtout ne remonte jamais d'exception qui perturberait le code appelant. **Un mécanisme de journalisation ne doit jamais être une nouvelle cause d'échec.**

## Le point critique : la rollback-safety

Un `insert` classique sur `Log__c`, exécuté dans la même transaction que le traitement métier, est annulé par `Database.rollback()` ou par une exception non gérée qui fait échouer la transaction globale — exactement au moment où le log serait le plus utile. Journaliser une erreur qui déclenche elle-même le rollback, dans la transaction qui rollback, est inefficace si l'insertion du log fait partie du même DML annulé.

Deux stratégies permettent de découpler le log de la transaction métier :

### 1. Platform Event en mode `Publish Immediately`

Attention à une nuance essentielle, souvent source d'erreur : le comportement de publication d'un Platform Event dépend d'un paramètre de configuration de l'objet `__e` lui-même, le **Publish Behavior**, qui a deux valeurs possibles :

- **`Publish After Commit` (comportement par défaut)** : `EventBus.publish()` met l'événement en file, mais celui-ci n'est réellement délivré aux abonnés qu'**à la fin de la transaction, si et seulement si elle commit avec succès**. En cas de rollback (exception non gérée, `Database.rollback()`), l'événement n'est **jamais publié** — il est annulé avec le reste de la transaction. Ce mode par défaut ne résout donc pas le problème de rollback-safety.
- **`Publish Immediately`** : l'événement est publié immédiatement au moment de l'appel à `EventBus.publish()`, de façon découplée, indépendamment du sort ultérieur de la transaction Apex qui l'a déclenché (rollback inclus). C'est ce mode, et lui seul, qui rend la publication réellement rollback-safe.

Le mode `Publish Immediately` **n'est pas activé par défaut** : il doit être configuré explicitement sur la définition du Platform Event dans Setup (champ *Publish Behavior* de l'objet `__e`), et cette activation est irréversible une fois effectuée. Un `LogEvent__e` créé sans cette configuration explicite reste en `Publish After Commit` et ne protège donc pas le log contre le rollback qu'il est censé documenter — vérifier ce paramètre est un prérequis avant de s'appuyer sur ce pattern.

```apex
public inherited sharing class LogEventPublisher {
    public static void publishError(String context, String message, Exception ex) {
        EventBus.publish(new LogEvent__e(
            Niveau__c = 'ERROR',
            Message__c = message,
            Contexte__c = context,
            StackTrace__c = ex != null ? ex.getStackTraceString() : null,
            TransactionId__c = Request.getCurrent().getRequestId()
        ));
    }
}
```

Un trigger sur `LogEvent__e` (`after insert`, exécuté dans un contexte séparé après publication) insère alors les `Log__c` correspondants. Si — et seulement si — `LogEvent__e` est configuré en `Publish Immediately`, l'événement a déjà été publié au moment de l'appel et le log persiste même si la transaction d'origine fait ensuite un rollback complet.

### 2. Finalizer / Queueable après coup

Un `Finalizer` attaché à un Queueable s'exécute dans un contexte transactionnel **distinct**, après la fin (succès ou échec) de l'exécution asynchrone qu'il surveille. Il peut donc journaliser un échec même si le Queueable d'origine a rollback ses propres DML.

```apex
public class RecalculationFinalizer implements Finalizer {
    public void execute(FinalizerContext ctx) {
        if (ctx.getResult() == ParentJobResult.UNHANDLED_EXCEPTION) {
            AppLogger.error(
                'AccountRecalculationJob',
                'Échec du job : ' + ctx.getException().getMessage(),
                ctx.getException()
            );
            AppLogger.flush();
        }
    }
}
```

Voir [`08_ASYNCHRONE.md`](08_ASYNCHRONE.md) pour l'attachement du Finalizer au Queueable et la gestion des reprises.

## Référence de l'industrie : Nebula Logger

Pour un projet nécessitant une journalisation sérieuse, ne pas réinventer ce pattern à la main : **Nebula Logger** est la bibliothèque open-source de référence de l'écosystème Salesforce pour ce besoin. Elle fournit un buffer de logs en mémoire (comme `AppLogger.buffer` ci-dessus, mais complet), un flush automatique en fin de transaction (y compris via Platform Events pour la rollback-safety), une corrélation automatique entre transactions synchrones et asynchrones, et une interface d'exploitation (objets, Lightning App) prête à l'emploi. L'IA doit recommander son évaluation avant de construire un système de logging maison au-delà d'un besoin ponctuel.

## Debug Logs et Trace Flags

`System.debug` reste pertinent en complément, avec un niveau explicite :

```apex
System.debug(LoggingLevel.ERROR, 'Échec de validation : ' + record.Id);
```

Niveaux disponibles, du plus filtré au plus verbeux : `ERROR`, `WARN`, `INFO`, `DEBUG`, `FINE`, `FINER`, `FINEST`. Un niveau trop verbeux (`FINEST` sur toute une transaction volumineuse) augmente le risque de troncature à 2 Mo et noie l'information utile. Restreindre les Trace Flags à un utilisateur et une fenêtre de temps précis pour du diagnostic ciblé, jamais en permanence sur un utilisateur d'intégration à fort volume.

## Diagnostic complet d'une exception

Ne jamais se limiter à `ex.getMessage()`. Pour un diagnostic exploitable :

```apex
try {
    AccountService.recalculate(accountIds);
} catch (Exception ex) {
    AppLogger.error(
        'AccountService.recalculate',
        ex.getTypeName() + ' : ' + ex.getMessage(),
        ex
    );
    // getStackTraceString() : localise la ligne exacte de l'échec.
    // getCause() : remonte l'exception originale si celle-ci a été
    // ré-encapsulée dans une exception custom (ex. AuraHandledException).
    if (ex.getCause() != null) {
        System.debug(LoggingLevel.ERROR, 'Cause : ' + ex.getCause().getMessage());
    }
    throw ex;
}
```

## Apex Exception Email : un filet insuffisant seul

Setup > Apex Exception Email envoie un email automatique à une adresse configurée dès qu'une exception non gérée survient dans un contexte asynchrone ou un composant géré par la plateforme. C'est un filet de sécurité utile en dernier recours, mais insuffisant seul en production : pas de structure exploitable, pas d'historique interrogeable, pas de corrélation, dépendant de la disponibilité d'une boîte mail surveillée. À combiner avec un logging persistant, jamais à utiliser comme unique stratégie de diagnostic.

## Anti-patterns

```apex
// Aucune valeur ajoutée : ne survit pas au-delà du debug log,
// non interrogeable, non corrélable.
System.debug('Erreur : ' + e.getMessage());
```

```apex
// DML de log dans la même transaction que le traitement métier :
// si le traitement fait un rollback, le log disparaît avec lui.
try {
    insert newOrder;
} catch (DmlException e) {
    insert new Log__c(Message__c = e.getMessage()); // Rollback global : ce log n'existera jamais.
    throw e;
}
```

```apex
// allOrNone = true (par défaut) : si le log lui-même échoue
// (champ trop long, validation), l'exception remonte et casse l'appelant.
insert logs;
```

## Règles IA

- Ne jamais considérer `System.debug` comme une solution de journalisation en production.
- Toujours vérifier la rollback-safety d'un mécanisme de log avant de le valider : le log doit être conçu pour survivre à l'échec qu'il décrit.
- Toujours utiliser `Database.insert(logs, false)` pour les DML de logging, jamais un `insert` strict.
- Toujours inclure un identifiant de corrélation (`Request.getCurrent().getRequestId()` ou équivalent) pour relier les logs d'une même transaction métier.
- Recommander Nebula Logger plutôt qu'un système maison dès que le besoin dépasse un logger ponctuel de dépannage.
- Ne jamais journaliser de secrets, tokens, mots de passe ou données personnelles sensibles inutiles, même dans un `StackTrace__c`.

## Checklist IA

- [ ] Le log survit-il si la transaction métier qui l'a généré fait un rollback ?
- [ ] Le mécanisme de log utilise-t-il un Platform Event ou un Finalizer/Queueable séparé plutôt qu'un DML dans la transaction en échec ?
- [ ] Le DML de log utilise-t-il `allOrNone = false` pour ne jamais faire échouer l'appelant ?
- [ ] Le niveau de journalisation est-il configurable dynamiquement (utilisateur, durée) plutôt que figé dans le code ?
- [ ] Les logs sont-ils corrélables entre eux (Id de transaction, de job, d'enregistrement) ?
- [ ] Le diagnostic capture-t-il `getTypeName()`, `getStackTraceString()` et `getCause()`, pas seulement `getMessage()` ?
- [ ] Les échecs asynchrones (Queueable, Batch) sont-ils systématiquement journalisés via un Finalizer ?
- [ ] Apex Exception Email est-il utilisé en complément d'un logging persistant, et non comme seule stratégie ?
- [ ] Les données sensibles sont-elles exclues des messages et stack traces journalisés ?

## Voir aussi

- [`08_ASYNCHRONE.md`](08_ASYNCHRONE.md)
- [`../principes/02_CONTEXTE_EXECUTION.md`](../principes/02_CONTEXTE_EXECUTION.md)
- [`13_PACKAGING_DEPLOIEMENT.md`](13_PACKAGING_DEPLOIEMENT.md)
