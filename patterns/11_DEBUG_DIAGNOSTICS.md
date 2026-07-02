# Debugging et diagnostics Apex

## Objectif

Le diagnostic ne consiste pas seulement à écrire `System.debug`. Il faut pouvoir comprendre un incident en production sans dépasser les limites ni exposer des données sensibles.

## Règles IA

- Prévoir une journalisation contrôlée par configuration.
- Éviter les logs volumineux dans les boucles.
- Ne jamais journaliser de secrets, tokens, mots de passe ou données sensibles inutiles.
- Prévoir des corrélations : request id, record id, job id, contexte.

## Pattern de logger minimal

```apex
public class AppLogger {
    public enum Level { ERROR, WARN, INFO, DEBUG }

    public static void error(String message, Exception ex) {
        System.debug(LoggingLevel.ERROR, message + ' - ' + ex.getMessage());
    }
}
```

## Diagnostic d’erreurs limites

Certaines erreurs, comme les dépassements CPU, peuvent interrompre brutalement la transaction. Pour les traitements critiques, envisager des mécanismes de journalisation plus résilients, par exemple événements ou objet de log dans les limites acceptables.

## Checklist IA

- [ ] Les erreurs sont-elles capturées au bon niveau ?
- [ ] Les logs sont-ils activables/désactivables ?
- [ ] Les données sensibles sont-elles protégées ?
- [ ] Les jobs asynchrones ont-ils une trace d’exécution ?
