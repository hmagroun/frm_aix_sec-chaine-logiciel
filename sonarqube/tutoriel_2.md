Reprenons le tutoriel "Hello World" Python, mais cette fois, nous allons l'intégrer directement dans un **pipeline Jenkins** pour automatiser l'analyse du code et l'application du **Quality Gate**.

Ce tutoriel utilise un **`Jenkinsfile`** (Pipeline Déclaratif) pour orchestrer les étapes.

---> à tester ...

## Tutoriel : Intégration SonarQube avec Docker Scanner dans Jenkins

Ce tutoriel utilise la syntaxe `agent { docker {...} }` pour exécuter l'étape d'analyse à l'intérieur d'un conteneur dédié.

### Prérequis Supplémentaires pour Docker

1.  Votre agent Jenkins doit avoir **Docker installé** et accessible par l'utilisateur qui exécute le pipeline.
2.  Vous devez utiliser un **Pipeline Déclaratif** pour la syntaxe `agent { docker }`.
3.  L'image officielle du scanner sera utilisée : `sonarsource/sonar-scanner-cli`.

### Étape 1 : Le Projet Python et le Secret Jenkins

Le fichier Python (`main.py`) et la configuration du Token SonarQube dans Jenkins (ID : `SONAR_TOKEN_SECRET`) restent **identiques** aux tutoriels précédents.



### Étape 2 : Le `Jenkinsfile` avec Image Docker

Dans cette version, nous utilisons l'agent Docker pour exécuter l'étape du scan. Nous devons monter le **répertoire de travail** de Jenkins dans le conteneur (`-v $PWD:/usr/src`) pour que le scanner puisse accéder au code source.

```groovy
pipeline {
    // L'agent principal du pipeline doit pouvoir exécuter Docker
    agent any 

    environment {
        // ID du Secret Text configuré dans Jenkins contenant votre Token SonarQube
        SONAR_TOKEN_CRED_ID = 'SONAR_TOKEN_SECRET'
        
        // Clé du projet créé sur le serveur SonarQube
        SONAR_PROJECT_KEY = 'jenkins-python-demo' 
        
        // Nom de la configuration de votre serveur SonarQube dans Jenkins
        SONAR_SERVER_NAME = 'SonarQube_Server' 
        
        // Temps d'attente maximum pour le résultat du Quality Gate (en minutes)
        SONAR_GATE_TIMEOUT = 5 
    }

    stages {
        
        // Étape de Construction (Build)
        stage('Build & Prepare') {
            steps {
                echo "Préparation de l'environnement..."
                // sh 'pip install -r requirements.txt' (pour un cas réel)
            }
        }
        
        // 1. Étape d'Analyse (Scan) exécutée dans un conteneur Docker
        stage('SonarQube Analysis (Docker)') {
            agent {
                // Utilisation de l'image Docker du scanner
                docker { 
                    image 'sonarsource/sonar-scanner-cli:latest' 
                    // Montage du répertoire de travail ($PWD) dans le conteneur pour que le scanner voie le code
                    args '-v $PWD:/usr/src' 
                }
            }
            steps {
                // Utilise les informations de connexion du serveur configurées dans Jenkins
                withSonarQubeEnv(SONAR_SERVER_NAME) { 
                    // Injecte le token secret pour l'authentification
                    withCredentials([string(credentialsId: SONAR_TOKEN_CRED_ID, variable: 'SONAR_AUTH_TOKEN')]) {
                        echo "Lancement du SonarScanner via Docker..."
                        // Le sonar-scanner est déjà dans le PATH du conteneur Docker
                        sh """
                            sonar-scanner \\
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \\
                                -Dsonar.sources=/usr/src \\
                                -Dsonar.language=py \\
                                -Dsonar.login=${SONAR_AUTH_TOKEN}
                        """
                    }
                }
            }
        }
        
        // 2. Étape de Vérification du Quality Gate (Blocking Step)
        // NOTE : Cette étape est exécutée sur l'agent principal, pas dans le conteneur Docker du scanner.
        stage('Quality Gate Check') {
            steps {
                script {
                    echo "Vérification du statut du Quality Gate..."
                    // Fonction du plugin SonarQube pour attendre la fin du traitement
                    def qualityGate = waitForQualityGate(
                        timeout: SONAR_GATE_TIMEOUT 
                    )

                    if (qualityGate.status != 'OK') {
                        error "❌ SonarQube Quality Gate FAILED. Construction arrêtée."
                    }
                    echo "✅ SonarQube Quality Gate PASSED. Le code est propre."
                }
            }
        }
        
        // 3. Étape de Continuité
        stage('Deploy If Quality Passed') {
            steps {
                echo 'Déploiement autorisé.'
            }
        }
    }
}
```

### Points Clés de l'Intégration Docker

1.  **`agent { docker { ... } }` :** Cette instruction dit à Jenkins d'utiliser l'image `sonarsource/sonar-scanner-cli:latest` pour exécuter toutes les commandes dans le bloc `steps`.
2.  **`args '-v $PWD:/usr/src'` :** Ceci est **crucial**. Il crée un lien (`volume mount`) entre le répertoire de travail actuel de Jenkins (`$PWD`) et le répertoire `/usr/src` à l'intérieur du conteneur. Sans cela, le scanner Docker ne pourrait pas voir votre fichier `main.py`.
3.  **`-Dsonar.sources=/usr/src` :** Le chemin des sources est ajusté pour pointer vers le répertoire où votre code a été monté à l'intérieur du conteneur Docker.
4.  **`sonar-scanner` direct :** Nous n'avons plus besoin de la variable `${SCANNER_HOME}` car l'exécutable `sonar-scanner` est déjà dans le PATH du conteneur Docker officiel.

Cette approche garantit une **analyse isolée, propre et facilement reproductible** sur n'importe quel agent Jenkins compatible Docker.

