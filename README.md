
# Déploiement d'application en utilisant Kubernetes.

Ce guide permet à déployer une API Node.js et un cache Redis en utilisant Kubernetes avec minikube




## Prérequis.

- Docker et minikube installés

    
## Etapes
### 1. Lancer et configurer minikube
1. Lancer minikube
```bash
minikube start 
```

2. verifier que docker client peut acceder à notre conteneur minikube 

verifier que l'adresse du docker engine dans les variables d'envs est bien localhost das notre cas
```bash
echo $(minikube docker-env)
```

pointer le shell au docker engine de minikube
```bash
eval $(minikube -p minikube docker-env)
```




### 2. Build les images du projet

#### API Image
1. Créer l'image de l'api:
```bash
docker build . -t api:v1 
```


#### Redis Image
1. Créer l'image du cache:
```bash
docker build . -t redis:v1 
```

### 2. Deployer l'API et Redis
#### Deploiement de l'api

1. créer un fichier de deployement ApiDeployment.yaml pour deployer deux instances:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ApiDeployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: api:v1
          env:
            - name: REDIS_URL
              value: redis://redis-service:6379
          ports:
            - containerPort: 1337
```

2. appliquer le deploiement:
```bash
kubectl apply -f ApiDeployment.yaml
```

3. exposer l'api vers l'exterieur avec le service NodePort
```bash
kubectl expose deployment ApiDeployment --type=NodePort --port=1337
```

4. Avoir la description du service
```bash
kubectl describe service ApiDeployment
```



#### Redis Deployment
1.Create a deployment YAML file named redis-deployment.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: RedisDeployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:v1
          ports:
            - containerPort: 6379
```
2. appliquer le deploiement:
```bash
kubectl apply -f RedisDeployment.yaml
```

3. exposer redis vers l'exterieur avec le service NodePort
```bash
kubectl expose deployment RedisDeployment --type=NodePort --port=6379
```

4. Avoir la description du service
```bash
kubectl describe service RedisDeployment
```

### 3. Exposer l'API Service avec NGINX Ingress
Au lieu d'exposer nos conteneurs avec NodePort , on peut le faire également avec NGINX Ingress , exemple pour l'api

1. créer api-service.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ApiService
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 1337
```
2. Create an Ingress YAML file named api-ingress.yaml:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ApiIngress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ApiService
                port:
                  number: 80
```

2. Apply the API Ingress resource to define the routing rules:

```bash
kubectl apply -f api-ingress.yaml
```
### 4. Verifier les Deploiements
1. Verifier les status des deploiements:
```bash
kubectl get deployments
```
2. Verifier les pods:
```bash
kubectl get pods
```

