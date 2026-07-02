# Configuration applicative Apex

## Principe

Ne pas coder en dur ce qui doit varier par org, environnement, package ou client.

## Options de configuration

| Besoin | Solution |
|---|---|
| Paramètre déployable et versionnable | Custom Metadata Type |
| Paramètre modifiable par admin | Custom Metadata ou Custom Setting selon cas |
| Donnée transactionnelle | Objet custom |
| Cache temporaire partagé | Platform Cache |
| Feature flag | Custom Metadata + service de configuration |

## Service de configuration

```apex
public class AppConfig {
    private static Map<String, App_Config__mdt> cacheByName;

    public static App_Config__mdt get(String developerName) {
        if (cacheByName == null) {
            cacheByName = new Map<String, App_Config__mdt>();
            for (App_Config__mdt cfg : [
                SELECT DeveloperName, Enabled__c, Value__c
                FROM App_Config__mdt
            ]) {
                cacheByName.put(cfg.DeveloperName, cfg);
            }
        }
        return cacheByName.get(developerName);
    }
}
```

## Règles IA

- Centraliser l’accès à la configuration.
- Ne pas répéter des requêtes de configuration partout.
- Prévoir des valeurs par défaut sûres.
- Rendre les feature flags explicites.

## Checklist IA

- [ ] Aucun identifiant Salesforce codé en dur ?
- [ ] Les paramètres métier sont-ils externalisés ?
- [ ] L’accès à la configuration est-il centralisé ?
- [ ] Le comportement par défaut est-il sûr ?
