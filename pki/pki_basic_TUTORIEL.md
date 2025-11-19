
-----

# Tutoriel : PKI et la Chaîne de Confiance

Ce guide détaille la mise en place d'une Autorité de Certification (CA) locale et valide la chaîne de confiance.

## 1\. Structure du Lab et Conventions

  * **`client.local`** (192.168.56.101) : Machine cliente pour les tests.
  * **`app.local`** (192.168.56.22) : Serveur Web Apache.
  * **`root-ca.local`** (192.168.56.21) : Autorité de Certification.

## 2\. Configuration Initiale (Sur TOUTES les VMs)

### 2.1. Installation des outils et Répertoires

Sur **toutes les VMs** (`root-ca`, `app.local`, `client`) :

```bash
sudo apt update
sudo apt install openssl -y
sudo mkdir -p /etc/pki-local/
sudo chown -R root:root /etc/pki-local
```

### 2.2. Configuration de la Résolution de Noms

Sur **toutes les VMs** (`root-ca`, `app.local`, `client`), modifiez `/etc/hosts` :

```bash
sudo nano /etc/hosts
# Ajoutez les lignes suivantes
192.168.56.21 root-ca.local
192.168.56.22 app.local
192.168.56.101 client.local
```

### 2.3. Installation d'Apache (Sur `app.local`)

Sur la VM **`app.local`** uniquement :

```bash
sudo apt install apache2 -y
```

-----

## 3\. Observation des Erreurs Initiales (Certificat Auto-Signé)

### 3.1. Activation de SSL par Défaut (Sur `app.local`)

Sur la VM **`app.local`** :

```bash
sudo a2enmod ssl
sudo a2enmod headers
sudo a2enmod rewrite
sudo a2ensite default-ssl.conf
sudo systemctl restart apache2
```

### 3.2. Vérification des Erreurs (Sur `client.local`)

Sur la VM **`client.local`** :

1.  **Vérification avec curl :**

    ```bash
    curl https://app.local
    ```

    **Résultat :** Échec (`SSL certificate problem: self-signed certificate`).

2.  **Vérification avec Firefox :** Accès à `https://app.local`.
    **Résultat :** Avertissement de sécurité (`SEC_ERROR_UNKNOWN_ISSUER`).

-----

## 4\. Mise en Place de l'Autorité de Certification (CA)

### 4.1. Création des Clés et du Certificat CA (Sur `root-ca.local`)

Sur la VM **`root-ca.local`** :

```bash
mkdir -p ~/pki_tmp
cd ~/pki_tmp

# 1. Génération de la clé privée de la CA
openssl genrsa -out ca.key 4096

# 2. Création du certificat Racine
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=FR/ST=Marseille/L=Aix/O=Lab_PKI/CN=root-ca.local"

# 3. Déplacement Sécurisé et Permissions
sudo mv ca.key /etc/pki-local/
sudo chown root:root /etc/pki-local/ca.key
sudo chmod 400 /etc/pki-local/ca.key

sudo mv ca.crt /etc/pki-local/
sudo chown root:root /etc/pki-local/ca.crt
sudo chmod 644 /etc/pki-local/ca.crt
```

-----

## 5\. Génération, Signature et Correction SAN

### 5.1. Création de la Clé, du CNF et du CSR (Sur `app.local`)

Sur la VM **`app.local`** :

```bash
export SERVER_NAME="app.local"
mkdir -p ~/pki_tmp
cd ~/pki_tmp
```

**Ouvrir l'éditeur pour le fichier de configuration :**

```bash
nano app_local.cnf
```

**Contenu à coller dans `app_local.cnf` (utilisez Ctrl+X, Y pour sauvegarder et quitter) :**

```ini
[ req ]
distinguished_name = req_distinguished_name
req_extensions = req_ext
prompt = no

[ req_distinguished_name ]
C = FR
ST = Marseille
L = Aix
O = Lab_PKI
CN = app.local

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = app.local
IP.1 = 192.168.56.22
```

**Fin du contenu du fichier**

```bash
# 2. Génération de la nouvelle clé privée
openssl genrsa -out app.local.key 2048

# 3. Création du CSR avec le fichier de configuration SAN
openssl req -new -key app.local.key -out app.local.csr -config app_local.cnf

# 4. Déplacement Sécurisé de la Clé Privée
sudo mv app.local.key /etc/pki-local/
sudo chown root:ssl-cert /etc/pki-local/app.local.key
sudo chmod 640 /etc/pki-local/app.local.key

# 5. Transfert du CSR et du CNF vers la CA
scp app.local.csr useradm@root-ca.local:/tmp/
scp app_local.cnf useradm@root-ca.local:/tmp/
```

### 5.2. Signature du Certificat Corrigé (Sur `root-ca.local`)

Sur la VM **`root-ca.local`** :

```bash
export SERVER_NAME="app.local"
cd /etc/pki-local/

# Signature avec inclusion des extensions SAN (req_ext)
sudo openssl x509 -req -in /tmp/app.local.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out app.local.crt -days 730 -sha256 -extensions req_ext -extfile /tmp/app_local.cnf

# Nettoyage
sudo rm /tmp/app_local.cnf

# Permissions finales
sudo chown root:root app.local.crt
sudo chmod 644 /etc/pki-local/app.local.crt
```

### 5.3. Retour du Certificat et Distribution de la CA

**Sur `root-ca.local`** (transfert des fichiers) :

```bash
export SERVER_NAME="app.local"
# Retour du Certificat signé
scp /etc/pki-local/app.local.crt useradm@app.local:/tmp/

# Distribution du Certificat Racine (ca.crt) vers toutes les VMs clientes
scp /etc/pki-local/ca.crt useradm@app.local:/tmp/
scp /etc/pki-local/ca.crt useradm@client.local:/tmp/
```

**Sur `app.local`** (déplacement du certificat serveur) :

```bash
sudo mv /tmp/app.local.crt /etc/pki-local/
```

-----

## 6\. Activation de la Confiance et Configuration HTTPS

### 6.1. Établissement de la Confiance Globale (Sur TOUTES les VMs)

Sur **toutes les VMs** (`root-ca`, `app.local`, `client`) :

```bash
# 1. Copie du ca.crt vers le magasin de confiance
sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/lab_ca.crt

# 2. Mise à jour du magasin de certificats, intégrant notre CA
sudo update-ca-certificates
```

### 6.2. Configuration Finale d'Apache (Sur `app.local`)

Sur la VM **`app.local`** :

1.  **Configuration du Virtual Host SSL :**
    Vérifiez et modifiez `/etc/apache2/sites-available/default-ssl.conf` pour qu'il pointe vers les certificats signés.
    Vous devez y spécifier les directives SSL suivantes :

```apache
<VirtualHost *:443>
    # ... autres directives de Virtual Host ...

    SSLEngine on
    SSLCertificateFile    /etc/pki-local/app.local.crt
    SSLCertificateKeyFile /etc/pki-local/app.local.key

    # ...
</VirtualHost>
```


2.  **Redirection HTTP vers HTTPS :**
    Vérifiez et modifiez `/etc/apache2/sites-available/000-default.conf` :

    ```apache
    <VirtualHost *:80>
        ServerName app.local
        Redirect permanent / https://app.local/
    </VirtualHost>
    ```

3.  **Redémarrage Final :**

    ```bash
    sudo apachectl configtest
    # Si "Syntax OK" :
    sudo systemctl restart apache2
    ```

-----

## 7\. Vérification Finale du Succès (Sur `client.local`)

Sur la **VM client.local**, redémarrez votre navigateur et effectuez les tests :

1.  **Vérification avec curl :**

    ```bash
    curl https://app.local
    ```

    **Résultat :** Succès.

2.  **Vérification avec Firefox :**
    Accédez à : `https://app.local`.
    **Résultat :** La connexion est établie **sans aucun avertissement de sécurité**.

