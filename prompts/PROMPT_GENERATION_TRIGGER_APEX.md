# Prompt de génération d'un trigger propre

Crée un trigger Apex propre pour l'objet [Objet] sur les événements [before insert / before update / after insert / after update / before delete / after delete / after undelete — préciser].

Contraintes :

- Un trigger "mince" par objet (un seul point d'entrée), qui ne fait que déléguer au handler.
- Un handler basé sur le framework de [`TEMPLATE_TRIGGER_HANDLER.md`](../templates/TEMPLATE_TRIGGER_HANDLER.md) : toutes les branches déclarées dans le trigger doivent être effectivement traitées (pas de `before insert` déclaré sans méthode `beforeInsert` appelée), avec guard anti-récursion et mécanisme de bypass.
- Une classe service par domaine métier, appelée depuis le handler, bulkifiée (collections en entrée, jamais un seul `SObject`).
- Collections `Set<Id>`/`Map<Id, SObject>` pour tout accès par identifiant.
- CRUD/FLS respectés sur les requêtes (`WITH USER_MODE` ou `Security.stripInaccessible`) — voir [`15_SECURITE_CRUD_FLS_SHARING.md`](../patterns/15_SECURITE_CRUD_FLS_SHARING.md).
- Une classe de test qui couvre : cas nominal, cas bulk (200 enregistrements), cas vide, cas de mise à jour d'un champ non pertinent (ne doit rien déclencher), et récursion (mise à jour depuis le handler lui-même ne doit pas boucler).

Avant le code, indique les hypothèses (ordre d'exécution attendu vis-à-vis des Flows/validation rules existants). Après le code, liste les points de vigilance.
