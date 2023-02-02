# BRIEF 7

![](https://i.imgur.com/84Zo0AU.png)

## Les choix effectués pour réaliser le brief:

- **Github** comme outil de versionning et d'hébergement du manifest de deployement pour AKS et des livrables à fournir. Il est également interconnectable à Azure Devops.


- **Azure Devops** comme outil de déploiement automatique pour AKS car il est simple d'utilisation, bien documenté et gratuit 

## Le manifest Github

Le manifest est en language YAML (configuration de déploiement) Kubernetes pour une app de vote exécutée sur Azure Kubernetes Service (AKS). Il comprend un déploiement, un autoscaler pod horizontal, un service ClusterIP, des ressources d'entrée pour l'app de vote et un serveur redis.

Le déploiement de l'app de vote crée 2 répliques du conteneur "voteapp" exécutant l'image "simplonasa/azure_voting_app:{{ version }}". Il relie le port 80 du conteneur au port de l'hôte et définit les demandes/limites de ressources et les variables d'environnement.

L'autoscaler pod horizontal augmente automatiquement le nombre de répliques du déploiement de l'app de vote en fonction de l'utilisation du CPU (min. 2, max. 8).

Le service ClusterIP rend les pods de l'app de vote et les pods redis accessibles au sein du cluster.

Le déploiement redis crée une réplique du conteneur "redis" exécutant l'image "redis" en reliant le port 6379 du conteneur au port de l'hôte.

L'ingress permet au trafic externe d'accéder au service de l'app de vote via une passerelle d'application Azure avec un secret TLS.

```
# Deployement de l'application voteapp connecté à une base de donnée Redis

apiVersion: apps/v1
kind: Deployment
metadata:
  name: voteapp
  labels:
    app: vote-app
spec:
  selector:
    matchLabels:
      app: vote-app
  replicas: 2
  template:
    metadata:
      labels:
        app: vote-app
    spec:
      containers:
      - name: voteapp
        image: simplonasa/azure_voting_app:{{ version }}
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m
        env:
        - name: REDIS
          value: "clusteredis"
        - name: STRESS_SECS
          value: "2"
        - name: REDIS_PWD
          valueFrom:
            secretKeyRef:
              name: redispwd
              key: nm-password
---
# Création d'un cluster pour voteapp accessible sur le port 80

apiVersion: v1
kind: Service
metadata:
  name: voteapp
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: vote-app
---
# Déployement de la base de donnée Redis

apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis-bd
spec:
  selector:
    matchLabels:
      app: redis-bd
  replicas: 1
  template:
    metadata:
      labels:
        app: redis-bd
    spec:
      volumes: 
      - name: redis-vol
        persistentVolumeClaim:
          claimName: nm-vol
      containers:
      - name: redis
        image: redis
        args: ["--requirepass","$(REDIS_PWD)"]
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "no"
        - name: REDIS_PWD
          valueFrom:
            secretKeyRef:
              name: redispwd
              key: nm-password
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: redis-vol
          mountPath: "/data"
---
# cR2
apiVersion: v1
kind: Service
metadata:
  name: clusteredis
spec:
  type: ClusterIP
  ports:
  - port: 6379
  selector:
    app: redis-bd
---
#Création d'un Ingress pour que l'application gateway puisse accèder aux nodes voteapp
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingressvote
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  tls:
    - secretName: test-tls
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: voteapp
            port:
              number: 80
---
# Création d'un autoscaling de l'application dès 70% d'utilisation du procésseur
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: scaleapp
spec:
  maxReplicas: 8
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: voteapp
  targetCPUUtilizationPercentage: 70
```

![](https://i.imgur.com/yhWYLF0.png)

## Pipeline de vérification des mises à jours

Ce pipeline deployé sur Azure DevOps, qui a pour but de déployer une application Voteapp avec une base de donnée Redis sur un cluster managé Kubernetes (AKS). Il se compose de 3 étapes principales:

- La première étape est la récupération des déploiements actuellement en cours sur le cluster en utilisant la tâche "Kubernetes@1". Elle se connecte au cluster via une "Kubernetes Service Connection", et utilise la commande "get" pour récupérer les déploiements dans le namespace "default". Les résultats sont retournés sous format "json".

- La deuxième étape utilise la tâche "CmdLine@2" pour effectuer des opérations shell sur l'instance Ubuntu utilisée par le pipeline. Elle télécharge le code de l'application à partir de Github, et récupère la version la plus récente de l'application publiée sur Docker Hub. Si la version actuellement de l'image voteapp déployée sur le cluster est différente de la version la plus récente, elle lance le déploiement de la nouvelle version.

- La dernière étape déploie la nouvelle version de l'application sur le cluster en utilisant la tâche "Kubernetes@1". Elle utilise la commande "apply" pour déployer la nouvelle version sur le cluster, en utilisant le fichier YAML mis à jour dans la tâche précédente.

Le pipeline est déclenché par une modification sur la branche "main", et est programmé pour s'exécuter toutes les heures grâce à une tâche programmée (cron: "0 * * * *").

```
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
- main

schedules:
- cron: "0 * * * *"
  displayName: repetition
  branches:
    include:
    - main
  always: true

pool:
  vmImage: ubuntu-latest

# Première étape

steps:
- task: Kubernetes@1
  inputs:
    connectionType: 'Kubernetes Service Connection'
    kubernetesServiceEndpoint: 'Nicolas-AKS-k8s'
    namespace: 'default'
    command: 'get'
    arguments: 'deployments'
    outputFormat: 'json'
  name: "kube"

# Deuxième étape

- task: CmdLine@2
  inputs:
    script: |
      versionnew=$((curl 'https://hub.docker.com/v2/repositories/simplonasa/azure_voting_app/tags' | jq '."results"[0]["name"]')| sed 's/^.//;s/.$//')
      versionold=$(echo $KUBE_KUBECTLOUTPUT | jq '.items[1].spec.template.spec.containers[].image' | cut -d: -f2 | sed 's/"//')
      echo "##vso[task.setvariable variable=vernew]$versionnew"
      echo "##vso[task.setvariable variable=verold]$versionold"
      cd /home/vsts
      git clone https://github.com/simplon-nicolasmarty/brief7.git
      cd brief7/k8s/
      sed -i 's/{{ version }}/'$versionnew'/g' vote-app.yml

  name: 'bashoter'

# Dernière étape

- task: Kubernetes@1
  condition: ne(variables['verold'],variables['vernew'])
  inputs:
    connectionType: 'Kubernetes Service Connection'
    kubernetesServiceEndpoint: 'Nicolas-AKS-k8s'
    command: 'apply'
    useConfigurationFile: true
    configuration: '/home/vsts/brief7/k8s/'
    arguments: ''
    secretType: 'generic'
    outputFormat: 'json'
```

![](https://i.imgur.com/IKOLrlI.png)

## Tableau d'évaluation des coûts Cloud associés : 

Evaluation réalisé avec la calculatrice d'évaluation Azure 
https://azure.microsoft.com/fr-fr/pricing/calculator

![](https://i.imgur.com/KTiZ0l5.png)
