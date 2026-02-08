## 📋 Prérequis et Installation du système

Avant le déploiement de Vaultwarden, l'environnement Ubuntu a été préparé avec les outils nécessaires.

### 1. Mise à jour du système
```bash
sudo apt update && sudo apt upgrade -y```

### 2. Installation de l'environnement Docker (V2)

Pour éviter les erreurs de compatibilité rencontrées avec les anciennes versions Python, le moteur Docker moderne a été installé :

# Installation des dépendances
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y

# Ajout du dépôt officiel Docker et installation
sudo apt install docker.io docker-compose-v2 -y

# Ajout de l'utilisateur au groupe docker pour éviter sudo
sudo usermod -aG docker $USER

(Note : Un redémarrage de la session est nécessaire pour appliquer les droits du groupe docker)

### 3. Préparation du répertoire de travail

sudo mkdir -p /opt/vaultwarden
sudo chown $USER:$USER /opt/vaultwarden
cd /opt/vaultwarden

Déploiement d'un Gestionnaire de Secrets Sécurisé (Vaultwarden)
Ce projet consiste en la mise en place d'une instance Vaultwarden (implémentation légère de Bitwarden) auto-hébergée sur une machine virtuelle Ubuntu. L'objectif était de sécuriser la gestion des identifiants à travers une infrastructure conteneurisée et un transit de données chiffré.

🛠️ Stack Technique
Système d'exploitation : Ubuntu 24.04 LTS (Machine Virtuelle)

Conteneurisation : Docker & Docker Compose v5.0.2

Application : Vaultwarden (Rust)

Sécurité réseau : TLS/SSL (OpenSSL), HTTPS

Chiffrement : AES-256 (Zero-Knowledge)

🏗️ Architecture et Installation
1. Configuration de l'infrastructure Docker
Le service a été déployé via Docker pour garantir l'isolation des processus.

Création du répertoire de travail : /opt/vaultwarden

Gestion des volumes : Persistance des données chiffrées dans ./vw-data.

Configuration réseau : Exposition du service sur le port 8443 (HTTPS).

2. Sécurisation du Transit (TLS/SSL)
Pour éviter les attaques de type Man-in-the-Middle (MITM) et les erreurs de sécurité du navigateur, un certificat SSL auto-signé a été généré :

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 365 -nodes -subj "/CN=localhost"```

3. Fichier de déploiement (docker-compose.yml)
Le fichier a été configuré pour forcer l'utilisation du HTTPS via la variable ROCKET_TLS :

```YAML
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - SIGNUPS_ALLOWED=false  # Hardening : Fermeture des inscriptions publiques
      - DOMAIN=https://localhost:8443
      - ROCKET_TLS={certs="/data/cert.pem",key="/data/key.pem"}
    volumes:
      - ./vw-data:/data
      - ./cert.pem:/data/cert.pem:ro
      - ./key.pem:/data/key.pem:ro
    ports:
      - "8443:80"```

🔒 Hardening et Sécurité Appliquée
Enforcement HTTPS : Obligation d'utiliser un canal chiffré pour transmettre le Master Password.

Gestion des identités (IAM) :

Utilisation d'un Master Password complexe (>16 caractères).

Désactivation de la variable SIGNUPS_ALLOWED après la création du compte administrateur pour réduire la surface d'attaque.

Chiffrement Zero-Knowledge : Les données sont chiffrées localement sur le client avant d'être envoyées à la base de données.

💾 Sauvegarde et Résilience
Exportation utilisateur
Test d'exportation au format JSON chiffré pour garantir la confidentialité des secrets en dehors du coffre-fort.

Sauvegarde Administrateur (Disaster Recovery)
Procédure de backup du volume de données via compression tar :

```bash
sudo tar -czvf backup_vaultwarden.tar.gz /opt/vaultwarden/vw-data```

🚀 Fonctionnalités validées
Génération de mots de passe : Création de secrets à haute entropie.

Remplissage automatique : Reconnaissance des URIs (identifiants liés aux sites).

Persistance : Vérification du stockage après redémarrage des conteneurs.




## 🛠️ Journal de bord et Résolution de problèmes (Troubleshooting)

Durant le déploiement, plusieurs obstacles techniques ont été rencontrés et résolus.

### 1. Conflit de versions Docker Compose
* **Problème :** L'utilisation de `docker-compose` (V1, Python) provoquait des erreurs de modules (`ModuleNotFoundError`) sur Ubuntu 24.04.
* **Solution :** Migration vers le plugin **Docker Compose V2** intégré. Remplacement de la commande par `docker compose` (sans tiret).

### 2. Restriction de sécurité HTTPS (Vaultwarden)
* **Problème :** Impossible de finaliser l'inscription ("Continue" grisé) car Vaultwarden refuse de traiter des secrets via une connexion HTTP non chiffrée.
* **Solution :** 1. Génération d'un certificat auto-signé via OpenSSL.
    2. Configuration de la variable `ROCKET_TLS` pour forcer le chiffrement.
    3. Acceptation manuelle de l'exception de sécurité dans le navigateur.

### 3. Gestion des permissions Linux
* **Problème :** Erreur lors du déplacement ou de la lecture des fichiers `.pem` par le conteneur Docker.
* **Solution :** Utilisation de `chown` pour réattribuer la propriété des fichiers à l'utilisateur courant et `chmod` pour restreindre les droits d'accès aux clés privées (sécurité des fichiers au repos).

### 4. Persistence et Volumes
* **Problème :** Risque de perte de données à l'arrêt du conteneur.
* **Solution :** Montage d'un volume local (`./vw-data`) vers le répertoire `/data` du conteneur, garantissant que la base de données SQLite survit aux mises à jour et redémarrages.
