# Base de connaissance IA — Apex Salesforce

Cette base de connaissance est destinée à être utilisée comme contexte par une IA qui génère, corrige ou audite du code Apex Salesforce.

Elle ne remplace pas la documentation officielle Salesforce. Elle formalise des règles opérationnelles inspirées des pratiques avancées Apex : contexte d’exécution, variables statiques, limites gouverneur, bulkification, triggers, asynchrone, concurrence, configuration, tests, packaging et maintenance.

## Utilisation recommandée

Avant chaque demande de génération de code Apex, fournir à l’IA au minimum :

1. `00_INSTRUCTIONS_IA_APEX.md`
2. `01_PRINCIPES_FONDAMENTAUX.md`
3. le fichier thématique correspondant au besoin
4. `checklists/CHECKLIST_CODE_REVIEW_APEX.md`
5. `prompts/PROMPT_SYSTEME_IA_APEX.md`

## Règle principale

L’IA doit toujours produire du code Apex :

- bulkifié ;
- testable ;
- compatible avec les limites gouverneur ;
- lisible et maintenable ;
- robuste face aux Flows, triggers, validations et packages existants ;
- sécurisé selon le contexte CRUD/FLS/sharing.
