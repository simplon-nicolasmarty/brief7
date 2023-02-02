# DAT Brief7


## Objectifs
Utiliser le service managé AKS pour déployer une application de vote, une base de donnée redis avec un volume de stockage persistant. 
L'application de vote doit être disponible via une url qui pointera vers une application gateway relié à un Ingress AKS qui permet l'accès au nodes voteapp.
Configurer le pipeline pour un déploiement automatique à chaque push dans le registry Dockerhub de l’application.
![](https://i.imgur.com/L6GXwjF.png)

Le CI/CD utilisé est un processus automatisé qui permet de construire, tester et déployer rapidement et en toute confiance des logiciels en production. Cela améliore la qualité et la rapidité du développement de logiciels.

Le pipeline à pour but la vérification du versioning de l'image Docker hébergé sur Dockerhub de la voteapp pour déclancher ou non son déployement su AKS via l'outil de CI/CD.

## Les choix effectués pour réaliser le brief:

- **Github** comme outil de versionning et d'hébergement du manifest de deployement pour AKS et des livrables à fournir. Il est également interconnectable à Azure Devops.


- **Azure Devops** comme outil de déploiement automatique pour AKS car il est simple d'utilisation, bien documenté et gratuit 


## Les éléments à déployer pour réaliser le brief :
- AKS (azure kubernetes service) 2 nodes pour la maintenabilité
- Un Ingress dans AKS pour les nodes voteapp
- Une application gateway sur Azure
- Pipeline de déployement via Azure Devops
- Dépot Github pour le manifest AKS



## La liste des éléments déployé avec kubernetes :
- Secrets
- mot de passe pour redis
- certificat TLS
- clé privée TLS
- volume persistant de stockage
- node base de donnée redis
- deux nodes voteapp


**Url d'accès à l'application pour les utilisateurs:
https://vote.uncia.fr**

