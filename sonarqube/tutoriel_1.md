
## Tutoriel "Hello World" SonarQube avec Python

Ce guide suppose que votre **SonarQube Server** est installé et accessible (par défaut, souvent sur `http://localhost:9000`).

### Étape 1 : Préparation du Code Python

Créons un simple fichier Python qui contient volontairement un défaut pour que SonarQube puisse le trouver.

Créez un dossier pour votre projet (ex: `sonar_hello_world`) et placez-y le fichier suivant :

#### Fichier : `main.py`

```python
# Fichier : main.py
import sys

def say_hello(name):
    """
    Fonction simple pour saluer l'utilisateur.
    Contient un Code Smell volontaire (variable non utilisée).
    """
    
    # Code Smell volontaire : 'unused_variable' n'est jamais utilisée
    unused_variable = 42 

    if name is None:
        # Bug potentiel si 'name' peut être None
        print("Hello, Guest!")
    else:
        print("Hello, " + name + "!")

if __name__ == "__main__":
    if len(sys.argv) > 1:
        say_hello(sys.argv[1])
    else:
        # Vulnérabilité potentielle si l'input vient d'un utilisateur
        user_input = input("Enter your name: ") 
        say_hello(user_input)

# Fin du fichier
```

### Étape 2 : Création du Fichier de Configuration

Le SonarScanner a besoin d'un fichier de configuration pour savoir quel code analyser et comment se connecter au serveur.

Créez le fichier de configuration principal dans le répertoire racine de votre projet :

#### Fichier : `sonar-project.properties`

```properties
# Fichier de configuration du projet SonarQube
# Doit être placé à la racine du projet

# --- Connexion au serveur ---
# L'URL de votre serveur SonarQube (à adapter si nécessaire)
sonar.host.url=http://localhost:9000

# Le token généré sur votre interface SonarQube (à remplacer par votre token)
# IMPORTANT : NE PAS LAISSER EN CLAIR DANS UN REPOSITORY PUBLIC !
# En CI/CD, utilisez une variable d'environnement sécurisée.
sonar.login=votre_token_personnel_ici 

# --- Informations sur le projet ---
# Clé unique du projet (doit être la même que celle créée sur le serveur)
sonar.projectKey=python_hello_world

# Nom affiché dans l'interface
sonar.projectName=Python Hello World Example

# Version du projet
sonar.projectVersion=1.0

# --- Définition du code à analyser ---
# Le répertoire contenant le code source (point de départ de l'analyse)
sonar.sources=.

# Fichiers à exclure (facultatif)
# sonar.exclusions=tests/**

# Langage du projet (important pour appliquer le bon Quality Profile)
sonar.language=py
```

### Étape 3 : Création du Projet sur le Serveur SonarQube

Avant de lancer le scan, vous devez informer le serveur SonarQube de l'existence de ce nouveau projet :

1.  Ouvrez l'interface web de votre serveur SonarQube (`http://localhost:9000`).
2.  Cliquez sur **"Create Project"** (Créer un projet).
3.  Sélectionnez **"Locally"** ou **"Manually"** (Manuellement).
4.  Entrez la **Project Key** (Clé du projet) : **`python_hello_world`** (doit correspondre à la valeur dans `sonar-project.properties`).
5.  Donnez un nom au projet : `Python Hello World Example`.
6.  Choisissez l'option pour générer un **Token** ou utilisez un Token existant.
7.  **Copiez ce Token** et remplacez la valeur `votre_token_personnel_ici` dans votre fichier `sonar-project.properties`.

---

Pour générer un jeton (Token) dans l'interface web de SonarQube, suivez ces étapes détaillées. Ce jeton est essentiel pour authentifier votre analyseur (comme Maven, Gradle, ou un pipeline CI/CD) auprès du serveur SonarQube.

#### 1. Accès au Menu de Sécurité

1.  **Connectez-vous** : Accédez à l'interface web de SonarQube et connectez-vous avec votre compte utilisateur.
2.  **Accédez au Profil Utilisateur** : Cliquez sur votre **nom d'utilisateur** ou votre **icône d'avatar** (généralement en haut à droite de l'écran).
3.  **Ouvrez le Menu Mon Compte** : Dans le menu déroulant, sélectionnez **"Mon Compte"** (ou "My Account").

#### 2. Génération du Jeton

1.  **Naviguez vers l'onglet Sécurité** : Une fois dans votre page "Mon Compte", cliquez sur l'onglet **"Sécurité"** (ou "Security") dans la barre de navigation latérale. Vous verrez la section "Tokens".
2.  **Générez un Nouveau Jeton** : Repérez la section "Generate Tokens" et entrez un **nom significatif** pour votre nouveau jeton (par exemple : `analyse_mon_projet_front` ou `jenkins_pipeline_ci`).
3.  **Définissez la Durée de Vie (Expiration)** : (Optionnel, mais recommandé) Vous pouvez choisir une date d'expiration pour le jeton. Pour les pipelines CI/CD critiques, il est souvent préférable de définir une date d'expiration courte.
4.  **Cliquez sur "Générer"** : Cliquez sur le bouton **"Générer"** (ou "Generate").

#### 3. Récupération du Jeton

1.  **Copiez le Jeton** : Le jeton généré apparaît dans une fenêtre contextuelle **une seule fois**. Copiez immédiatement la chaîne de caractères affichée.
    * **Avertissement de Sécurité** : Si vous fermez cette fenêtre avant de copier le jeton, vous ne pourrez plus le récupérer. Vous devrez le révoquer et en générer un nouveau.
2.  **Stockage Sécurisé** : Conservez ce jeton dans un gestionnaire de secrets sécurisé (comme Vault, des variables d'environnement chiffrées de votre outil CI/CD, ou un gestionnaire de mots de passe) car il permet d'interagir avec SonarQube en votre nom.

Vous utiliserez ensuite cette chaîne de caractères (le Token) dans vos scripts de construction (ex: `sonar.token=VOTRE_TOKEN_SECREt`) pour que l'analyse soit authentifiée. 

---

### Étape 4 : Lancement du SonarScanner

Nous allons maintenant exécuter le scanner depuis la ligne de commande (terminal) :

1.  **Assurez-vous** que le **SonarScanner CLI** est installé sur votre machine et que le répertoire de son exécution est dans votre PATH, ou naviguez jusqu'à son répertoire bin.

2.  **Ouvrez** un terminal et naviguez jusqu'au dossier racine de votre projet (`sonar_hello_world`).

3.  **Lancez la commande d'analyse :**

    ```bash
    sonar-scanner
    ```

4.  **Vérifiez le Terminal :**

      * Le scanner va lire le fichier `sonar-project.properties`.
      * Il analysera le fichier `main.py`.
      * À la fin, vous devriez voir un message indiquant que le rapport a été envoyé au serveur et une **Task ID** (Identifiant de tâche) pour le suivi.

### Étape 5 : Interprétation du Rapport

1.  **Ouvrez** l'interface web de SonarQube et trouvez votre projet : `Python Hello World Example`.

2.  Le tableau de bord affichera immédiatement les résultats. Vous devriez voir des défauts trouvés :

      * **Code Smell :** La variable `unused_variable` sera signalée car elle n'est pas utilisée.
      * **Bugs/Vulnérabilités :** SonarQube pourrait signaler des faiblesses dans le code d'entrée/sortie ou la gestion de `None`.

3.  **Vérification du Quality Gate :** Le statut sera probablement **FAILED** (Rouge) car vous avez introduit de nouveaux défauts (le Code Smell).

4.  **Correction :** Pour corriger l'échec et passer le Quality Gate :

      * Supprimez la ligne `unused_variable = 42` dans `main.py`.
      * Relancez le `sonar-scanner`.

5.  **Conclusion :** Après la correction et le deuxième scan, votre Quality Gate devrait passer au statut **PASSED** (Vert), confirmant que votre nouveau code est propre.

