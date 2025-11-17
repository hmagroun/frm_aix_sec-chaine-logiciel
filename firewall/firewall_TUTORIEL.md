-----

# Tutoriel : Protection des Services Serveur avec UFW

Ce tutoriel utilise **UFW (Uncomplicated Firewall)** sur Ubuntu 24.04 pour sécuriser un serveur, en restreignant l'accès aux services nécessaires (HTTP, HTTPS) tout en bloquant d'autres services (FTP). Le nom de la machine cible est **`serveur`**.

## 1\. Structure du Lab et Configuration Initiale

Nous utilisons deux VMs Ubuntu 24.04 : `desktop` (client) et `serveur` (machine à protéger).

### 1.1. Installation des Services (Sur `serveur`)

Sur la VM **`serveur`**, installez Apache, VSFTPD et les outils SSL :

```bash
sudo apt update
sudo apt install apache2 vsftpd openssl -y
```

### 1.2. Configuration SSL (HTTPS) (Sur `serveur`)

Pour activer le port **443** d'Apache :

```bash
sudo a2enmod ssl
sudo a2ensite default-ssl.conf

# Redémarrer Apache pour appliquer les changements
sudo systemctl restart apache2
```

### 1.3. Vérification des Ports Ouverts (Avant Firewall)

Sur la VM **`serveur`**, utilisez `ss` pour lister les ports d'écoute TCP. Vous devriez voir les ports **80** (HTTP), **443** (HTTPS) et **21** (FTP) en écoute.

```bash
sudo ss -ntulp | grep -E "apache2|vsftpd|LISTEN"
```

-----

## 2\. Étape Initiale : Test d'Accès Complet (Non Sécurisé)

Cette étape montre que tous les services sont accessibles par défaut.

### 2.1. Scan des Ports (Sur `desktop`)

Pour identifier les services exposés, lancez un scan :

```bash
sudo apt install nmap -y
nmap serveur
```

**Résultat :** Le scan identifie clairement les ports **80**, **443** et **21** comme **ouverts**.

### 2.2. Test des Accès (Sur `desktop`)

  * **Accès HTTP (80)** :

    ```bash
    curl http://serveur
    ```

    **Résultat Attendu :** OK (Contenu HTML affiché).

  * **Accès HTTPS (443)** :

    ```bash
    curl -k https://serveur
    ```

    **Résultat Attendu :** OK (Contenu HTML affiché, l'option `-k` ignore l'avertissement du certificat).

  * **Accès FTP (21)** :

    ```bash
    nmap -p 21 serveur
    ```

    **Résultat Attendu :** OK (Port 21 est **ouvert**).

-----

## 3\. Mise en Place du Pare-feu UFW

Nous allons configurer UFW pour autoriser uniquement les services Web et SSH.

### 3.1. Définition des Règles (Sur `serveur`)

Sur la VM **`serveur`**, définissez la politique par défaut sur "Refuser" (Deny) et autorisez les ports nécessaires :

```bash
# Politique par défaut : tout refuser en entrée
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser les connexions administratives (SSH sur le port 22)
sudo ufw allow ssh

# Autoriser le service Web non sécurisé (HTTP sur le port 80)
sudo ufw allow http

# Autoriser le service Web Sécurisé (HTTPS sur le port 443)
sudo ufw allow https
```

### 3.2. Activation et Illustration IPTABLES (Sur `serveur`)

Activez le pare-feu :

```bash
# Activation du pare-feu
sudo ufw enable

# Vérification du statut et des règles UFW
sudo ufw status verbose
```

**Résultat :** Le statut doit être **actif** et les règles `ALLOW` doivent apparaître pour `SSH`, `HTTP` et `HTTPS`.

Pour voir comment UFW a modifié les règles du moteur **IPTABLES** :

```bash
# Afficher les règles de la chaîne INPUT gérées par UFW
sudo iptables -L ufw-before-input -n
```

**Résultat :** Les règles IPTABLES devraient refléter les directives d'autorisation pour les ports 22, 80 et 443, et la règle finale de rejet pour tout le reste.

-----

## 4\. Test d'Accès Restreint (Sécurisé)

Cette étape vérifie que les règles de filtrage sont appliquées.

### 4.1. Scan du Pirate (Sur `desktop`)

Lancez à nouveau le scan `nmap` :

```bash
nmap serveur
```

**Résultat :** Le scan identifie uniquement les ports **80** et **443** comme **ouverts**. Le port **21 (ftp)** devrait apparaître comme **filtré/fermé**.

### 4.2. Test des Accès Restreints (Sur `desktop`)

  * **Accès HTTP (80)** :

    ```bash
    curl http://serveur
    ```

    **Résultat Attendu :** OK. L'accès est maintenu.

  * **Accès HTTPS (443)** :

    ```bash
    curl -k https://serveur
    ```

    **Résultat Attendu :** OK. L'accès est maintenu.

  * **Accès FTP (21)** :

    ```bash
    nmap -p 21 serveur
    ```

    **Résultat Attendu :** Échec (Port 21 est **filtré**). La tentative de connexion est bloquée.

## Conclusion

En appliquant une politique restrictive, la surface d'attaque du serveur est réduite aux seuls services Web autorisés.

**Pour désactiver UFW à la fin du lab (Sur `serveur`) :**

```bash
sudo ufw disable
```
