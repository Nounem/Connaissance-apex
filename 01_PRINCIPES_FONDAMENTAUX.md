# Principes fondamentaux Apex pour IA

## Penser Apex, pas Java

Apex ressemble syntaxiquement à Java ou C#, mais son comportement réel dépend fortement de la plateforme Salesforce. L’IA doit donc raisonner en termes de transaction Salesforce, contexte d’exécution, limites gouverneur, ordre d’exécution et automatisations déclaratives.

## Les quatre concepts à vérifier systématiquement

### 1. Contexte d’exécution

Un contexte d’exécution est la transaction dans laquelle s’exécute le code Apex. Plusieurs morceaux de code peuvent partager ce même contexte : triggers, Flows, validations, workflows historiques, Process Builder, appels synchrones et packages.

### 2. Variables statiques

Les variables statiques vivent uniquement pendant le contexte d’exécution. Elles ne sont pas un stockage permanent. Elles sont utiles pour le cache transactionnel, les guards anti-récursion et les comportements de test contrôlés.

### 3. Limites gouverneur

Les limites ne sont pas un détail technique. Elles structurent l’architecture : nombre de requêtes, lignes interrogées, DML, CPU, heap, callouts, jobs asynchrones.

### 4. Bulk patterns

Tout code doit être conçu pour fonctionner sur plusieurs enregistrements. Une méthode qui accepte un seul `Id` est souvent moins robuste qu’une méthode qui accepte un `Set<Id>`.

## Règle de conception

Toujours concevoir d’abord la version bulk, puis fournir éventuellement une surcharge pour un seul enregistrement.

```apex
public static void processOne(Id recordId) {
    processMany(new Set<Id>{ recordId });
}

public static void processMany(Set<Id> recordIds) {
    if (recordIds == null || recordIds.isEmpty()) return;
    // Requête unique, traitement en mémoire, DML groupé.
}
```
