
# Tutoriel : Scanner des Images Docker avec Trivy

Ce tutoriel vous guidera à travers l'installation de **Trivy** et son utilisation pour scanner des images Docker, des systèmes de fichiers locaux et des dépôts Git afin de détecter des vulnérabilités.

## 1\. Installation de Trivy

Nous allons installer Trivy sur un système basé sur Debian/Ubuntu en utilisant le dépôt officiel d'Aqua Security.

1.  **Télécharger la clé GPG et ajouter le dépôt :**
    ```bash
    sudo apt-get install wget apt-transport-https gnupg lsb-release
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
    echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
    ```
2.  **Installer Trivy :**
    ```bash
    sudo apt-get update
    sudo apt-get install trivy -y
    ```
3.  **Vérifier l'installation :**
    ```bash
    trivy -v
    ```

## 2\. Scan d'une Image Docker

L'analyse d'images de conteneurs est l'usage principal de Trivy pour détecter les vulnérabilités dans le système d'exploitation et les dépendances.

### A. Scanner une Image Inconnue (À risque)

Nous analysons une ancienne image connue pour contenir des vulnérabilités.

1.  **Lancer le scan :**

    ```bash
    trivy image ubuntu:18.04
    ```

2.  **Interprétation du Résultat :** Trivy télécharge sa base de données de vulnérabilités, puis analyse l'image. Il affiche les résultats classés par gravité (**LOW, MEDIUM, HIGH, CRITICAL**), indiquant les paquets affectés et la version de correction (`Fixed Version`).

### B. Scanner une Image Minimaliste (Bonne Pratique)

Pour illustrer le concept DevSecOps de réduction de la surface d'attaque, nous scannons une image minimaliste.

```bash
trivy image alpine:latest
```

Le résultat montrera que l'image **Alpine** contient généralement beaucoup moins de vulnérabilités critiques que des images plus lourdes comme Ubuntu, validant la pratique d'utiliser des images de base minimalistes.

## 3\. Scan d'un Système de Fichiers Local

Trivy peut analyser des répertoires locaux à la recherche de vulnérabilités dans les dépendances de langages ou les fichiers de configuration.

### A. Préparation (Simulation)

Nous simulons un petit projet avec un fichier de dépendances Node.js.

```bash
mkdir trivy_test
cd trivy_test

# Crée un fichier de dépendances Node.js (package.json)
echo '{ "dependencies": { "express": "4.15.2" } }' > package.json
```

### B. Lancer le Scan du Fichier

```bash
trivy fs .
```

Trivy détectera les dépendances spécifiées (`express` version `4.15.2`) et rapportera toutes les vulnérabilités connues liées à cette version de la bibliothèque.

## 4\. Filtrage et Options Avancées (CI/CD)

Pour l'intégration dans un pipeline (CI/CD), le filtrage est crucial.

### A. Filtrer par Gravité

Vous pouvez ajuster le niveau de détail du rapport :

```bash
# N'afficher que les vulnérabilités de gravité ÉLEVÉE et CRITIQUE
trivy image --severity HIGH,CRITICAL ubuntu:18.04
```

### B. Mettre en Échec le Pipeline

Cette commande est essentielle en CI/CD. Elle force Trivy à renvoyer un **code d'erreur (`exit code 1`)** si des vulnérabilités critiques sont trouvées, interrompant ainsi le processus de *build* ou de déploiement.

```bash
# Échoue si une vulnérabilité CRITIQUE est détectée
trivy image --exit-code 1 --severity CRITICAL ubuntu:18.04
```