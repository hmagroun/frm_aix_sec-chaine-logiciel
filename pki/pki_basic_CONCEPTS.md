
# Notes de Cours : Infrastructure à Clé Publique (PKI)

## 1. Introduction et Objectifs de la PKI

L'Infrastructure à Clé Publique (**PKI**) est un cadre de rôles, de politiques, de normes et de procédures requis pour créer, gérer, distribuer, utiliser, stocker et révoquer des **certificats numériques**. Son rôle fondamental est d'établir et de gérer la **confiance** dans les communications et les transactions numériques.

La PKI vise à garantir quatre objectifs de sécurité essentiels pour le protocole TLS/SSL :

1.  **Authentification :** Assurer qu'une entité (serveur, utilisateur) est bien celle qu'elle prétend être.
2.  **Confidentialité :** Chiffrer les données pour les rendre illisibles à toute partie non autorisée.
3.  **Intégrité :** Vérifier que les données n'ont pas été modifiées ou altérées pendant la transmission.
4.  **Non-Répudiation :** Empêcher l'émetteur d'un message de nier l'avoir envoyé, grâce à la signature numérique.

---

## 2. Les Composants de la Cryptographie Asymétrique

La PKI repose sur la cryptographie asymétrique, qui utilise une paire de clés mathématiquement liées.

### La Paire de Clés

1.  **Clé Privée (.key)** :
    * **Nature** : Doit être maintenue **absolument secrète** par le détenteur.
    * **Fonctions** : Elle est utilisée pour **déchiffrer** les données envoyées au détenteur et pour **signer** les certificats ou les documents.
2.  **Clé Publique (intégrée au certificat)** :
    * **Nature** : Est **publique** et largement distribuée.
    * **Fonctions** : Elle est utilisée par les tiers pour **chiffrer** les données destinées au détenteur de la clé privée et pour **vérifier** sa signature numérique.

### L'Autorité de Certification (CA)

L'**Autorité de Certification (CA)** est l'entité de **tiers de confiance** qui garantit l'identité des participants.

* **Rôle** : La CA est responsable de la vérification de l'identité du demandeur de certificat. Une fois l'identité confirmée, elle utilise sa propre clé privée pour **signer** la clé publique et les informations d'identité du demandeur.
* **CA Racine (Root CA)** : Il s'agit du sommet de la hiérarchie de confiance. Le certificat racine est **auto-signé** et doit être installé dans le **magasin de confiance** des clients pour que tous les certificats qu'elle émet soient considérés comme valides.

---

## 3. Le Certificat Numérique X.509

Le certificat numérique est le document standard (format **X.509**) qui lie de manière vérifiable une clé publique à une identité.

### Structure du Certificat

Un certificat contient des informations cruciales pour la validation :

* **Subject (Sujet)** : L'identité du propriétaire (ex: le serveur web).
* **Issuer (Émetteur)** : L'identité de la CA qui a signé le certificat.
* **Période de Validité** : Les dates de début et de fin de validité du certificat.
* **Clé Publique** : La clé publique de l'entité.
* **Signature de l'Émetteur** : La preuve cryptographique que la CA a validé l'identité.

### Les Champs d'Identification Cruciaux

Les navigateurs modernes s'appuient sur deux champs pour vérifier le nom du serveur :

1.  **Common Name (CN)** : Le nom de domaine principal (ex: `serveur.local`). Historiquement utilisé, il est aujourd'hui considéré comme **obsolète** par la majorité des navigateurs pour la validation de domaine.
2.  **Subject Alternative Name (SAN)** : C'est le champ **obligatoire** qui liste explicitement tous les noms d'hôtes et adresses IP couverts par le certificat. La validation du nom par le client s'effectue strictement sur la base des entrées du champ SAN.

### Certificat Auto-Signé vs. Certificat Émis

* **Certificat Auto-Signé** : Un certificat où le champ **Émetteur** est identique au champ **Sujet**. Il est utilisé dans les environnements de test ou pour les CA racines. Les clients refusent ces certificats car ils ne peuvent pas remonter la chaîne de confiance jusqu'à une CA tierce reconnue.
* **Certificat Émis** : Un certificat signé par une CA externe (même une CA privée comme dans un laboratoire). La confiance est établie si le certificat de la CA est présent dans le magasin de confiance du client.

---

## 4. Le Cycle de Vie et la Validation

### 4.1. La Requête de Signature (CSR)

Le processus commence par la **Requête de Signature de Certificat (CSR)**. C'est un fichier généré par le serveur qui contient sa clé publique et ses informations d'identité (y compris le SAN). Le CSR est ensuite transmis à la CA pour signature.

### 4.2. Le Magasin de Confiance (Trust Store)

Le **Magasin de Confiance** est un répertoire de certificats racines fiables préinstallés ou ajoutés manuellement sur un système d'exploitation ou une application.

* **Fonctionnement Linux (Debian/Ubuntu)** : Les certificats de CA locales sont déposés dans des répertoires désignés (comme `/usr/local/share/ca-certificates/`). La commande **`update-ca-certificates`** est alors exécutée pour compiler ces fichiers dans le fichier système maître, rendant la CA reconnue par les bibliothèques SSL/TLS du système (utilisées par des outils comme `curl`).

### 4.3. La Chaîne de Validation

Lors d'une connexion sécurisée, la validation du certificat du serveur passe par les étapes suivantes :

1.  **Vérification de l'Émetteur** : Le client reçoit le certificat du serveur et identifie son Émetteur (la CA).
2.  **Remontée de la Chaîne** : Le client recherche le certificat de cette CA dans son Magasin de Confiance.
3.  **Vérification de la Signature** : Si le certificat de la CA est trouvé, le client utilise la clé publique de la CA pour vérifier la signature du certificat du serveur.
4.  **Vérification du Nom (SAN)** : Le client vérifie que le nom d'hôte utilisé dans l'URL correspond à une entrée dans le champ SAN du certificat.

Si l'une de ces étapes échoue (CA inconnue, signature invalide, ou nom non concordant), la connexion est rejetée ou un avertissement de sécurité est affiché. L'erreur de non-concordance de nom (`SSL_ERROR_BAD_CERT_DOMAIN`) est un échec de la quatrième étape.

