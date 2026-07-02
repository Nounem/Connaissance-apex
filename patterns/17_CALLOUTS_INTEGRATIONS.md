# Callouts HTTP, REST/SOAP et intégrations

## Objectif

Un callout Apex sort de l'org vers un système externe (API REST, service SOAP legacy, webhook). Il doit gérer explicitement trois choses que le code métier interne n'a pas à gérer : la latence et l'indisponibilité réseau, l'authentification vers un système tiers, et des erreurs HTTP qui ne sont pas des exceptions Apex mais des codes de statut à interpréter.

## `Http`, `HttpRequest`, `HttpResponse` : callout GET

```apex
public inherited sharing class WeatherClient {
    public class WeatherClientException extends Exception {}

    public static Integer getCurrentTemperature(String city) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:WeatherApi/v1/current?city=' + EncodingUtil.urlEncode(city, 'UTF-8'));
        req.setMethod('GET');
        req.setHeader('Accept', 'application/json');
        req.setTimeout(10000); // ms — 10s ; max autorisé 120000 (120s)

        Http http = new Http();
        HttpResponse res;
        try {
            res = http.send(req);
        } catch (System.CalloutException ex) {
            // Timeout, DNS, connexion refusée : erreur réseau, pas une erreur HTTP.
            throw new WeatherClientException('Callout météo indisponible : ' + ex.getMessage());
        }

        if (res.getStatusCode() == 200) {
            Map<String, Object> body = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
            return (Integer) body.get('temperature');
        }

        if (res.getStatusCode() == 404) {
            throw new WeatherClientException('Ville inconnue de l’API météo : ' + city);
        }

        if (res.getStatusCode() >= 500) {
            throw new WeatherClientException('Service météo en erreur (' + res.getStatusCode() + ') : ' + res.getBody());
        }

        throw new WeatherClientException('Réponse inattendue (' + res.getStatusCode() + ') : ' + res.getBody());
    }
}
```

## Callout POST avec corps JSON

```apex
public inherited sharing class OrderExportClient {
    public class OrderExportException extends Exception {}

    public static void exportOrder(Id orderId, String payloadJson) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:ExternalErp/api/orders');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(payloadJson);
        req.setTimeout(15000);

        Http http = new Http();
        HttpResponse res = http.send(req);

        // 2xx = succès ; toute autre plage doit être traitée explicitement,
        // jamais assumée comme un succès silencieux.
        if (res.getStatusCode() == 200 || res.getStatusCode() == 201) {
            return;
        }
        if (res.getStatusCode() == 400) {
            throw new OrderExportException('Payload rejeté par l’ERP : ' + res.getBody());
        }
        if (res.getStatusCode() == 401 || res.getStatusCode() == 403) {
            throw new OrderExportException('Authentification ERP refusée — vérifier le Named Credential.');
        }
        if (res.getStatusCode() == 429) {
            throw new OrderExportException('Throttling ERP (429) — prévoir un retry différé.');
        }
        throw new OrderExportException('Échec export commande ' + orderId + ' (' + res.getStatusCode() + ')');
    }
}
```

Ne jamais se contenter d'un `try { ... } catch (Exception e) { ... }` générique autour d'un callout : cela confond une erreur réseau (`CalloutException`), un délai dépassé et une réponse HTTP 4xx/5xx parfaitement valide du point de vue du protocole. Chaque cas appelle une décision différente (retry, alerte, correction de payload, abandon).

## Named Credentials : ne jamais coder en dur endpoint et authentification

Ne jamais écrire une URL de base ni un token/mot de passe en dur dans une classe Apex, et ne jamais les stocker dans un Custom Metadata ou Custom Setting non chiffré accessible en lecture — un Custom Metadata Type standard est lisible par tout utilisateur ayant accès à Setup ou via requête SOQL selon les droits. Le Named Credential centralise l'endpoint et l'authentification (Basic, OAuth 2.0 Client Credentials/JWT Bearer Flow, certificat) côté plateforme :

```apex
req.setEndpoint('callout:MonNamedCredential/api/v2/comptes');
```

Salesforce résout l'endpoint réel, injecte les en-têtes d'authentification et gère le rafraîchissement de token automatiquement — le code Apex ne manipule jamais le secret. Bénéfices concrets :

- **rotation sans déploiement** : changer un mot de passe, un secret OAuth ou un certificat se fait dans Setup, sans toucher au code ni redéployer ;
- **auditabilité** : un seul endroit à vérifier pour savoir vers quel système un flux appelle et avec quelles credentials ;
- **portabilité entre environnements** (sandbox/prod) : le même code Apex fonctionne partout, seul le Named Credential change de valeur par org.

## External Credentials : la génération actuelle

Depuis les releases récentes, Salesforce sépare le Named Credential (endpoint + comportement du callout) de l'**External Credential** (schéma d'authentification et paramètres associés : principal, scope OAuth, séquence d'en-têtes). Un même External Credential peut être réutilisé par plusieurs Named Credentials, et plusieurs Principals (par utilisateur ou nommé) peuvent partager un External Credential avec des identités différentes. Pour une intégration nouvelle, préférer ce modèle (Named Credential legacy « avec authentification intégrée » reste supporté mais moins flexible pour du multi-principal).

## Remote Site Settings : legacy, insuffisant seul

Un Remote Site Setting autorise seulement un domaine à recevoir des callouts — il ne gère ni authentification, ni rotation de secret, ni endpoint dynamique. Il reste nécessaire techniquement dans certains cas historiques, mais ne doit jamais être le seul mécanisme de sécurisation d'une intégration authentifiée : utiliser un Named Credential (qui enregistre implicitement l'autorisation de domaine) plutôt que de gérer manuellement un Remote Site Setting à côté d'un token codé en dur.

## Limites de gouverneur applicables aux callouts

Ne pas dupliquer les chiffres ici : voir [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md). À retenir en synthèse pour la conception : 100 callouts maximum par transaction (synchrone ou asynchrone), timeout configurable via `setTimeout()` jusqu'à 120 secondes (10 secondes par défaut si non précisé), taille d'une requête de callout ou de sa réponse limitée à 6 Mo en synchrone et 12 Mo en asynchrone (limite appliquée à chacune, pas à leur somme — et qui compte dans la heap size globale de la transaction). Un flux qui approche ces plafonds doit être redécoupé (pagination côté API externe, agrégation batch) plutôt que de pousser la limite.

## Callouts asynchrones : Queueable et Continuation

Un callout ne peut pas être exécuté après un DML non committé dans la même transaction synchrone. Pour découpler un callout du flux transactionnel courant (Trigger, DML préalable, traitement long), utiliser un Queueable avec `Database.AllowsCallouts`, ou une `Continuation` pour un callout long déclenché depuis l'UI. Ce sujet est détaillé dans [`08_ASYNCHRONE.md`](08_ASYNCHRONE.md) — ne pas le réexpliquer ici, s'y référer directement pour le chaînage, le Finalizer de suivi d'erreur et les limites de jobs en file.

## Tester un callout : `HttpCalloutMock`

Un test unitaire ne doit jamais effectuer un vrai appel réseau. Exemple minimal :

```apex
@IsTest
private class WeatherClientMock implements HttpCalloutMock {
    public HttpResponse respond(HttpRequest req) {
        HttpResponse res = new HttpResponse();
        res.setStatusCode(200);
        res.setHeader('Content-Type', 'application/json');
        res.setBody('{"temperature":18}');
        return res;
    }
}

@IsTest
static void shouldReturnTemperatureFromMock() {
    Test.setMock(HttpCalloutMock.class, new WeatherClientMock());
    Test.startTest();
    Integer temp = WeatherClient.getCurrentTemperature('Paris');
    Test.stopTest();
    Assert.areEqual(18, temp);
}
```

Détail complet (mock multi-endpoint, `StaticResourceCalloutMock`, cas d'erreur 4xx/5xx simulés) dans [`12_TESTS_UNITAIRES.md`](12_TESTS_UNITAIRES.md).

## SOAP : WSDL2Apex

Pour consommer un service SOAP legacy, générer les classes de stub Apex depuis Setup (Apex Classes > Generate from WSDL), qui produit une classe cliente typée et ses classes de types de données à partir du WSDL fourni. Cette classe générée expose ensuite des méthodes Apex classiques masquant l'enveloppe SOAP :

```apex
ExternalErpService.OrderPort port = new ExternalErpService.OrderPort();
port.timeout_x = 15000;
ExternalErpService.OrderResponse response = port.submitOrder(orderRequest);
```

Le stub généré respecte les mêmes limites de callout (100/transaction, timeout, taille) que `HttpRequest`. Pour toute intégration nouvelle, préférer REST/JSON quand le système externe l'expose : plus léger, plus simple à débuguer (payload lisible), mieux outillé côté Named Credential/External Credential. Ne recourir à WSDL2Apex que lorsque le système tiers n'expose que du SOAP.

## Sécurité des callouts

- Ne jamais logger un token, une clé d'API, un mot de passe ou un en-tête `Authorization` en clair, y compris dans un message d'exception ou un `StackTrace__c` — voir [`11_DEBUG_DIAGNOSTICS.md`](11_DEBUG_DIAGNOSTICS.md) pour la stratégie de journalisation rollback-safe et la règle d'exclusion des données sensibles.
- Ne jamais faire confiance aux données reçues d'un système externe : valider le format, les types et les bornes avant toute écriture (DML), et appliquer les mêmes contrôles CRUD/FLS que pour une saisie utilisateur — voir [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md).
- Vérifier systématiquement le code de statut avant de traiter le corps de la réponse : un corps de réponse peut être présent et bien formé même sur un 4xx/5xx (message d'erreur structuré de l'API), le confondre avec une réponse de succès casse silencieusement le traitement aval.
- Préférer HTTPS systématiquement ; un Named Credential empêche par construction un callout vers un endpoint non déclaré ou non autorisé.

## Anti-patterns

```apex
// URL et token en dur : aucune rotation possible sans déploiement,
// secret visible dans le code source et l'historique Git.
req.setEndpoint('https://api.externe.com/v1/comptes');
req.setHeader('Authorization', 'Bearer sk_live_abc123...');
```

```apex
// catch générique : masque la différence entre timeout réseau,
// erreur HTTP 4xx/5xx et succès — aucune décision fiable possible ensuite.
try {
    HttpResponse res = http.send(req);
} catch (Exception e) {
    // silence total, y compris sur un 500 qui n'a pourtant levé aucune exception
}
```

```apex
// Aucune vérification du statut : un 500 avec un corps JSON d'erreur
// est traité comme si la commande avait réussi.
HttpResponse res = http.send(req);
Map<String, Object> body = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
processOrder(body); // suppose un succès sans l'avoir vérifié
```

```apex
// Log d'un secret en clair : fuite potentielle via un objet Log__c
// consultable par tout utilisateur ayant accès à cet objet.
System.debug('Appel avec token=' + accessToken);
```

## Règles IA

- Toujours utiliser un Named Credential (ou External Credential + Named Credential) pour l'endpoint et l'authentification, jamais une URL/un token en dur ni un Custom Metadata non protégé.
- Toujours vérifier `getStatusCode()` explicitement (2xx, 4xx, 5xx) avant d'exploiter `getBody()` — ne jamais assumer un succès par défaut.
- Toujours distinguer une `System.CalloutException` (réseau/timeout) d'une réponse HTTP d'erreur : ce sont deux causes différentes appelant des traitements différents.
- Toujours définir un `setTimeout()` explicite adapté au SLA du système appelé plutôt que de garder la valeur par défaut sans y réfléchir.
- Ne jamais logger un secret, un token ou un en-tête d'authentification, même en cas d'erreur de callout.
- Toujours valider/assainir les données reçues d'un système externe avant toute écriture DML.
- Ne jamais dupliquer les chiffres de limites de gouverneur ici : renvoyer vers `../principes/04_LIMITES_GOUVERNEUR.md`.
- Ne jamais réexpliquer Queueable/Continuation ni `HttpCalloutMock` en détail dans ce fichier : renvoyer vers `08_ASYNCHRONE.md` et `12_TESTS_UNITAIRES.md`.
- Pour une intégration nouvelle sans contrainte SOAP imposée par le système tiers, recommander REST/JSON plutôt que WSDL2Apex.

## Checklist IA

- [ ] L'endpoint et l'authentification passent-ils par un Named Credential (ou External Credential), sans URL/token en dur ?
- [ ] Le code de statut HTTP est-il vérifié explicitement avant d'exploiter le corps de la réponse ?
- [ ] Les erreurs réseau (`CalloutException`) et les erreurs HTTP (4xx/5xx) sont-elles distinguées et traitées séparément ?
- [ ] Un `setTimeout()` explicite est-il défini, cohérent avec le SLA du système appelé ?
- [ ] Le callout est-il correctement positionné vis-à-vis du DML (pas de callout après un DML non committé dans le même contexte synchrone) ?
- [ ] Le test associé utilise-t-il `HttpCalloutMock`/`Test.setMock()` sans jamais effectuer un vrai appel réseau ?
- [ ] Aucun secret, token ou en-tête d'authentification n'est journalisé, y compris en cas d'erreur ?
- [ ] Les données reçues du système externe sont-elles validées avant toute écriture (DML) ?
- [ ] Le nombre de callouts par transaction reste-t-il très en-deçà de la limite (100), avec un fallback asynchrone si le volume l'exige ?

## Voir aussi

- [`08_ASYNCHRONE.md`](08_ASYNCHRONE.md)
- [`12_TESTS_UNITAIRES.md`](12_TESTS_UNITAIRES.md)
- [`15_SECURITE_CRUD_FLS_SHARING.md`](15_SECURITE_CRUD_FLS_SHARING.md)
- [`11_DEBUG_DIAGNOSTICS.md`](11_DEBUG_DIAGNOSTICS.md)
- [`../principes/04_LIMITES_GOUVERNEUR.md`](../principes/04_LIMITES_GOUVERNEUR.md)
