Absolument. Voici le tutoriel complet et détaillé couvrant l'installation de **Nexus Repository Manager** et de **Nexus Lifecycle (IQ Server)**, la gestion des dépôts Docker, et la configuration des politiques de sécurité avec un exemple Python, tout en utilisant des installations basées sur `systemd`.

-----

## 🛠️ Tutoriel Complet : Nexus Repository Manager et Nexus Lifecycle (IQ Server)

Ce guide est basé sur des installations de services **systemd** sous Linux pour des environnements de production.

### Partie I : Installation de Nexus Repository Manager (NXRM) 📦

Nous installons NXRM 3 en tant que service pour gérer vos artefacts.

#### 1\. Prérequis et Installation

Assurez-vous que le **JDK 11 ou supérieur** est installé.

  * **Création de l'utilisateur :** Créez un utilisateur système pour exécuter Nexus :
    ```bash
    sudo useradd -r -m -d /opt/sonatype-work -s /bin/bash nexus
    ```
  * **Téléchargement et Déplacement :** Téléchargez l'archive NXRM et déplacez-la :
    ```bash
    wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
    sudo tar xvzf latest-unix.tar.gz -C /opt
    sudo mv /opt/nexus-* /opt/nexus
    ```
  * **Permissions :** Créez le répertoire de travail et ajustez les permissions pour l'utilisateur `nexus` :
    ```bash
    sudo chown -R nexus:nexus /opt/nexus /opt/sonatype-work
    ```

#### 2\. Configuration du Service systemd

  * **Création du fichier de service :**

    ```bash
    sudo nano /etc/systemd/system/nexus.service
    ```

  * **Configuration du service :** Collez le contenu ci-dessous. (Le port par défaut est `8081`).

    ```ini
    [Unit]
    Description=Nexus Repository Manager
    After=network.target

    [Service]
    User=nexus
    Group=nexus
    Type=simple
    ExecStart=/opt/nexus/bin/nexus start
    ExecStop=/opt/nexus/bin/nexus stop
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```

  * **Démarrage :**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start nexus
    sudo systemctl enable nexus # Activation au démarrage
    ```

  * Accédez à l'interface web : **`http://<Adresse_IP>:8081`**. Connectez-vous avec `admin` et le mot de passe récupéré dans `/opt/sonatype-work/nexus3/admin.password`.

-----

### Partie II : Configuration des Dépôts Docker dans NXRM 🐳

Nous allons créer un dépôt privé et un dépôt proxy pour Docker Hub.

#### 1\. Création des Dépôts

Accédez à **Server Administration** \> **Repositories** \> **Create Repository**.

  * **Dépôt Privé (Hosted)** :
      * Type : `docker (hosted)`
      * Nom : `docker-private`
      * Port HTTP : **`8082`**
      * Cochez **Force basic authentication**.
  * **Dépôt Cache (Proxy)** :
      * Type : `docker (proxy)`
      * Nom : `docker-hub-proxy`
      * Remote Storage : `https://registry-1.docker.io`
      * Port HTTP : **`8083`**

#### 2\. Configuration Firewall et Accès Docker

Vous devez ouvrir les ports **8082** et **8083** sur votre firewall.

  * **Configuration Docker Locale :** Sur la machine client (votre poste de travail ou serveur CI), vous devez autoriser l'accès non sécurisé à ces ports (si vous n'utilisez pas HTTPS) en modifiant `/etc/docker/daemon.json` :
    ```json
    {
      "insecure-registries": ["<Adresse_IP>:8082", "<Adresse_IP>:8083"]
    }
    ```
    Puis redémarrez le démon Docker.

#### 3\. Test et Utilisation

  * **Login :**
    ```bash
    docker login <Adresse_IP>:8082
    ```
  * **Push vers le Privé :**
    ```bash
    docker tag my-local-image <Adresse_IP>:8082/app:latest
    docker push <Adresse_IP>:8082/app:latest
    ```
  * **Pull depuis le Proxy :**
    ```bash
    docker pull <Adresse_IP>:8083/library/alpine:latest
    ```

-----

### Partie III : Installation de Nexus Lifecycle (IQ Server) 🔒

Nous installons l'IQ Server, le composant de DevSecOps, en tant que service.

#### 1\. Installation du Service systemd

  * **Création de l'utilisateur :**

    ```bash
    sudo useradd -r -m -d /opt/sonatype-work/iqserver -s /bin/bash nexus-iq
    ```

  * **Téléchargement et Déplacement :**

    ```bash
    wget https://download.sonatype.com/clm/server/latest.zip -O nexus-iq-server.zip
    sudo unzip nexus-iq-server.zip -d /opt
    sudo mv /opt/nexus-iq-server-* /opt/nexus-iq-server
    ```

  * **Permissions :**

    ```bash
    sudo chown -R nexus-iq:nexus-iq /opt/nexus-iq-server /opt/sonatype-work/iqserver
    ```

  * **Création du fichier de service (`/etc/systemd/system/nexus-iq.service`) :**

    ```ini
    [Unit]
    Description=Nexus IQ Server
    After=network.target

    [Service]
    User=nexus-iq
    Group=nexus-iq
    Type=simple
    ExecStart=/opt/nexus-iq-server/bin/nexus-iq-server
    Restart=always
    StandardOutput=journal
    StandardError=journal
    SuccessExitStatus=143

    [Install]
    WantedBy=multi-user.target
    ```

  * **Démarrage :**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start nexus-iq
    ```

  * Accédez à l'interface web : **`http://<Adresse_IP>:8070`**. Identifiants par défaut : `admin` / `admin123`.

-----

### Partie IV : Exemple DevSecOps Python avec Nexus Lifecycle 🐍

Nous allons configurer une politique et scanner une application Python simple.

#### 1\. Configuration de l'Application et de la Politique

1.  **Application :** Dans l'interface IQ Server, créez une Organisation et une Application avec l'ID : **`python-hello-world`**.
2.  **Politique (Exemple) :** Créez une politique nommée **`PyPI-Critical-Vulnerability-Block`** qui s'applique au format **PyPI**.
      * Action : **Fail** (Échec du scan).
      * Condition : `Security Vulnerability Severity` **is greater than or equal to** `9`.

#### 2\. Préparation du Projet Python

Nous utilisons un fichier `requirements.txt` avec une dépendance connue pour avoir une vulnérabilité critique (ex: `requests==2.6.0`).

  * Installez les dépendances :
    ```bash
    pip install -r requirements.txt
    ```

#### 3\. Analyse et Blocage

Nous utilisons le **Nexus IQ CLI** pour soumettre le **Software Bill of Materials (SBOM)** de notre projet.

  * **Téléchargement du CLI :** Récupérez le `nexus-iq-cli.jar` depuis **Administration** \> **System Preferences** \> **System Links** et placez-le dans un répertoire accessible.

  * **Exécution de l'Analyse :** Installez l'outil `pip-audit` pour générer le SBOM et soumettez-le :

    ```bash
    # 1. Génération du SBOM au format CycloneDX
    pip-audit --output-format cyclonedx-json --output bom.json

    # 2. Soumission à l'IQ Server via le CLI
    java -jar /chemin/vers/iq-cli.jar -s http://<Adresse_IP>:8070 -a admin -p <Votre_Mot_de_Passe> -i python-hello-world bom.json
    ```

  * **Résultat :** Le CLI retournera un **code de sortie non nul** et affichera la violation, car la politique a été déclenchée. Le pipeline de construction aurait été arrêté.

  * **Rapport :** Vous pouvez visualiser le rapport complet dans l'interface IQ Server, sous l'Application **`Python Hello World App`**, montrant la CVE et la recommandation de correction.

-----

### Partie V : Gestion des Accès et Sécurité (NXRM)

Il est essentiel d'utiliser des comptes non-administrateurs pour les opérations quotidiennes.

#### 1\. Création d'un Rôle

1.  Accédez à NXRM (**8081**) \> **Server Administration** \> **Security** \> **Roles**.
2.  Créez un rôle : **`docker-dev`**.
3.  Ajoutez les privilèges suivants pour gérer les dépôts Docker créés :
      * `nx-repository-view-docker-docker-private-add` (Push)
      * `nx-repository-view-docker-docker-private-read` (Pull)
      * `nx-repository-view-docker-docker-hub-proxy-read` (Pull depuis le Proxy)

#### 2\. Création d'un Utilisateur

1.  Accédez à **Server Administration** \> **Security** \> **Users**.
2.  Créez un utilisateur : **`ci-service`** avec un mot de passe fort.
3.  Dans la section **Roles**, ajoutez le rôle **`docker-dev`** (et assurez-vous de retirer tout rôle administratif).

Cet utilisateur peut maintenant se connecter aux registres Docker (ports 8082/8083) pour les opérations de CI/CD sans avoir accès aux paramètres d'administration.