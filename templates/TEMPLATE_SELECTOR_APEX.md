# Template Selector Apex

Centraliser toutes les requêtes SOQL d'un objet dans une classe Selector dédiée : les champs interrogés sont listés une seule fois, l'enforcement FLS/CRUD est appliqué systématiquement, et les services appellent le Selector au lieu d'écrire leur propre `SELECT`.

```apex
public inherited sharing class AccountSelector {
    // Liste explicite des champs : jamais de SELECT *, jamais de champ
    // ajouté "au cas où". Toute nouvelle consommation de champ passe par ici.
    public static List<String> getFields() {
        return new List<String>{
            'Id', 'Name', 'Industry', 'OwnerId', 'BillingCountry'
        };
    }

    public static List<Account> selectById(Set<Id> ids) {
        if (ids == null || ids.isEmpty()) return new List<Account>();

        return Database.query(
            'SELECT ' + String.join(getFields(), ', ') +
            ' FROM Account WHERE Id IN :ids WITH USER_MODE'
        );
    }

    public static Map<Id, Account> selectByIds(Set<Id> accountIds) {
        return new Map<Id, Account>(selectById(accountIds));
    }

    // Requête dynamique : utile quand un appelant n'a besoin que d'un
    // sous-ensemble de champs (ex: LWC léger). Rester simple : pas de
    // constructeur de requête générique pour tous les objets, un Selector
    // par objet reste plus lisible et plus facile à sécuriser.
    // ATTENTION injection SOQL : fieldCsv est concaténé tel quel dans le
    // texte de la requête (seuls les :ids passent par bind variable). Si
    // fieldCsv provient d'un appelant non fiable (LWC, paramètre exposé),
    // valider chaque champ contre la whitelist getFields() avant de
    // construire la chaîne, par exemple :
    //   Set<String> allowed = new Set<String>(getFields());
    //   for (String f : fieldCsv.split(',')) {
    //       if (!allowed.contains(f.trim())) { throw new IllegalArgumentException('Champ non autorisé: ' + f); }
    //   }
    public static List<Account> selectByIdWithFields(Set<Id> ids, String fieldCsv) {
        if (ids == null || ids.isEmpty() || String.isBlank(fieldCsv)) {
            return new List<Account>();
        }
        return Database.query(
            'SELECT ' + fieldCsv + ' FROM Account WHERE Id IN :ids WITH USER_MODE'
        );
    }
}
```

## Pourquoi centraliser dans un Selector

- **Réutilisabilité** : un seul endroit à modifier quand la règle métier de sélection change (ex: exclure les comptes archivés).
- **Cohérence des champs** : évite que chaque service liste ses propres champs, avec des risques de divergence et de requêtes redondantes dans une même transaction.
- **Mockabilité en test** : en extrayant les méthodes derrière une interface (`IAccountSelector`), un test peut injecter un faux Selector et vérifier la logique du service sans dépendre de données réelles ni de gouverneurs SOQL.
- **Sécurité homogène** : `WITH USER_MODE` (ou `Security.stripInaccessible` si besoin de granularité) est posé une seule fois, au lieu d'être oublié dans un service isolé.

## Checklist IA

- [ ] Aucun `SELECT *` ; les champs sont listés explicitement dans une constante ou une méthode dédiée.
- [ ] Toute requête utilise `WITH USER_MODE` (ou un équivalent `Security.stripInaccessible`) pour l'enforcement FLS/CRUD.
- [ ] Les méthodes acceptent des `Set<Id>` (jamais un seul `Id` isolé) pour rester bulk-safe.
- [ ] La classe ne contient aucune logique métier, uniquement des requêtes.
- [ ] Un Selector existe par objet plutôt qu'un constructeur de requête générique partagé entre objets.
- [ ] Toute liste de champs dynamique injectée dans un `Database.query()` (ex. `fieldCsv`) est validée contre une whitelist de champs connus avant concaténation, pour éviter l'injection SOQL.

## Voir aussi

- [`../patterns/14_MAINTENANCE.md`](../patterns/14_MAINTENANCE.md)
- [`../patterns/15_SECURITE_CRUD_FLS_SHARING.md`](../patterns/15_SECURITE_CRUD_FLS_SHARING.md)
- [`../patterns/16_SOQL_SOSL_AVANCE.md`](../patterns/16_SOQL_SOSL_AVANCE.md)
- [`TEMPLATE_SERVICE_APEX.md`](TEMPLATE_SERVICE_APEX.md)
