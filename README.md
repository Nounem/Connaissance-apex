# Base de connaissance IA — Apex Salesforce

Cette base de connaissance est destinée à être utilisée comme contexte par une IA qui génère, corrige ou audite du code Apex Salesforce.

Elle ne remplace pas la documentation officielle Salesforce. Elle formalise des règles opérationnelles inspirées des pratiques avancées Apex : contexte d'exécution, sécurité, variables statiques, limites gouverneur, bulkification, triggers, collections, SOQL/SOSL avancé, asynchrone, concurrence, callouts et intégrations, configuration, intégration LWC, debug, tests, packaging et maintenance.

Pour un apprentissage progressif plutôt qu'un contexte IA, voir [`Apex_Guide_Debutant_Bonnes_Pratiques_FR.md`](Apex_Guide_Debutant_Bonnes_Pratiques_FR.md) — un guide pédagogique autonome destiné à un lecteur humain, complémentaire à cette base.

## Structure

```
00_INSTRUCTIONS_IA_APEX.md          Méta-instructions comportementales pour l'IA
01_PRINCIPES_FONDAMENTAUX.md        Les 5 concepts à vérifier systématiquement

principes/                          Fondations transverses à tout code Apex
  02_CONTEXTE_EXECUTION.md            Sharing, system/user mode, ordre d'exécution, transactions
  03_VARIABLES_STATIQUES.md           Cache transactionnel, guards, constantes
  04_LIMITES_GOUVERNEUR.md            Limites chiffrées, classe Limits
  05_BULK_PATTERNS.md                 Pattern bulk standard, delta, collections 1-N

patterns/                           Sujets techniques par domaine
  06_COLLECTIONS.md                   List/Set/Map, tri, collections imbriquées
  07_TRIGGERS.md                      Architecture trigger, ordre d'exécution avec Flows
  08_ASYNCHRONE.md                    Queueable, Batch, Schedulable, Future, Finalizer, Platform Events
  09_CONCURRENCE_VERROUILLAGE.md      Locking, retry, data skew
  10_CONFIGURATION.md                 Custom Metadata, Custom Settings, Platform Cache
  11_DEBUG_DIAGNOSTICS.md             Logging persistant et rollback-safe
  12_TESTS_UNITAIRES.md               @TestSetup, Assert, mocks, Stub API
  13_PACKAGING_DEPLOIEMENT.md         2GP, scratch orgs, CI/CD, destructive changes
  14_MAINTENANCE.md                   Architecture en couches, Selector, ApexDoc
  15_SECURITE_CRUD_FLS_SHARING.md     CRUD/FLS/Sharing, injection SOQL — à lire en priorité
  16_SOQL_SOSL_AVANCE.md              Relations, agrégats, requêtes dynamiques
  17_CALLOUTS_INTEGRATIONS.md         HTTP/REST/SOAP, Named Credentials
  18_INTEGRATION_LWC_APEX.md          @AuraEnabled, @wire, gestion d'erreurs LWC

checklists/
  CHECKLIST_CODE_REVIEW_APEX.md       Checklist de revue de code généraliste
  CHECKLIST_SECURITE_APEX.md          Checklist sécurité dédiée

prompts/
  PROMPT_SYSTEME_IA_APEX.md           Prompt système pour la génération de code
  PROMPT_AUDIT_CODE_APEX.md           Prompt pour l'audit de code existant
  PROMPT_GENERATION_TRIGGER_APEX.md   Prompt spécifique à la génération d'un trigger

templates/
  TEMPLATE_SERVICE_APEX.md            Service bulkifié avec Selector et gestion d'erreurs
  TEMPLATE_TRIGGER_HANDLER.md         Trigger + framework handler (bypass, recursion guard)
  TEMPLATE_SELECTOR_APEX.md           Classe Selector avec enforcement FLS
  TEMPLATE_EXCEPTION_CUSTOM.md        Exception custom et stratégie try/catch
  TEMPLATE_TEST_DATA_FACTORY.md       Factory de données de test bulkifiée

Apex_Guide_Debutant_Bonnes_Pratiques_FR.md   Guide pédagogique autonome (public humain)
```

Chaque fichier de `principes/` et `patterns/` se termine par une section « Voir aussi » qui renvoie vers les fichiers connexes — suivre ces liens pour couvrir un sujet transversal (ex. sécurité, bulkification) à travers plusieurs fichiers.

## Utilisation recommandée

Avant chaque demande de génération de code Apex, fournir à l'IA au minimum :

1. `00_INSTRUCTIONS_IA_APEX.md`
2. `01_PRINCIPES_FONDAMENTAUX.md`
3. `patterns/15_SECURITE_CRUD_FLS_SHARING.md` (sécurité — systématique, quel que soit le besoin)
4. le ou les fichiers thématiques `principes/`/`patterns/` correspondant au besoin (ex. `07_TRIGGERS.md` + `templates/TEMPLATE_TRIGGER_HANDLER.md` pour un trigger)
5. `checklists/CHECKLIST_CODE_REVIEW_APEX.md` et `checklists/CHECKLIST_SECURITE_APEX.md`
6. `prompts/PROMPT_SYSTEME_IA_APEX.md` (génération) ou `prompts/PROMPT_AUDIT_CODE_APEX.md` (revue)

Pour un audit de code existant, remplacer 4-6 par `prompts/PROMPT_AUDIT_CODE_APEX.md` seul, les checklists y étant déjà référencées.

## Règle principale

L'IA doit toujours produire du code Apex :

- bulkifié ;
- testable ;
- compatible avec les limites gouverneur ;
- lisible et maintenable ;
- robuste face aux Flows, triggers, validations et packages existants ;
- sécurisé selon le contexte CRUD/FLS/sharing (voir `patterns/15_SECURITE_CRUD_FLS_SHARING.md`).
