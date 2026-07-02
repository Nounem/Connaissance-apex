# Checklist sécurité Apex

- [ ] La classe utilise un mode de sharing explicite.
- [ ] Les opérations appelées depuis LWC/Aura respectent CRUD/FLS.
- [ ] Les requêtes dynamiques protègent les injections SOQL.
- [ ] Les erreurs retournées à l’UI ne divulguent pas d’informations internes.
- [ ] Les logs ne contiennent pas de secrets ou données sensibles.
- [ ] Les endpoints REST valident les entrées.
- [ ] Les permissions nécessaires sont documentées.
