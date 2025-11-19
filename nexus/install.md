
-----

## Installation de Nexus Repository sur Ubuntu 24.04 

### 0\. Préparation et Initialisation des Variables 

> **⚠️ ATTENTION :** **Vous devez vérifier et remplacer** `3.86.0-08` par le numéro de la toute dernière version disponible sur le site de Sonatype.

```bash
# Variable pour la version (à remplacer par la dernière version)
NEXUS_VERSION="3.86.0-08"

# Variable pour le nom du répertoire extrait
NEXUS_VERSION_DIR="nexus-${NEXUS_VERSION}"

# Variable pour le nom de l'archive téléchargée
NEXUS_ARCHIVE="nexus-${NEXUS_VERSION}-linux-x86_64.tar.gz"
```

-----

### Étape 1 : Prérequis et Utilisateur Dédié 

1.  **Installation de Java (OpenJDK 17) :**
    ```bash
    sudo apt update
    sudo apt install openjdk-17-jdk -y
    ```
2.  **Création de l'utilisateur `nexus` :**
    ```bash
    sudo useradd -r -m -d /opt/nexus -s /bin/bash nexus
    ```

-----

### Étape 2 : Téléchargement et Organisation des Fichiers 

1.  **Téléchargement et Extraction :**
    *Ceci extrait les fichiers d'application (ex: `nexus-3.86.0-08`) et de données (`sonatype-work`) sous `/opt/nexus/`.*
    ```bash
    cd /tmp
    wget "https://download.sonatype.com/nexus/3/${NEXUS_ARCHIVE}"
    sudo mkdir -p /opt/nexus
    sudo tar xzf "${NEXUS_ARCHIVE}" -C /opt/nexus/
    ```
2.  **Création du Lien Symbolique :**
    *Permet d'utiliser un chemin stable (`nexus-current`) pour les mises à jour et le service `systemd`.*
    ```bash
    sudo ln -s /opt/nexus/$NEXUS_VERSION_DIR /opt/nexus/nexus-current
    ```
3.  **Application des Permissions :**
    *La propriété est donnée à l'utilisateur `nexus` pour l'ensemble des répertoires d'application et de données (`sonatype-work`).*
    ```bash
    sudo chown -R nexus:nexus /opt/nexus
    ```

-----

### Étape 3 : Configuration du Service `systemd`

1.  **Création du fichier de service `systemd` :**
    ```bash
    sudo nano /etc/systemd/system/nexus.service
    ```
2.  **Contenu du fichier `nexus.service` :**
    ```ini
    [Unit]
    Description=Sonatype Nexus Repository Manager
    After=network.target

    [Service]
    Type=forking
    LimitNOFILE=65536
    # Utilisation du chemin stable via le lien symbolique
    ExecStart=/opt/nexus/nexus-current/bin/nexus start
    ExecStop=/opt/nexus/nexus-current/bin/nexus stop
    # DEFINITION CRITIQUE DE L'UTILISATEUR NON-ROOT
    User=nexus
    Restart=on-abort

    [Install]
    WantedBy=multi-user.target
    ```

-----

### Étape 4 : Démarrage du Service et Accès

1.  **Rechargement et Démarrage :**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable nexus.service
    sudo systemctl start nexus.service
    ```
2.  **Vérification de l'état :**
    ```bash
    sudo systemctl status nexus.service
    ```
3.  **Première Connexion (après quelques minutes de démarrage) :**
      * **Accès Web :** `http://VOTRE_IP_SERVEUR:8081`
      * **Identifiant :** `admin`
      * **Mot de passe initial :** Récupérez le mot de passe temporaire en utilisant le chemin par défaut de l'application :
        ```bash
        sudo cat /opt/nexus/sonatype-work/nexus3/admin.password
        ```
      * Suivez les étapes dans l'interface web pour définir un nouveau mot de passe.

<!-- end list -->

```
```
