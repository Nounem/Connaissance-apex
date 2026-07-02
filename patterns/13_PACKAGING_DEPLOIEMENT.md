# Packaging et déploiement

## Principe

Sur Salesforce, le mode de déploiement influence l'architecture. Une solution spécifique à une org et un package distribué n'ont pas les mêmes contraintes. Choisir le mauvais modèle (ex. tout gérer en changesets sur une org unique quand plusieurs équipes travaillent en parallèle) génère de la dette technique difficile à corriger a posteriori.

> Note : les commandes `sf` ci-dessous (et leurs flags) reflètent la Salesforce CLI actuelle, mais cette CLI évolue vite (renommages, dépréciations). Vérifier la syntaxe exacte dans `sf <commande> --help` ou la doc officielle au moment de l'exécution, en particulier en CI/CD.

## Règles IA

- Demander ou expliciter le contexte : projet interne, client unique, package managé, unlocked package.
- Éviter les dépendances implicites à la configuration d'une seule org.
- Prévoir l'évolution des métadonnées.
- Limiter l'exposition `global` au strict nécessaire.
- Prévoir la compatibilité ascendante des APIs publiques.
- Privilégier les Unlocked Packages en 2GP pour tout développement source-driven multi-sandbox.
- Toujours faire un `--dry-run` (validation-only) avant un déploiement réel en production.
- Gérer les suppressions de composants via `destructiveChanges.xml`, jamais en supprimant manuellement dans l'org cible.
- Préférer les Permission Sets aux Profiles pour toute permission nouvelle ou évolutive.

## 2GP vs 1GP

Le packaging Salesforce a deux générations, avec des implications structurantes pour le code que l'IA génère.

- **1GP (First-Generation Packaging)** : le package est développé directement dans une org "de développement" (Developer Edition). La source de vérité est l'org, pas Git. Fragile en CI/CD, difficile à versionner proprement, dépendances entre packages peu explicites.
- **2GP (Second-Generation Packaging)** : le code source (dans Git) est la source de vérité. Le package est généré à partir du repo via Salesforce CLI. Chaque version de package est immuable et traçable à un commit.

Pourquoi 2GP est recommandé aujourd'hui :

- **Source control-driven** : le repo Git fait autorité, pas une org de packaging fragile à maintenir.
- **CI/CD friendly** : création de version de package scriptable (`sf package version create`), intégrable dans un pipeline.
- **Dépendances explicites** : un package 2GP déclare ses dépendances vers d'autres packages dans `sfdx-project.json` (champ `dependencies`), ce qui évite les couplages implicites et cachés.
- **Isolation** : chaque package 2GP a son propre namespace de test et ses propres org de scratch pour la validation, sans polluer une org partagée.

En pratique : pour tout nouveau projet, générer les packages en 2GP. Ne conserver du 1GP que pour la maintenance de packages legacy déjà publiés sous cette forme (la migration 1GP -> 2GP n'est pas automatique et doit être planifiée).

## Unlocked Packages vs Managed Packages

| Critère | Unlocked Package | Managed Package |
|---|---|---|
| Usage cible | Interne à une entreprise, multi-org (dev/UAT/prod), multi-équipes | Distribution externe (AppExchange, ISV) |
| Namespace | Optionnel | Obligatoire |
| Protection de l'IP (code caché) | Non — le code reste lisible dans l'org cible | Oui — Apex compilé, non lisible |
| Réversibilité | Composants modifiables/désinstallables plus librement | Certains éléments (classes `global`) sont irréversibles une fois publiés |
| Cas d'usage type | Application interne développée par plusieurs équipes d'une même entreprise, brique technique partagée entre plusieurs orgs internes | Éditeur logiciel vendant une solution à des clients externes |

Règle de décision simple : si le consommateur final est une autre équipe de la même entreprise (ou vous-même dans un autre sandbox), utiliser un **Unlocked Package**. Si le consommateur est un client externe qui installe le produit depuis l'AppExchange sans accès au code source, utiliser un **Managed Package**.

## Scratch Orgs

Les scratch orgs sont des orgs jetables, créées à la demande à partir d'une définition déclarative, utilisées pour le développement isolé et les pipelines CI.

Structure minimale de `sfdx-project.json` :

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true,
      "package": "mon-package",
      "versionName": "ver 1.0",
      "versionNumber": "1.0.0.NEXT"
    }
  ],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "60.0",
  "packageAliases": {
    "mon-package": "0Ho000000000000AAA"
  }
}
```

Création d'une scratch org à partir d'un fichier de définition (`config/project-scratch-def.json`) :

```bash
sf org create scratch -f config/project-scratch-def.json -a mon-org --duration-days 7 --set-default
sf project deploy start -o mon-org
sf apex run test -o mon-org --code-coverage --result-format human
sf org delete scratch -o mon-org --no-prompt
```

Points clés du cycle de vie :

- Durée de vie courte (1 à 30 jours), pensée pour être recréée à volonté, jamais comme environnement persistant.
- Toujours créée depuis une **Dev Hub** org, à partir de l'état du repo Git (pas de configuration manuelle post-création qui ne serait pas versionnée).
- Idéale pour la CI : une scratch org par pull request, détruite après les tests, garantit l'absence de pollution entre exécutions.
- Ne jamais stocker de données de production dans une scratch org ; utiliser des jeux de données de test ou des Data Factories Apex.

## Stratégie de branches

Adapter la stratégie Git au nombre d'environnements Salesforce disponibles. Deux approches courantes :

- **Trunk-based simplifié** : une branche `main` unique, des feature branches courtes fusionnées rapidement, déploiement continu vers une org d'intégration. Convient aux petites équipes avec peu de sandboxes.
- **Gitflow simplifié à la Salesforce** : une branche par environnement persistant (`main` = production, `uat` = sandbox UAT, `develop` = sandbox d'intégration), chaque merge déclenchant un déploiement vers l'org correspondante. Convient aux organisations avec plusieurs sandboxes stables.

Règle commune : ne jamais déployer manuellement en production hors pipeline. Toute promotion d'un environnement à l'autre doit suivre le même chemin Git -> CI/CD, sans exception "juste cette fois".

## CI/CD minimal

Exemple de pipeline GitHub Actions : validation (`--dry-run`) sur chaque pull request, déploiement réel avec exécution des tests Apex après merge sur `main`.

```yaml
name: salesforce-ci-cd

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  validate:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Salesforce CLI
        run: npm install --global @salesforce/cli
      - name: Authentifier l'org cible
        run: sf org login sfdx-url -f <(echo "$SFDX_AUTH_URL") -a target-org
        env:
          SFDX_AUTH_URL: ${{ secrets.SFDX_AUTH_URL }}
      - name: Validation-only (dry-run) avec tests
        run: |
          sf project deploy start \
            --source-dir force-app \
            --target-org target-org \
            --dry-run \
            --test-level RunLocalTests \
            --wait 30

  deploy:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Salesforce CLI
        run: npm install --global @salesforce/cli
      - name: Authentifier l'org cible
        run: sf org login sfdx-url -f <(echo "$SFDX_AUTH_URL") -a target-org
        env:
          SFDX_AUTH_URL: ${{ secrets.SFDX_AUTH_URL }}
      - name: Déploiement réel avec tests
        run: |
          sf project deploy start \
            --source-dir force-app \
            --target-org target-org \
            --test-level RunLocalTests \
            --wait 30
```

Principes à respecter :

- Le job de validation (`--dry-run`) tourne sur chaque pull request, jamais le déploiement réel.
- Le déploiement réel ne se déclenche qu'après merge sur la branche protégée.
- `--test-level RunLocalTests` garantit l'exécution des tests Apex du package (obligatoire pour tout déploiement en production avec du code Apex).
- Les identifiants d'authentification (`SFDX_AUTH_URL`) sont stockés en secrets CI, jamais en clair dans le repo.

## Delta deployment

Un déploiement complet (déployer tout `force-app`) est fiable mais lent et risqué sur de gros repos (redéploiement de composants non modifiés, risque accru de conflits). Le **delta deployment** ne déploie que les fichiers modifiés entre deux commits (ou deux branches).

Outil de référence : `sfdx-git-delta`, qui génère les dossiers `package/` (à déployer) et `destructiveChanges/` (à supprimer) à partir d'un diff Git.

```bash
sf sgd source delta \
  --to HEAD --from origin/main \
  --output-dir .delta \
  --generate-delta

sf project deploy start \
  --manifest .delta/package/package.xml \
  --target-org target-org \
  --test-level RunLocalTests
```

Usage recommandé : delta deployment en CI pour accélérer les validations sur pull request ; déploiement complet réservé aux mises en production planifiées ou en cas de doute sur la cohérence de l'org cible.

## Destructive Changes

Salesforce ne supprime jamais un composant par simple absence dans le package déployé : la suppression doit être déclarée explicitement.

- `destructiveChangesPre.xml` : suppressions exécutées **avant** le déploiement du `package.xml` (utile quand un composant doit disparaître avant qu'un autre du même nom soit recréé).
- `destructiveChangesPost.xml` : suppressions exécutées **après** le déploiement (cas le plus courant, ex. suppression d'un champ ou d'une classe devenus obsolètes).

```xml
<!-- destructiveChangesPost.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>Ancien_Champ__c</members>
        <name>CustomField</name>
    </types>
    <types>
        <members>AncienneClasseObsolete</members>
        <name>ApexClass</name>
    </types>
    <version>60.0</version>
</Package>
```

```bash
sf project deploy start \
  --manifest package.xml \
  --post-destructive-changes destructiveChangesPost.xml \
  --target-org target-org
```

Règle : ne jamais supprimer un composant manuellement dans l'org de production. Toute suppression passe par un `destructiveChanges*.xml` versionné, revu en pull request comme n'importe quel autre changement.

## Permission Sets vs Profiles

Pour tout droit d'accès (CRUD objet, FLS, accès Apex/VF, permissions systèmes) déployable et évolutif, préférer les **Permission Sets** aux Profiles.

- **Profiles** : un seul profil par utilisateur, difficile à faire évoluer sans casser d'autres droits déjà accordés, mélange souvent des paramètres d'org (langue, fuseau horaire) avec des droits métier. À réduire au minimum (accès de base, license type).
- **Permission Sets** (et **Permission Set Groups**) : cumulables, assignables dynamiquement, découplés du profil, donc plus faciles à tester et déployer indépendamment. Une nouvelle fonctionnalité Apex/Flow doit venir avec son propre Permission Set dédié plutôt que de modifier un Profile existant.

```xml
<!-- permissionsets/Acces_Facturation.permissionset-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Accès Facturation</label>
    <classAccesses>
        <apexClass>FacturationService</apexClass>
        <enabled>true</enabled>
    </classAccesses>
    <objectPermissions>
        <object>Facture__c</object>
        <allowRead>true</allowRead>
        <allowCreate>true</allowCreate>
        <allowEdit>true</allowEdit>
        <allowDelete>false</allowDelete>
        <viewAllRecords>false</viewAllRecords>
        <modifyAllRecords>false</modifyAllRecords>
    </objectPermissions>
</PermissionSet>
```

Règle : si une revue de code fait apparaître une modification de Profile pour accorder un accès à une nouvelle classe Apex ou un nouvel objet, la refuser et demander la création/mise à jour d'un Permission Set à la place.

## Package managé

Attention particulière à :

- classes `global` irréversibles ;
- signatures publiques exposées ;
- noms de champs/objets ;
- configuration par Custom Metadata ;
- tests indépendants des données client ;
- interactions avec automatisations inconnues.

## Déploiement projet interne

Attention particulière à :

- conventions d'équipe ;
- dépendances avec Flows existants ;
- données de production ;
- stratégie Git/SFDX ;
- rollback.

## Checklist IA

- [ ] Le type de déploiement est-il identifié (interne, unlocked, managed) ?
- [ ] Le mode de packaging (1GP/2GP) est-il cohérent avec le besoin de CI/CD ?
- [ ] Les dépendances entre packages sont-elles déclarées explicitement dans `sfdx-project.json` ?
- [ ] Les APIs exposées sont-elles stables et documentées ?
- [ ] Les tests sont-ils portables (indépendants de données spécifiques à une org) ?
- [ ] La configuration est-elle déployable (Custom Metadata, pas de saisie manuelle post-déploiement) ?
- [ ] Un `--dry-run` est-il exécuté avant tout déploiement réel ?
- [ ] Les suppressions de composants passent-elles par `destructiveChanges*.xml` ?
- [ ] Les nouveaux droits d'accès sont-ils portés par un Permission Set plutôt qu'un Profile ?
- [ ] Le delta deployment est-il envisagé pour accélérer les validations sur pull request ?

## Voir aussi

- [`14_MAINTENANCE.md`](14_MAINTENANCE.md)
- [`12_TESTS_UNITAIRES.md`](12_TESTS_UNITAIRES.md)
- [`11_DEBUG_DIAGNOSTICS.md`](11_DEBUG_DIAGNOSTICS.md)
