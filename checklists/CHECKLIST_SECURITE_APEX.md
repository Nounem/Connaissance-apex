# Checklist sécurité Apex

## Sharing

- [ ] `with sharing` / `without sharing` / `inherited sharing` choisi et justifié explicitement (jamais `without sharing` sans commentaire de justification).

## CRUD/FLS

- [ ] `WITH USER_MODE` ou `Security.stripInaccessible()` utilisé sur les requêtes exposées à un contexte utilisateur.
- [ ] DML sensibles passés en `AccessLevel.USER_MODE` si le contexte l'exige.
- [ ] Les opérations appelées depuis LWC/Aura (`@AuraEnabled`) respectent CRUD/FLS.

## Injection et requêtes dynamiques

- [ ] Toute requête SOQL/SOSL dynamique utilise des bind variables, jamais de concaténation directe d'une entrée utilisateur.
- [ ] Si concaténation inévitable, `String.escapeSingleQuotes()` appliqué systématiquement.

## Secrets et intégrations

- [ ] Named Credentials utilisées pour toute authentification externe (jamais de token/mot de passe en dur dans le code ou un Custom Metadata non protégé).
- [ ] Les tokens/secrets ne sont jamais loggés en clair.

## Exposition et surface d'attaque

- [ ] Pas d'exposition inutile en `global`/`public` (préférer le scoping le plus restrictif possible, `global` seulement si un package managé l'exige).
- [ ] Les endpoints REST (`@RestResource`) valident les entrées et retournent des codes HTTP appropriés.
- [ ] Attention particulière si le composant est accessible à un Guest User / Experience Cloud (accès anonyme) : sharing et FLS revus avec une rigueur accrue.

## Erreurs et logs

- [ ] Les erreurs retournées à l'UI ne divulguent pas de détails internes (stack trace, nom de champ technique).
- [ ] Les logs ne contiennent pas de données personnelles/sensibles.

## Permissions

- [ ] Les Permission Sets/Custom Permissions nécessaires sont documentés plutôt que supposés.

## Voir aussi

- [`../patterns/15_SECURITE_CRUD_FLS_SHARING.md`](../patterns/15_SECURITE_CRUD_FLS_SHARING.md)
- [`CHECKLIST_CODE_REVIEW_APEX.md`](CHECKLIST_CODE_REVIEW_APEX.md)
- [`../patterns/17_CALLOUTS_INTEGRATIONS.md`](../patterns/17_CALLOUTS_INTEGRATIONS.md)
- [`../patterns/18_INTEGRATION_LWC_APEX.md`](../patterns/18_INTEGRATION_LWC_APEX.md)
