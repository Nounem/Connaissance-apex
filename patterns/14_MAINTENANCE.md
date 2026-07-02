# Maintenance Apex

## Principe

Le coût principal d’un logiciel vient de sa maintenance. L’IA doit donc générer du code compréhensible, modulaire, testable et facile à faire évoluer.

## Règles IA

- Favoriser les noms explicites.
- Séparer orchestration, logique métier, accès aux données et configuration.
- Éviter le code magique ou trop dynamique sans justification.
- Documenter les hypothèses et limites.
- Préférer les petites méthodes cohérentes.
- Ne pas optimiser prématurément au prix de la lisibilité, sauf contrainte de limites claire.

## Architecture simple recommandée

```text
force-app/main/default/classes/
├── AccountTriggerHandler.cls
├── AccountService.cls
├── AccountSelector.cls
├── AccountDomain.cls
├── AppConfig.cls
└── AppLogger.cls
```

## Responsabilités

| Classe | Responsabilité |
|---|---|
| TriggerHandler | Router le contexte trigger |
| Service | Orchestrer les cas d’usage |
| Selector | Centraliser SOQL |
| Domain | Règles métier d’un objet |
| Config | Accès configuration |
| Logger | Diagnostics |

## Checklist IA

- [ ] Le code sera-t-il compréhensible dans 6 mois ?
- [ ] Les responsabilités sont-elles séparées ?
- [ ] Les tests protègent-ils les comportements métier ?
- [ ] Les noms sont-ils explicites ?
- [ ] Les dépendances sont-elles visibles ?
