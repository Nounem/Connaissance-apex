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
| Feature flag global | Custom Metadata + service de configuration |
| Feature flag lié à un profil/permission set | Custom Permission |
| Valeur par utilisateur/profil sans déploiement | Custom Setting (Hierarchy) |

## Règle par défaut : Custom Metadata Type

Custom Metadata Type est le choix par défaut pour toute configuration applicative moderne, sauf besoin explicite contraire :

- déployable comme du code (inclus dans les packages, versionné dans les repositories) ;
- disponible avant DML dans les triggers (pas de dépendance à une transaction de données) ;
- accessible sans DML via `Type_mdt.getAll()` / `getInstance()`.

Idée reçue à corriger explicitement : les requêtes SOQL sur du Custom Metadata **comptent bien** dans les limites SOQL classiques (100 requêtes en synchrone / 200 en asynchrone, 50 000 lignes par transaction), contrairement à une croyance répandue. Seul l'accès via les méthodes `getAll()`/`getInstance()` de la classe générée évite une requête SOQL explicite ; une requête `SELECT ... FROM Xxx__mdt` reste une requête SOQL normale et doit être mise en cache statique comme n'importe quelle autre.

Custom Setting (Hierarchy) reste pertinent uniquement pour un besoin réel de valeur différenciée par utilisateur ou par profil, sans nécessité de déploiement (ex. seuil ajustable par un admin directement en production). En dehors de ce cas précis, préférer Custom Metadata Type.

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

Précision sur `cacheByName` : cette carte statique n'est valide que pour la durée d'une seule transaction Apex. Elle n'est jamais partagée entre transactions ni entre utilisateurs — chaque nouvelle exécution repart avec un cache vide. C'est différent de Platform Cache, qui peut être partagé au niveau de l'org entière et persiste entre transactions.

## Platform Cache

Utiliser Platform Cache (`Cache.OrgPartition` ou `Cache.SessionPartition`) pour partager une donnée coûteuse à recalculer entre plusieurs transactions ou utilisateurs, avec un TTL explicite.

```apex
public class AppConfigCache {
    private static final String PARTITION_NAME = 'local.AppConfig';

    public static App_Config__mdt get(String developerName) {
        Cache.OrgPartition partition = Cache.Org.getPartition(PARTITION_NAME);
        Object cached = partition.get(developerName);

        if (cached != null) {
            return (App_Config__mdt) cached;
        }

        App_Config__mdt cfg = App_Config__mdt.getInstance(developerName);
        if (cfg != null) {
            // TTL en secondes : ici 3600s (1h). Max 48h (172800s).
            partition.put(developerName, cfg, 3600);
        }
        return cfg;
    }
}
```

- `Cache.OrgPartition` : partagé par tous les utilisateurs de l'org, adapté à de la configuration commune.
- `Cache.SessionPartition` : propre à la session utilisateur courante, adapté à des données contextuelles par utilisateur (ex. filtres UI).
- Mise en garde : une valeur mise en cache ne peut pas dépasser 100 Ko (limite uniforme, quelle que soit l'édition) et la capacité totale de la partition est limitée par l'allocation Platform Cache de l'org ; ne pas y stocker de volumes de données larges ou non bornés.

## Custom Permission pour les feature flags par utilisateur

Quand un feature flag doit dépendre de qui exécute le code (profil, permission set) plutôt que d'un paramètre global d'org, préférer une `Custom Permission` assignée via des permission sets, vérifiée avec `FeatureManagement.checkPermission`.

```apex
if (FeatureManagement.checkPermission('Beta_New_Pricing_Engine')) {
    // Chemin de code réservé aux utilisateurs disposant de la permission personnalisée.
}
```

Ne pas utiliser Custom Metadata pour ce cas : cela obligerait à gérer soi-même des listes d'utilisateurs/profils autorisés, alors que Custom Permission s'appuie sur le modèle de sécurité natif (permission sets, profils).

## Création dynamique de Custom Metadata (cas avancé)

Pour créer ou modifier du Custom Metadata depuis Apex (ex. un outil d'administration interne), utiliser `Metadata.Operations.enqueueDeployment`. Le déploiement est asynchrone (`Metadata.DeployCallback` pour suivre le résultat) et ne doit pas être utilisé pour de la configuration lue dans le même contexte transactionnel.

```apex
Metadata.CustomMetadata customMetadata = new Metadata.CustomMetadata();
customMetadata.fullName = 'App_Config__mdt.New_Flag';
customMetadata.label = 'New_Flag';

Metadata.CustomMetadataValue enabledField = new Metadata.CustomMetadataValue();
enabledField.field = 'Enabled__c';
enabledField.value = true;
customMetadata.values.add(enabledField);

Metadata.DeployContainer container = new Metadata.DeployContainer();
container.addMetadata(customMetadata);

Metadata.Operations.enqueueDeployment(container, null);
```

À réserver aux outils d'administration ou de setup ; ne jamais dépendre du résultat de ce déploiement dans la même transaction (il est traité de façon asynchrone par la plateforme).

## Règles IA

- Centraliser l’accès à la configuration.
- Ne pas répéter des requêtes de configuration partout.
- Prévoir des valeurs par défaut sûres.
- Rendre les feature flags explicites.
- Préférer Custom Metadata Type par défaut ; ne descendre vers Custom Setting (Hierarchy) que pour un besoin réel par utilisateur/profil sans déploiement.
- Ne jamais supposer qu'une requête SOQL sur du Custom Metadata échappe aux limites SOQL.
- Utiliser Platform Cache pour partager une donnée coûteuse entre transactions/utilisateurs, avec un TTL explicite.
- Utiliser Custom Permission pour un feature flag lié à des droits utilisateur, pas à une configuration globale.

## Checklist IA

- [ ] Aucun identifiant Salesforce codé en dur ?
- [ ] Les paramètres métier sont-ils externalisés ?
- [ ] L’accès à la configuration est-il centralisé ?
- [ ] Le comportement par défaut est-il sûr ?
- [ ] Custom Metadata Type est-il le choix par défaut, avec justification si un autre mécanisme est utilisé ?
- [ ] Les requêtes sur Custom Metadata sont-elles mises en cache statique plutôt que répétées ?
- [ ] Platform Cache est-il utilisé avec un TTL et une taille de clé raisonnables ?
- [ ] Un feature flag par utilisateur/permission utilise-t-il Custom Permission plutôt qu'un paramètre global ?
- [ ] Une création dynamique de Custom Metadata via `Metadata.Operations` ne suppose-t-elle pas un résultat synchrone ?

## Voir aussi

- [`09_CONCURRENCE_VERROUILLAGE.md`](09_CONCURRENCE_VERROUILLAGE.md)
- [`../principes/03_VARIABLES_STATIQUES.md`](../principes/03_VARIABLES_STATIQUES.md)
- [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md)
