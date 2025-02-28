# Kube TP 2 - `Kubernetes Ingress`

## Etape 1. Installer `Kind` et créer votre premier cluster 
Suivre cette documentation : https://kind.sigs.k8s.io/docs/user/quick-start/

- For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

## Etape 2.  Installer le Nginx ingress Controller 
Suivre cette documentation : https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx 

- Créer un cluster dans kind :
```bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```
- `cat <<EOF ... EOF` → C’est une commande Bash qui crée un fichier temporaire contenant la configuration.
- `kind create cluster --config=-` →
    - `kind create cluster` → Crée un cluster Kubernetes dans Docker avec Kind.
    - `--config=-` → Charge la configuration directement depuis stdin (entrée standard) au lieu d'un fichier.
- `kind: Cluster` → Spécifie que nous créons un cluster Kind.
- `apiVersion: kind.x-k8s.io/v1alpha4` → Version de l'API utilisée pour définir le cluster.
- `nodes:` → Définit les nœuds du cluster.
- `role: control-plane` → Ce nœud joue le rôle de control-plane (maître du cluster).
📌 Kind crée un cluster "single-node" où ce nœud gère tout (master + worker).

- `extraPortMappings:` → Cette section mappe les ports du conteneur Docker du cluster vers l’hôte (localhost).
- `containerPort: 80 `→ Le port 80 dans le conteneur Kind (Ingress) sera exposé.
- `hostPort: 80` → Ce port sera redirigé vers le port 80 de la machine hôte.
- `protocol: TCP` → Protocole utilisé (TCP).
📌 Pourquoi cette configuration ?
Permet d’accéder aux sites (`monbonlait.fr`,` mesbonslegumes.fr`) via localhost sans préciser de port spécifique (`http://monbonlait.fr` au lieu de` http://localhost:port`).

## Etape 3. Compléter le schéma avec des objets Kubernetes
![alt text](image.png)


- Dans Kubernetes, les Pods sont éphémères.

- Chaque fois qu’un Pod est redémarré, il obtient une nouvelle adresse IP.
Sans un Service, on ne peut pas avoir une adresse stable pour notre application.

- Un Service crée une adresse IP fixe qui redirige le trafic vers les Pods.
- Il permet le Load Balancing entre plusieurs Pods (utile pour monbonlait.fr qui doit gérer plus de trafic).
- Il est obligatoire pour que l'Ingress puisse router les requêtes vers les bons Pods.

- Un ReplicaSet est responsable uniquement de maintenir un certain nombre de pods en fonctionnement.
- Mais il ne permet pas de gérer les mises à jour ou les changements de version facilement.

## Etape 4. Nous allons créer trois images Docker basées sur NGINX
- Chacune contenant une page HTML personnalisée pour chaque site:
`monbonlait.fr` (Magasin de lait)
`mesbonslegumes.fr` (Magasin de légumes)
`mesbonslegumes.fr/bio` (Magasin de légumes bio)
Ces images seront ensuite publiées sur Docker Hub.

1️⃣ Créer les fichiers pour chaque site
Dans un répertoire de travail, créez trois dossiers:
```bash
mkdir -p monbonlait mesbonslegumes mesbonslegumesbio

```
Dans chacun, ajoutez un fichier index.html pour la page d'accueil.

🔹` monbonlait/index.html`
```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Mon Bon Lait</title>
</head>
<body>
    <h1>Bienvenue sur MonBonLait.fr</h1>
    <p>Ici, nous vendons du lait frais directement des producteurs.</p>
</body>
</html>

```
🔹 `mesbonslegumes/index.html`

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Mes Bons Légumes</title>
</head>
<body>
    <h1>Bienvenue sur MesBonsLegumes.fr</h1>
    <p>Des légumes frais livrés directement de nos fermes.</p>
</body>
</html>
```
🔹 `mesbonslegumesbio/index.html`
```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Mes Bons Légumes Bio</title>
</head>
<body>
    <h1>Bienvenue sur MesBonsLegumes.fr/bio</h1>
    <p>Découvrez notre gamme de légumes bio, cultivés avec soin.</p>
</body>
</html>

```
2️⃣ Créer un Dockerfile pour chaque site
Dans chaque dossier, créez un fichier Dockerfile contenant:

`Dockerfile`
```dockerfile
# Utiliser NGINX comme base
FROM nginx:latest

# Copier le fichier index.html dans le dossier par défaut de NGINX
COPY index.html /usr/share/nginx/html/index.html

# Exposer le port 80
EXPOSE 80

```
3️⃣ Construire et publier les images sur Docker Hub
🔹 Se connecter à Docker Hub (si ce n'est pas déjà fait)
```bash
docker login
```
🔹 Remplacez votre_utilisateur par votre identifiant Docker Hub
Construisez et poussez les images une par une:

🌍 `MonBonLait`
```bash
cd monbonlait
docker build -t votre_utilisateur/monbonlait:latest .
docker push votre_utilisateur/monbonlait:latest
cd ..
```

🥦 `MesBonsLegumes`

```bash
cd mesbonslegumes
docker build -t votre_utilisateur/mesbonslegumes:latest .
docker push votre_utilisateur/mesbonslegumes:latest
cd ..
```
🍃 `MesBonsLegumesBio`
```bash
cd mesbonslegumesbio
docker build -t votre_utilisateur/mesbonslegumesbio:latest .
docker push votre_utilisateur/mesbonslegumesbio:latest
cd ..
```
4️⃣ Vérifier sur Docker Hub
Allez sur hub.docker.com et vérifiez que les trois images ont bien été publiées.

## Étape 5. Déployer les images Docker dans Kubernetes avec Kind

Maintenant que nous avons créé et publié nos images Docker, nous allons déployer les sites web sur Kubernetes en utilisant:

- `Deployment` pour gérer les Pods et assurer leur disponibilité.
- `Service` pour exposer les sites au réseau interne du cluster.
- `Ingress` pour gérer les requêtes HTTP et rediriger vers les bons services.

1️⃣ Déployer monbonlait.fr
Créons un fichier `monbonlait-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monbonlait-deployment
  labels:
    app: monbonlait
spec:
  replicas: 3
  selector:
    matchLabels:
      app: monbonlait
  template:
    metadata:
      labels:
        app: monbonlait
    spec:
      containers:
      - name: monbonlait
        image: votre_utilisateur/monbonlait:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: monbonlait-service
spec:
  selector:
    app: monbonlait
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Appliquez-le :

```bash
kubectl apply -f monbonlait-deployment.yaml
```
2️⃣ Déployer mesbonslegumes.fr

Créons un fichier `mesbonslegumes-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mesbonslegumes-deployment
  labels:
    app: mesbonslegumes
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mesbonslegumes
  template:
    metadata:
      labels:
        app: mesbonslegumes
    spec:
      containers:
      - name: mesbonslegumes
        image: votre_utilisateur/mesbonslegumes:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mesbonslegumes-service
spec:
  selector:
    app: mesbonslegumes
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Appliquez-le :

```bash
kubectl apply -f mesbonslegumes-deployment.yaml
```
3️⃣ Déployer mesbonslegumes.fr/bio
Créons un fichier` mesbonslegumesbio-deployment.yaml `:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mesbonslegumesbio-deployment
  labels:
    app: mesbonslegumesbio
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mesbonslegumesbio
  template:
    metadata:
      labels:
        app: mesbonslegumesbio
    spec:
      containers:
      - name: mesbonslegumesbio
        image: votre_utilisateur/mesbonslegumesbio:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mesbonslegumesbio-service
spec:
  selector:
    app: mesbonslegumesbio
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Appliquez-le :

```bash
kubectl apply -f mesbonslegumesbio-deployment.yaml
```

4️⃣ Vérifier les déploiements
Vérifiez si tout fonctionne :

```bash
kubectl get pods
kubectl get services
```
Vous devriez voir 3 Deployments, 3 Services, et des Pods en cours d'exécution.

5️⃣ Configurer Ingress pour gérer les requêtes HTTP

Créons un fichier `ingress.yaml `:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: monbonlait.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monbonlait-service
            port:
              number: 80
  - host: mesbonslegumes.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mesbonslegumes-service
            port:
              number: 80
  - host: mesbonslegumes.fr
    http:
      paths:
      - path: /bio
        pathType: Prefix
        backend:
          service:
            name: mesbonslegumesbio-service
            port:
              number: 80
```
Appliquez-le :

```bash
kubectl apply -f ingress.yaml
```
6️⃣ Tester les accès

Ajoutez les entrées suivantes dans votre /etc/hosts pour simuler un DNS local :

```bash
sudo nano /etc/hosts
```
Ajoutez :
```bash
127.0.0.1 monbonlait.fr
127.0.0.1 mesbonslegumes.fr
```
Enregistrez et quittez.

Puis testez :

```bash
curl http://monbonlait.fr
curl http://mesbonslegumes.fr
curl http://mesbonslegumes.fr/bio

```
![alt text](image-3.png)

![alt text](image-1.png)

![alt text](image-2.png)

