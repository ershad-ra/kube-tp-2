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