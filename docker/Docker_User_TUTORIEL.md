
# Tutoriel : Exécution d'Application Docker avec Utilisateur Non-Root (Moindre Privilège)

Ce tutoriel démontre comment empêcher une application conteneurisée d'opérer avec les privilèges `root` (administrateur), réduisant ainsi le risque d'escalade en cas de compromission.

-----

## 1\. Préparation du Projet et de l'Application de Test

Nous créons une application Python simple dont le rôle est de vérifier son identité à l'intérieur du conteneur et de tenter une action qui nécessite des privilèges élevés (écriture dans le répertoire système `/root`).

### A. Initialisation

```bash
mkdir docker-privileges-demo
cd docker-privileges-demo
```

### B. Création de l'Application (`app.py`)

```bash
nano app.py
```

Contenu de `app.py` :

```python
import os
import sys

print(f"--- Vérification de l'Utilisateur ---")
# Récupère l'utilisateur et l'ID (UID). UID 0 = root.
current_user = os.getenv('USER') or os.getenv('LOGNAME')
print(f"Utilisateur actuel : {current_user} (UID: {os.getuid()})")
print(f"--- Test d'Accès Privilégié ---")

# Tenter d'écrire dans un répertoire réservé à l'utilisateur root
try:
    with open("/root/test_root_access.txt", "w") as f:
        f.write("Tentative d'écriture réussie.")
    print("ÉCHEC SÉCURITAIRE : Accès root obtenu. L'écriture dans /root a réussi.")
    
except PermissionError:
    print("SUCCÈS SÉCURITAIRE : Accès root refusé (PermissionError).")
    
except Exception as e:
    print(f"Erreur inattendue: {e}")

print(f"--- Fin de l'application ---")
```

### C. Fichier des Dépendances

```bash
echo "Flask" > requirements.txt
```

-----

## 2\. Étape 1 : Exécution par Défaut en tant que Root (Non Sécurisé)

Nous construisons une image en omettant délibérément la configuration de sécurité pour voir le comportement par défaut de Docker.

### A. Dockerfile (Version Root)

```bash
nano Dockerfile.root
```

Contenu de **`Dockerfile.root`** :

```dockerfile
# L'utilisateur par défaut de cette image de base est 'root'.
FROM python:3.11-slim

WORKDIR /usr/src/app
COPY requirements.txt .

# Installation des dépendances (effectuée par l'utilisateur 'root')
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

# Nous omettons la commande 'USER' pour garder l'utilisateur 'root'
CMD ["python", "app.py"]
```

### B. Construction et Test (Root)

1.  **Construction de l'Image :**
    ```bash
    docker build -t app-as-root -f Dockerfile.root .
    ```
2.  **Lancement du Conteneur :**
    ```bash
    docker run --rm app-as-root
    ```

### C. Résultat de l'Étape 1 (Risque de Sécurité)

Le résultat montre que l'application s'exécute en tant que `root` (UID 0) et parvient à effectuer une action privilégiée.

```
--- Vérification de l'Utilisateur ---
Utilisateur actuel : root (UID: 0)
--- Test d'Accès Privilégié ---
ÉCHEC SÉCURITAIRE : Accès root obtenu. L'écriture dans /root a réussi.
--- Fin de l'application ---
```

-----

## 3\. Étape 2 : Exécution avec Utilisateur Non-Root (Moindre Privilège)

Nous modifions le `Dockerfile` pour créer un utilisateur sans privilèges (`appuser`) et forcer l'exécution de l'application sous ce compte.

### A. Dockerfile (Version Sécurisée)

```bash
nano Dockerfile.nonroot
```

Contenu de **`Dockerfile.nonroot`** :

```dockerfile
# PHASE ROOT (Installation des dépendances)
FROM python:3.11-slim

WORKDIR /usr/src/app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# --- Application du Moindre Privilège ---

# 1. Création de l'utilisateur non-root 'appuser'
RUN useradd -ms /bin/false appuser

# 2. S'assurer que 'appuser' possède le répertoire de travail
COPY app.py .
RUN chown -R appuser:appuser /usr/src/app

# 3. Changer l'utilisateur par défaut pour l'exécution finale
USER appuser

# PHASE NON-ROOT (Exécution de l'application)
CMD ["python", "app.py"]
```

### B. Construction et Test (Non-Root)

1.  **Construction de l'Image :**
    ```bash
    docker build -t app-non-root -f Dockerfile.nonroot .
    ```
2.  **Lancement du Conteneur :**
    ```bash
    docker run --rm app-non-root
    ```

### C. Résultat de l'Étape 2 (Succès Sécuritaire)

Le résultat montre que l'application s'exécute en tant que `appuser` (UID 1000) et se voit refuser l'accès aux zones privilégiées.

```
--- Vérification de l'Utilisateur ---
Utilisateur actuel : appuser (UID: 1000)
--- Test d'Accès Privilégié ---
SUCCÈS SÉCURITAIRE : Accès root refusé (PermissionError).
--- Fin de l'application ---
```

-----

Le passage de l'utilisateur **`root`** à **`appuser`** via la commande `USER` est la mesure de sécurité qui garantit que l'application ne dispose que des droits nécessaires pour fonctionner.
