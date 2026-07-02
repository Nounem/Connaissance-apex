# Packaging et déploiement

## Principe

Sur Salesforce, le mode de déploiement influence l’architecture. Une solution spécifique à une org et un package distribué n’ont pas les mêmes contraintes.

## Règles IA

- Demander ou expliciter le contexte : projet interne, client unique, package managé, unlocked package.
- Éviter les dépendances implicites à la configuration d’une seule org.
- Prévoir l’évolution des métadonnées.
- Limiter l’exposition `global` au strict nécessaire.
- Prévoir la compatibilité ascendante des APIs publiques.

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

- conventions d’équipe ;
- dépendances avec Flows existants ;
- données de production ;
- stratégie Git/SFDX ;
- rollback.

## Checklist IA

- [ ] Le type de déploiement est-il identifié ?
- [ ] Les dépendances sont-elles documentées ?
- [ ] Les APIs exposées sont-elles stables ?
- [ ] Les tests sont-ils portables ?
- [ ] La configuration est-elle déployable ?
