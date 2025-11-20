
# Tutoriel : Répartition de Charge (Load Balancing) Interne avec Service ClusterIP

Ce tutoriel illustre le concept de **Load Balancing** opéré par un **Service Kubernetes** de type `ClusterIP` pour distribuer le trafic entre les instances d'une application (Pods Nginx) au sein du cluster.

## 1\. Concepts Clés

  * **Deployment** : Gère les répliques de l'application (les Pods Nginx).
  * **Service (ClusterIP)** : Crée une adresse IP virtuelle et un port stables, accessibles **uniquement** à l'intérieur du cluster. Ce Service gère la répartition de charge vers les Pods cibles.
  * **Répartition de Charge** : Le Service utilise le mécanisme de répartition par défaut de Kubernetes (souvent Round Robin, via `kube-proxy`) pour diriger les requêtes vers l'un des Pods disponibles de manière cyclique.

-----

## 2\. Étape 1 : Création du Projet Nginx Déployé (Deployment)

Nous créons un Deployment pour maintenir **trois répliques** (Pods) du serveur Nginx.

### A. Fichier de Déploiement (`nginx-deployment.yaml`)

Ce fichier demande à Kubernetes de maintenir trois copies du serveur Nginx, chacune affichant son nom d'hôte unique pour identifier quelle instance a traité la requête.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-load-balancer-deployment
  labels:
    app: nginx-backend
spec:
  replicas: 3 # Trois Pods pour démontrer la répartition
  selector:
    matchLabels:
      app: nginx-backend
  template:
    metadata:
      labels:
        app: nginx-backend
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
        # Commande qui modifie la page d'accueil pour afficher l'Hostname du Pod
        command: ["/bin/sh", "-c"]
        args: ["echo 'Serveur Nginx ID: $HOSTNAME' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]
```

### B. Application du Déploiement

```bash
# Appliquer le déploiement
kubectl apply -f nginx-deployment.yaml

# Vérifier les Pods (attendre que 3 Pods soient 'Running')
kubectl get pods -l app=nginx-backend
```

-----

## 3\. Étape 2 : Création du Service ClusterIP

Nous créons le Service de type `ClusterIP`. Ce service est le point d'entrée pour la répartition de charge et est **accessible uniquement depuis l'intérieur du cluster**.

### A. Fichier de Service (`nginx-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-balancer-service
spec:
  # Type de Service pour l'accès INTERNE au cluster
  type: ClusterIP 
  # Le selecteur lie ce service aux Pods avec le label 'app: nginx-backend'
  selector:
    app: nginx-backend 
  ports:
    - protocol: TCP
      port: 80 # Le Service écoute en interne sur le port 80
      targetPort: 80 # Le trafic est redirigé vers le port 80 des conteneurs Nginx
```

### B. Application du Service

```bash
# Appliquer le service
kubectl apply -f nginx-service.yaml

# Obtenir l'adresse ClusterIP du service (IP Virtuelle)
kubectl get service nginx-balancer-service
# Notez l'adresse IP dans la colonne CLUSTER-IP (ex: 10.96.0.150)
```

-----

## 4\. Étape 3 : Démonstration du Load Balancing Interne

Puisque le Service est de type `ClusterIP`, nous ne pouvons pas y accéder directement depuis l'extérieur. Nous devons créer un **Pod de test** à l'intérieur du cluster pour interroger le Service.

### A. Création et Accès au Pod de Test

1.  **Lancement du Pod de Test** (utilise une image simple comme `busybox` ou `ubuntu` avec `curl`):
    ```bash
    kubectl run test-client --image=ubuntu --rm -it -- bash
    ```
2.  **Installation de Curl** (à l'intérieur de la session bash du Pod) :
    ```bash
    apt update && apt install curl -y
    ```
    *Si le Pod `test-client` n'a pas accès à Internet, utilisez une image comme `nicolaka/netshoot` qui a déjà `curl`.*

### B. Test de la Répartition de Charge Interne (Loop Test)

À l'intérieur de la session du Pod de test, nous utilisons le **nom du Service** (qui est résolu par DNS interne en son `ClusterIP`) pour tester la répartition.

```bash
# Le nom du service est 'nginx-balancer-service'
for i in 1 2 3 4 5 6 7 8; do 
  echo "Requête $i : $(curl -s nginx-balancer-service)"
done
```

### C. Résultat Attendu

Vous devriez voir les réponses alterner entre les différents identifiants de Pods, prouvant que le Service `ClusterIP` agit bien comme un Load Balancer interne.

```
Requête 1 : Serveur Nginx ID: nginx-load-balancer-deployment-xxxx-1a2b
Requête 2 : Serveur Nginx ID: nginx-load-balancer-deployment-xxxx-5c6d
Requête 3 : Serveur Nginx ID: nginx-load-balancer-deployment-xxxx-7e8f
Requête 4 : Serveur Nginx ID: nginx-load-balancer-deployment-xxxx-1a2b
...
```

Ce processus confirme que le **Service** gère le Load Balancing des Pods cibles, rendant le système fiable et capable de gérer la charge en distribuant les requêtes.