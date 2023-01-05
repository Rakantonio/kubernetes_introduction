# Kubernetes
## Installer Minikube et Kubectl
### Installer minikube
```bash=
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
sudo usermod -aG docker kube && newgrp docker
minikube start
```
### Installer kubectl
```bash=
$ minikube kubectl -- get pods -A
$ alias kubectl="minikube kubectl --"
```

## Pod nginx
### Héberger un premier Pod Nginx
* On peut lancer en ligne de commande
```bash=
$ kubectl run my-nginx --image=nginx  --port=80
```
* On peut également créer un fichier de déploiement nginx-deployment.yml
```
$ mkdir tp1/deployment
$ vim tp1/deployment/nginx-deployment.yml
```
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
* Créer le déploiement en indiquant le fichier de ressource
```bash=
$ kubectl apply -f tp1/nginx-deployment.yaml
```
* vérifier que le pod est prêt
```bash=
$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7fb96c846b-m76hf   1/1     Running   0          3m49s
```

### A l’aide de la commande kubectl port-forward et d’un navigateur accéder à la page par défaut de votre pod Nginx
```bash=
$ kubectl port-forward nginx-deployment-7fb96c846b-m76hf --address=192.168.233.130 8080:80
Forwarding from 192.168.233.130:8080 -> 80
```
![](https://i.imgur.com/iSnZe7M.png)

## Connexion entre plusieurs Pods
### A l’image du TP 1 sur Docker (question 7 et 8), héberger un Pod phpmyadmin et mysql, cette fois-ci en utilisant minikube

* Créer un fichier de déploiement mysql
```yaml=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: P@ssw0rd
        - name: MYSQL_USER
          value: toto
        - name: MYSQL_PASSWORD
          value: P@ssw0rd
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
```
* Lancer le déploiement.
```bash=
$ kubectl apply -f mysql-deployment.yaml
```
* Créer un fichier de déploiement phpmyadmin
```yaml=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin-deployment
  labels:
    app: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
        - name: phpmyadmin
          image: phpmyadmin/phpmyadmin
          ports:
            - containerPort: 80
          env:
            - name: PMA_HOST
              value: mysql
            - name: PMA_PORT
              value: "3306"
            - name: MYSQL_ROOT_PASSWORD
              value: P@ssw0rd
            - name: MYSQL_USER
              value: toto
            - name: MYSQL_PASSWORD
              value: P@ssw0rd
```
* Lancer le déploiement.
```bash=
$ kubectl apply -f phpmyadmin-deployment.yaml
```
* Vérifier que les 2 déploiements sont prêts
```bash=
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
mysql-746ff99494-rh2zv                  1/1     Running   0          108s
phpmyadmin-deployment-f78d696ff-t6p52   1/1     Running   0          5m45s
$ kubectl get deployments
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
mysql                   1/1     1            1           114s
phpmyadmin-deployment   1/1     1            1           5m51s
```

### Créer un service associé au Pod mysql
* Créer un fichier db_mysql.yaml qui sera le service communicant avec le déploiement.
```yaml=
---
apiVersion: v1
kind: Service    
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
```
* Lancer le déploiement du service
```bash=
$ kubectl apply -f db_service.yaml
```

### Connecter phpmyadmin avec le Service mysql et vérifier que vous pouvez administrer cette base de données
* Créer un fichier db_mysql.yaml qui sera le service communicant avec le déploiement.
```yaml=
---
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-service
spec:
  type: NodePort
  selector:
    app: phpmyadmin
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```
* Lancer le déploiement du service
```bash=
$ kubectl apply -f pma-service.yaml
```

### Avec la commande kubectl-port forward, vérifier que phpmyadmin arrive à contacter et administrer votre base de données mysql
* Créer une base de données en ajoutant l'environnement MYSQL_DATABASE dans le fichier mysql-deployment.yaml
```yaml=
env:
  - name: MYSQL_DATABASE
    value: ynov
```
* Appliquer les configurations pour le pod mysql
```bash=
$ kubectl apply -f mysql-deployment.yaml
```
* Vérifier que phpmyadmin se connecte à la base de données avec l'utilisateur "toto"
```bash=
$ kubectl port-forward phpmyadmin-deployment-f78d696ff-t6p52 --address=192.168.233.130 8080:80
Forwarding from 192.168.233.130:8080 -> 80
```
![](https://i.imgur.com/cl9HNzE.png)
![](https://i.imgur.com/B3o08CV.png)

### Ajouter un Ingress pour accéder à phpmyadmin sans utiliser la commande kubectl port-forward
* Créer un fichier ingress.yaml
```yaml=
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: phpmyadmin-http-ingress
spec:
  rules:
  - host: phpmyadmin.local
  - http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: phpmyadmin-service
            port:
              number: 80
```
* Activer ingress
```bash=
$ minikube addons enable ingress
```
* Créer un DNS pour le host: phpmyadmin.local
```bash=
$sudo vim /etc/hosts
192.168.233.130 phpmyadmin.local
192.168.49.2 phpmyadmin.local
```
> L'IP de l'ingress s'obtient en faisant la commande suivante: 
```bash
$ kubectl get ingress
NAME                      CLASS   HOSTS              ADDRESS        PORTS   AGE
phpmyadmin-http-ingress   nginx   phpmyadmin.local   192.168.49.2   80      6m2s
```

