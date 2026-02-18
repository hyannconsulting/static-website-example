# Pipeline de Déploiement CI/CD sur AWS

Ce document décrit les différentes étapes de la pipeline Jenkins pour le déploiement de l'application sur AWS, ainsi que les instructions pour créer les instances EC2 nécessaires.

---

## Étapes de la Pipeline

### Pipeline CI (Intégration Continue)

#### 1. Build

Cette étape construit l'image Docker de l'application à partir du `Dockerfile` présent dans le projet.

- Commande : `docker build -t <ID_DOCKER>/<IMAGE_NAME>:<IMAGE_TAG> .`
- L'image est basée sur `nginx:alpine` et copie les fichiers statiques du site dans le répertoire nginx.

#### 2. Code Quality

Cette étape vérifie la qualité du code et la validité de la configuration nginx.

- Validation de la configuration nginx avec `nginx -t`.
- Permet de détecter les erreurs de configuration avant le déploiement.

#### 3. Package

Cette étape lance un conteneur Docker à partir de l'image construite pour préparer les tests.

- Supprime tout conteneur précédent portant le même nom.
- Démarre le conteneur sur le port configuré (port 80 par défaut).

### Review / Tests

Cette étape exécute les tests d'acceptation sur le conteneur en cours d'exécution.

- Vérifie que l'application répond correctement avec un code HTTP 200.
- Nettoie le conteneur de test après l'exécution (succès ou échec).

### Push de l'Image Docker

Après validation, l'image Docker est poussée vers Docker Hub.

- Authentification sur Docker Hub avec les credentials Jenkins.
- Push de l'image taguée vers le registre.

### Pipeline CD (Déploiement Continu)

#### 5. Deploy in Staging

Déploie l'application sur le serveur de staging AWS.

- Connexion SSH vers l'instance EC2 de staging.
- Pull de l'image Docker depuis Docker Hub.
- Suppression de l'ancien conteneur et lancement du nouveau.

#### 6. Deploy in Production

Déploie l'application sur le serveur de production AWS.

- Connexion SSH vers l'instance EC2 de production.
- Pull de l'image Docker depuis Docker Hub.
- Suppression de l'ancien conteneur et lancement du nouveau.

---

## Prérequis Jenkins

Les credentials suivants doivent être configurés dans Jenkins :

| Credential ID      | Type                  | Description                                  |
|---------------------|-----------------------|----------------------------------------------|
| `dockerhub-credentials`   | Username/Password     | Identifiants Docker Hub                      |
| `SSH_AUTH_SERVER`   | SSH Username with private key | Clé SSH pour les serveurs de déploiement (staging et production) |

**Important :** Le credential `SSH_AUTH_SERVER` est utilisé pour les deux environnements (staging et production). Assurez-vous que la même clé SSH a accès aux deux instances EC2.

Les variables suivantes doivent être définies en paramètres du job Jenkins :

| Variable       | Description                         |
|----------------|-------------------------------------|
| `IMAGE_NAME`  | Nom de l'image Docker               |
| `IMAGE_TAG`   | Tag de l'image Docker (ex: latest)  |

---

## Création des Instances EC2 sur AWS

Deux instances EC2 sont nécessaires : une pour le **staging** et une pour la **production**.

### Étape 1 : Se connecter à la console AWS

1. Accédez à [https://console.aws.amazon.com](https://console.aws.amazon.com).
2. Connectez-vous avec vos identifiants AWS.

### Étape 2 : Accéder au service EC2

1. Dans la barre de recherche, tapez **EC2** et sélectionnez le service.
2. Cliquez sur **Instances** dans le menu de gauche.

### Étape 3 : Lancer une nouvelle instance

1. Cliquez sur **Launch instances**.
2. Configurez les paramètres suivants :

#### Nom et Tags

- **Staging** : `webapp-staging`
- **Production** : `webapp-production`

#### AMI (Amazon Machine Image)

- Sélectionnez **CentOS 7** ou **Amazon Linux 2** (compatible avec Docker).
- Vous pouvez rechercher une AMI CentOS dans le Marketplace AWS.

#### Type d'instance

- **Staging** : `t2.micro` (éligible au free tier) ou `t2.small`
- **Production** : `t2.small` ou `t2.medium` selon la charge attendue

#### Paire de clés (Key Pair)

1. Cliquez sur **Create new key pair** (ou sélectionnez une paire existante).
2. Donnez un nom à la clé (ex: `webapp-key`).
3. Format : **PEM** (pour Linux/Mac) ou **PPK** (pour Windows/PuTTY).
4. **Téléchargez et conservez la clé privée** — elle sera nécessaire pour la configuration Jenkins.

#### Configuration Réseau (Security Group)

Créez un Security Group avec les règles suivantes :

| Type  | Protocole | Port | Source          | Description         |
|-------|-----------|------|-----------------|---------------------|
| SSH   | TCP       | 22   | IP du Jenkins   | Accès SSH Jenkins   |
| HTTP  | TCP       | 80   | 0.0.0.0/0       | Accès web public    |

> **Important** : Limitez l'accès SSH à l'adresse IP de votre serveur Jenkins pour des raisons de sécurité.

#### Stockage

- **Taille** : 20 Go (gp3) minimum
- Le type gp3 offre un bon rapport performance/coût.

### Étape 4 : Lancer l'instance

1. Vérifiez la configuration.
2. Cliquez sur **Launch instance**.
3. Attendez que l'état passe à **running**.
4. Notez le **DNS public** de l'instance (ex: `ec2-100-24-13-223.compute-1.amazonaws.com`).

### Étape 5 : Installer Docker sur l'instance EC2

Connectez-vous à l'instance via SSH :

```bash
ssh -i webapp-key.pem centos@<DNS_PUBLIC_EC2>
```

Installez Docker :

```bash
# Mise à jour du système
sudo yum update -y

# Installation de Docker
sudo yum install -y docker

# Démarrage du service Docker
sudo systemctl start docker
sudo systemctl enable docker

# Ajout de l'utilisateur centos au groupe docker
sudo usermod -aG docker centos

# Déconnectez-vous et reconnectez-vous pour appliquer les changements
exit
```

Vérifiez l'installation :

```bash
ssh -i webapp-key.pem centos@<DNS_PUBLIC_EC2>
docker --version
```

### Étape 6 : Configurer les credentials dans Jenkins

#### Clé SSH

1. Dans Jenkins, allez dans **Manage Jenkins** > **Credentials**.
2. Ajoutez un nouveau credential de type **SSH Username with private key**.
3. **ID** : `SSH_AUTH_SERVER` (staging) ou `SSH_AUTH_PROD` (production).
4. **Username** : `centos`.
5. **Private Key** : Collez le contenu du fichier `.pem` téléchargé.

#### Docker Hub

1. Ajoutez un credential de type **Username with password**.
2. **ID** : `DOCKERHUB_AUTH`.
3. Renseignez votre identifiant et mot de passe Docker Hub.

### Étape 7 : Mettre à jour les hostnames dans le Jenkinsfile

Remplacez les valeurs des variables d'environnement dans le `Jenkinsfile` par les DNS publics de vos instances EC2 :

```groovy
// Staging
environment {
    HOSTNAME_DEPLOY_STAGING = "ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com"
}

// Production
environment {
    HOSTNAME_DEPLOY_PROD = "ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com"
}
```

---

## Répéter pour la deuxième instance

Répétez les étapes 3 à 6 pour créer la deuxième instance EC2 (production si vous avez commencé par staging, ou inversement).

---

## Architecture du Déploiement

```
                    ┌─────────────┐
                    │   Jenkins   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Build   │ │  Code    │ │ Package  │
        │  Image   │ │ Quality  │ │          │
        └──────────┘ └──────────┘ └──────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Review/Tests │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Docker Hub  │
                    │    Push      │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌────────────────┐       ┌────────────────┐
     │  EC2 Staging   │       │ EC2 Production │
     │  (AWS)         │       │ (AWS)          │
     └────────────────┘       └────────────────┘
```

---

## Dépannage

### Erreur "No such DSL method 'sshagent'"

Si vous rencontrez l'erreur :
```
java.lang.NoSuchMethodError: No such DSL method 'sshagent' found among steps
```

**Solution :**

Le plugin SSH Agent n'est pas nécessaire avec cette pipeline. Nous utilisons `withCredentials` avec `sshUserPrivateKey` qui est une fonctionnalité native de Jenkins. Cette approche présente plusieurs avantages :

1. **Pas de plugin supplémentaire requis** : `withCredentials` est disponible par défaut dans Jenkins
2. **Meilleure sécurité** : La clé SSH est exposée uniquement dans le contexte de l'exécution du bloc
3. **Compatibilité** : Fonctionne avec toutes les versions récentes de Jenkins

La configuration dans le Jenkinsfile utilise :
```groovy
withCredentials([sshUserPrivateKey(credentialsId: 'SSH_AUTH_SERVER', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
    sh '''
        ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${HOSTNAME} ...
    '''
}
```

**Note importante :** Assurez-vous que le credential `SSH_AUTH_SERVER` est bien configuré dans Jenkins avec :
- Type : **SSH Username with private key**
- ID : `SSH_AUTH_SERVER`
- Username : `ubuntu` (ou `centos` selon votre AMI)
- Private Key : Le contenu de votre clé privée `.pem`

### Problèmes de connexion SSH

- Vérifiez que le Security Group autorise le port 22 depuis l'IP Jenkins.
- Vérifiez les permissions de la clé : `chmod 400 webapp-key.pem`.
- Assurez-vous que l'utilisateur est bien `centos` pour les AMI CentOS.

### Problèmes Docker

- Vérifiez que Docker est démarré : `sudo systemctl status docker`.
- Vérifiez que l'utilisateur est dans le groupe docker : `groups centos`.

### Problèmes de déploiement

- Vérifiez les logs Jenkins pour identifier l'étape en échec.
- Testez la connexion SSH manuellement depuis le serveur Jenkins.
- Vérifiez que le port 80 est accessible dans le Security Group.
