# Variables statiques en Apex

## Principe

En Apex, une variable statique n’est pas une variable globale permanente. Elle existe uniquement pendant le contexte d’exécution courant.

## Usages recommandés

### 1. Cache transactionnel

```apex
public class CurrentUserInfo {
    private static User cachedUser;

    public static User getUser() {
        if (cachedUser == null) {
            cachedUser = [
                SELECT Id, ProfileId, TimeZoneSidKey
                FROM User
                WHERE Id = :UserInfo.getUserId()
                LIMIT 1
            ];
        }
        return cachedUser;
    }
}
```

### 2. Guard anti-récursion ciblé

Utiliser une clé métier précise, pas un simple booléen global.

### 3. Contrôle de tests

```apex
@TestVisible
private static Boolean forceFailureForTest = false;
```

## Usages interdits

- Stocker des données entre transactions.
- Simuler une session utilisateur.
- Remplacer Custom Metadata, Custom Settings, Platform Cache ou un objet persistant.
- Utiliser un booléen global qui désactive tout un trigger sans distinction.

## Risques

- Une variable statique trop large peut masquer des traitements nécessaires.
- Un cache transactionnel trop volumineux peut consommer le heap.
- Une variable de test mal protégée peut introduire un comportement non souhaité.

## Checklist IA

- [ ] La variable statique est-elle limitée à la transaction ?
- [ ] Le nom indique-t-il clairement son usage ?
- [ ] Le cache réduit-il vraiment SOQL ou CPU ?
- [ ] Le cache ne consomme-t-il pas trop de heap ?
- [ ] Les variables de test sont-elles `@TestVisible` et contrôlées ?
