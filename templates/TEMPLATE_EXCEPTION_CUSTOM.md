# Template exception custom Apex

Une exception custom documente l'intention métier de l'erreur et permet à l'appelant de la distinguer d'une exception système. Ne jamais laisser remonter une `NullPointerException` ou une `DmlException` brute quand une erreur métier claire peut être levée à la place.

```apex
public class AccountServiceException extends Exception {
    // Dès qu'un constructeur custom est déclaré, Apex ne génère plus
    // automatiquement les 4 constructeurs par défaut (dont celui à un seul
    // String) : il faut donc le redéclarer explicitement si on en a besoin
    // ailleurs (ex: throw new AccountServiceException('...') ci-dessous).
    public AccountServiceException(String message) {
        super(message);
    }

    // Constructeur additionnel : utile pour agréger plusieurs erreurs
    // (ex: résultat d'un Database.update en mode partiel) en un seul message.
    public AccountServiceException(List<String> errors) {
        this(String.join(errors, '; '));
    }
}
```

## Usage dans un service

```apex
public inherited sharing class AccountService {
    public static void activate(Account accountRecord) {
        if (accountRecord.OwnerId == null) {
            throw new AccountServiceException(
                'Impossible d\'activer le compte ' + accountRecord.Id +
                ' : aucun propriétaire défini.'
            );
        }
        // ...
    }
}
```

Le message doit être clair et actionnable : il doit dire quoi corriger, pas seulement constater un état invalide.

## Règle : ne jamais catcher `Exception` en silence

```apex
// À proscrire : erreur avalée, aucune trace, débogage impossible en prod.
try {
    AccountService.activate(accountRecord);
} catch (Exception e) {}
```

Tout `catch` doit soit relancer (éventuellement enrichi), soit gérer explicitement le cas, et dans tous les cas logger l'erreur pour la diagnostiquer en production — voir [`../patterns/11_DEBUG_DIAGNOSTICS.md`](../patterns/11_DEBUG_DIAGNOSTICS.md).

```apex
try {
    AccountService.activate(accountRecord);
} catch (AccountServiceException e) {
    System.debug(LoggingLevel.ERROR, e.getMessage());
    throw e;
}
```

## Cas particulier : méthode `@AuraEnabled` exposée à un LWC

Un LWC ne peut afficher que le message d'une `AuraHandledException` ; toute autre exception est masquée côté client pour des raisons de sécurité. Il faut donc catcher l'exception métier et la reconvertir explicitement — voir [`../patterns/18_INTEGRATION_LWC_APEX.md`](../patterns/18_INTEGRATION_LWC_APEX.md) pour le détail du pattern.

```apex
@AuraEnabled
public static void activateAccount(Id accountId) {
    try {
        AccountService.activate(new Account(Id = accountId));
    } catch (AccountServiceException e) {
        // setMessage() explicite obligatoire : voir 18_INTEGRATION_LWC_APEX.md
        // (le constructeur seul ne garantit pas la propagation du message au LWC).
        AuraHandledException ex = new AuraHandledException(e.getMessage());
        ex.setMessage(e.getMessage());
        throw ex;
    }
}
```

## Checklist IA

- [ ] L'exception custom hérite d'`Exception` et porte un nom explicite du domaine métier concerné.
- [ ] Le message d'erreur est clair et actionnable, pas une simple constatation technique.
- [ ] Aucun `catch (Exception e) {}` vide dans le code livré.
- [ ] Toute exception catchée est loggée avant d'être relancée ou gérée.
- [ ] Une méthode `@AuraEnabled` catche les exceptions métier et les reconvertit en `AuraHandledException`.

## Voir aussi

- [`TEMPLATE_SERVICE_APEX.md`](TEMPLATE_SERVICE_APEX.md)
- [`../patterns/11_DEBUG_DIAGNOSTICS.md`](../patterns/11_DEBUG_DIAGNOSTICS.md)
- [`../patterns/18_INTEGRATION_LWC_APEX.md`](../patterns/18_INTEGRATION_LWC_APEX.md)
