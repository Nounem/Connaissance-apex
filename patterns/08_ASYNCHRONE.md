# Apex asynchrone

## Quand utiliser l’asynchrone ?

Utiliser l’asynchrone lorsque le traitement :

- consomme trop de CPU ;
- effectue des callouts ;
- traite un volume important ;
- peut être différé ;
- doit être isolé du contexte transactionnel courant.

## Choisir le bon mécanisme

| Besoin | Option recommandée |
|---|---|
| Traitement différé simple | Queueable |
| Callout contrôlé | Queueable avec `Database.AllowsCallouts` |
| Gros volumes | Batch Apex |
| Planification | Scheduled Apex |
| Notification événementielle | Platform Event |
| Réaction à changement de données | Change Data Capture |

## Queueable recommandé

```apex
public class AccountRecalculationJob implements Queueable {
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

## Attention aux appels asynchrones multiples

Regrouper les demandes dans un seul job quand possible. Ne pas créer un job par enregistrement.

## Pattern : éviter l’enchaînement incontrôlé

- stocker l’état de progression ;
- limiter explicitement la profondeur ;
- journaliser les erreurs ;
- prévoir une reprise.

## Platform Events

Utiles pour la journalisation durable de certains échecs ou pour communiquer avec d’autres composants. À utiliser avec prudence pour ne pas transformer chaque traitement en architecture événementielle inutilement complexe.

## Checklist IA

- [ ] Le traitement doit-il vraiment être synchrone ?
- [ ] Le volume justifie-t-il Batch plutôt que Queueable ?
- [ ] Les callouts sont-ils séparés du DML incompatible ?
- [ ] Un job unique regroupe-t-il plusieurs enregistrements ?
- [ ] Les erreurs asynchrones sont-elles traçables ?
