
# Notes de Cours : Les Pare-feu sous Linux

## 1\. Introduction aux Pare-feu

Un **pare-feu** (firewall) est un dispositif de sécurité réseau qui surveille et filtre le trafic entrant et sortant selon un ensemble de règles définies. Son rôle essentiel est d'établir une barrière entre un réseau de confiance (interne) et un réseau non fiable (ex: Internet).

Le principe fondamental du pare-feu est le **filtrage des paquets** : chaque paquet de données est inspecté en fonction de critères (adresse source/destination, port, protocole) avant d'être autorisé à passer, refusé, ou rejeté.

### Pare-feu de Machine (Host Firewall) vs. Pare-feu de Réseau (Network Firewall)

Il existe deux catégories principales de pare-feu :

1.  **Pare-feu de Réseau (ou Périmétrique)** :
      * **Position** : Placé à la frontière entre deux réseaux (ex: routeur, appliance dédiée).
      * **Rôle** : Protéger l'ensemble d'un segment de réseau ou d'un réseau local contre les menaces externes. Il gère la traduction d'adresses (NAT) et les politiques globales.
2.  **Pare-feu de Machine (ou Hôte)** :
      * **Position** : Réside directement sur la machine hôte (serveur, poste de travail).
      * **Rôle** : Offrir une couche de défense supplémentaire, gérant l'accès aux services spécifiques et aux applications en cours d'exécution sur cette machine, quelle que soit la protection périmétrique. C'est le cas de Netfilter sous Linux.

-----

## 2\. L'Architecture de Filtrage sous Linux : Netfilter et IPTABLES

### Netfilter : Le Moteur du Noyau

**Netfilter** est le *framework* (cadre) intégré au **noyau (kernel)** Linux responsable de toutes les fonctions de gestion et de filtrage du réseau, y compris le NAT et le suivi des connexions (état). Netfilter intercepte les paquets à différents points critiques du traitement réseau.

### IPTABLES : L'Interface de Configuration

**IPTABLES** est l'utilitaire en ligne de commande historique qui permet aux administrateurs de communiquer avec le *framework* Netfilter pour définir, modifier et gérer les règles de filtrage. Bien qu'IPTABLES soit le nom souvent utilisé, il configure en réalité le moteur Netfilter.

### Les Tables et les Chaînes d'IPTABLES

IPTABLES organise ses règles en **tables** et **chaînes** :

#### Les Tables (Fonctionnalités)

Les tables définissent le type de traitement à appliquer au paquet :

  * **`filter` (par défaut)** : Utilisée pour les règles de filtrage standard (ACCEPT, DROP, REJECT).
  * **`nat`** : Utilisée pour la traduction d'adresses réseau (NAT).
  * **`mangle`** : Utilisée pour modifier les en-têtes de paquets (TTL, QoS, etc.).
  * **`raw`** : Utilisée pour marquer les paquets qui ne doivent pas être traités par le suivi de connexion (Connection Tracking).

#### Les Chaînes (Points de Traitement)

Les chaînes définissent le moment où les règles d'une table sont appliquées dans le flux de traitement des paquets :

  * **`PREROUTING`** : Appliquée aux paquets dès leur arrivée, avant la décision de routage. (Utilisée souvent par la table `nat`).
  * **`INPUT`** : Appliquée aux paquets **destinés au système local** lui-même (protège le serveur).
  * **`FORWARD`** : Appliquée aux paquets qui **traversent** le système (si la machine est utilisée comme routeur).
  * **`OUTPUT`** : Appliquée aux paquets **générés par le système local** avant qu'ils ne quittent la machine.
  * **`POSTROUTING`** : Appliquée aux paquets juste avant qu'ils ne soient transmis hors du système. (Utilisée souvent par la table `nat`).

Dans notre tutoriel, la chaîne la plus importante est **`INPUT`**, car elle protège les services locaux (`serveur`).

-----

## 3\. Les Pare-feu Simplifiés (Frontends)

Manipuler directement IPTABLES peut être complexe et source d'erreurs (une erreur peut bloquer l'accès à distance). Pour simplifier la gestion, des interfaces utilisateur ont été développées.

  * **UFW (Uncomplicated Firewall)** : Le frontal par défaut pour les distributions basées sur Debian (Ubuntu, Mint).
  * **FirewallD** : Le frontal par défaut pour les distributions basées sur Red Hat (Fedora, CentOS, RHEL).

Ces outils gèrent l'état, les zones (pour FirewallD), et traduisent les commandes simples en règles complexes IPTABLES.

### 4\. Détail sur UFW (Uncomplicated Firewall)

UFW est conçu pour la facilité d'utilisation et la réduction de la complexité d'IPTABLES.

#### Principes de Configuration UFW

1.  **Politiques par Défaut (Defaults)** : Définissent l'action à prendre si aucune règle ne correspond au trafic. La politique recommandée pour un serveur est **`deny incoming`** (tout refuser par défaut) et `allow outgoing` (autoriser la machine à se connecter à l'extérieur).
2.  **Règles d'Exception** : Les règles `ufw allow` et `ufw deny` définissent des exceptions à la politique par défaut. Les règles sont lues de haut en bas.
3.  **Profils de Services** : UFW utilise des profils préconfigurés basés sur le fichier `/etc/services` (ex: `ufw allow http` autorise les ports 80 et 443; `ufw allow ssh` autorise le port 22).

-----

## 5\. Relation entre UFW et IPTABLES : Un Exemple

UFW est une couche d'abstraction qui génère et gère les chaînes IPTABLES.

### Exemple de Traduction

Lorsqu'un administrateur saisit une commande UFW simple, elle est traduite en une ou plusieurs règles IPTABLES complexes.

**Commande UFW :**
L'administrateur autorise le trafic Web non sécurisé.

```bash
sudo ufw allow http
```

**Traduction IPTABLES (simplifiée) :**
UFW insère des règles dans les chaînes du noyau (souvent dans des chaînes créées par UFW, comme `ufw-user-input` ou `ufw-before-input`) :

```bash
# Règle pour IPv4 : Autoriser le trafic TCP sur le port 80 (HTTP)
sudo iptables -A ufw-user-input -p tcp --dport 80 -j ACCEPT
# Règle pour IPv4 : Autoriser le trafic TCP sur le port 443 (HTTPS)
sudo iptables -A ufw-user-input -p tcp --dport 443 -j ACCEPT
```

### Illustration de la Protection

Dans notre laboratoire :

1.  **`sudo ufw default deny incoming`** : UFW place la politique par défaut sur `DROP` (ou `REJECT` dans certaines chaînes) dans les chaînes clés d'IPTABLES.
2.  **`ufw allow http`** : UFW insère une règle `ACCEPT` pour le port 80 (et 443).
3.  **Accès FTP (Port 21)** : Comme aucune règle `ufw allow 21` n'a été spécifiée, le trafic FTP arrive, il ne correspond à aucune règle `ACCEPT` et tombe sur la politique par défaut, qui est `DENY` (refusé), protégeant ainsi le serveur.

L'utilisation d'UFW est donc recommandée pour la sécurité quotidienne, car elle minimise les risques d'erreurs tout en utilisant la puissance et la robustesse de Netfilter/IPTABLES.

