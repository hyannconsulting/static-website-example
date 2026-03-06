# Pipeline de Déploiement CI/CD sur AWS

Ce document décrit les différentes étapes de la pipeline Jenkins pour le déploiement de l'application sur AWS, ainsi que les instructions pour créer les instances EC2 nécessaires.

---

## Étapes de la Pipeline

### Pipeline CI (Intégration Continue)

#### 1. Build

Cette étape construit l'image Docker de l'application à partir du `Dockerfile` présent dans le projet.

- Commande : `docker build -t <ID_DOCKER>/<IMAGE_NAME>:<IMAGE_TAG> .`
- L'image est basée sur `nginx:alpine` et copie les fichiers statiques du site dans le répertoire nginx.

#### 2. SonarQube Analysis

Cette étape analyse le code source avec SonarQube pour détecter les problèmes de qualité et de sécurité.

- Utilise l'environnement SonarQube configuré dans Jenkins (`withSonarQubeEnv('SonarQube')`).
- Paramètres utilisés :
  - `sonar.projectKey` : `static-website-example-jenkins`
  - `sonar.projectName` : `static-website-example-jenkins`
  - `sonar.organization` : `hyannconsulting`
  - `sonar.sources` : `.`

#### 3. Quality Gate

Cette étape attend le résultat de l'analyse SonarQube et bloque le pipeline si la qualité du code ne satisfait pas les critères définis.

- Timeout configuré à **5 minutes**.
- Si la Quality Gate échoue, le pipeline est interrompu (`abortPipeline: true`).

#### 4. Code Quality

Cette étape vérifie la validité de la configuration nginx embarquée dans l'image Docker.

- Validation de la configuration nginx avec `nginx -t`.
- Permet de détecter les erreurs de configuration avant le déploiement.

#### 5. Package

Cette étape lance un conteneur Docker à partir de l'image construite pour préparer les tests.

- Supprime tout conteneur précédent portant le même nom.
- Démarre le conteneur sur le port exposé (`PORT_EXPOSED=8090` → port 80 du conteneur).

### Review / Tests

Cette étape exécute les tests d'acceptation sur le conteneur en cours d'exécution.

- Vérifie que l'application répond correctement avec un code HTTP 200 via `curl`.
- Nettoie automatiquement le conteneur de test après l'exécution (succès ou échec).

### Push de l'Image Docker

Après validation, l'image Docker est poussée vers Docker Hub.

- Authentification sur Docker Hub avec les credentials Jenkins (`dockerhub-credentials`).
- Push de l'image taguée vers le registre.

### Pipeline CD (Déploiement Continu)

#### 7. Deploy in Staging

Déploie l'application sur le serveur de staging AWS.

- Connexion SSH vers l'instance EC2 de staging (`ec2-18-208-200-72.compute-1.amazonaws.com`).
- Pull de l'image Docker depuis Docker Hub.
- Suppression de l'ancien conteneur et lancement du nouveau sur le port 8090.

#### 8. Deploy in Production

Déploie l'application sur le serveur de production AWS.

- Connexion SSH vers l'instance EC2 de production (`ec2-3-215-181-230.compute-1.amazonaws.com`).
- Pull de l'image Docker depuis Docker Hub.
- Suppression de l'ancien conteneur et lancement du nouveau sur le port 8090.

---

## Variables d'Environnement du Jenkinsfile

Les variables suivantes sont définies directement dans le `Jenkinsfile` :

| Variable               | Valeur                    | Description                                      |
|------------------------|---------------------------|--------------------------------------------------|
| `ID_DOCKER`            | `$DOCKERHUB_AUTH_USR`     | Identifiant Docker Hub (issu des credentials)    |
| `PORT_EXPOSED`         | `8090`                    | Port exposé sur les serveurs de déploiement      |
| `IMAGE_NAME`           | `static-website`          | Nom de l'image Docker                            |
| `IMAGE_TAG`            | `latest`                  | Tag de l'image Docker                            |
| `APP_NAME`             | `hyann-consulting`         | Nom de l'application                             |
| `IMAGE_URL`            | `http://192.168.55.90`    | URL locale utilisée pour les tests d'acceptation |

---

## Prérequis Jenkins

Les credentials suivants doivent être configurés dans Jenkins :

| Credential ID           | Type                          | Description                                                         |
|-------------------------|-------------------------------|---------------------------------------------------------------------|
| `dockerhub-credentials` | Username/Password             | Identifiants Docker Hub (utilisé via `$DOCKERHUB_AUTH_USR/PSW`)    |
| `SSH_AUTH_SERVER`       | SSH Username with private key | Clé SSH pour les serveurs de déploiement (staging et production)    |

**Important :** Le credential `SSH_AUTH_SERVER` est utilisé pour les deux environnements (staging et production). Assurez-vous que la même clé SSH a accès aux deux instances EC2.

Le plugin **SonarQube Scanner** doit être installé sur l'agent Jenkins et un environnement SonarQube nommé `SonarQube` doit être configuré dans **Manage Jenkins > Configure System**.

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

- Sélectionnez **Ubuntu Server 22.04 LTS** (compatible avec Docker).

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

| Type        | Protocole | Port | Source          | Description                    |
|-------------|-----------|------|-----------------|--------------------------------|
| SSH         | TCP       | 22   | IP du Jenkins   | Accès SSH Jenkins              |
| Custom TCP  | TCP       | 8090 | 0.0.0.0/0       | Accès web public sur port 8090 |

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
ssh -i webapp-key.pem ubuntu@<DNS_PUBLIC_EC2>
```

Installez Docker :

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Docker
sudo apt install -y docker.io

# Démarrage du service Docker
sudo systemctl start docker
sudo systemctl enable docker

# Ajout de l'utilisateur ubuntu au groupe docker
sudo usermod -aG docker ubuntu

# Déconnectez-vous et reconnectez-vous pour appliquer les changements
exit
```

Vérifiez l'installation :

```bash
ssh -i webapp-key.pem ubuntu@<DNS_PUBLIC_EC2>
docker --version
```

### Étape 6 : Configurer les credentials dans Jenkins

#### Clé SSH

1. Dans Jenkins, allez dans **Manage Jenkins** > **Credentials**.
2. Ajoutez un nouveau credential de type **SSH Username with private key**.
3. **ID** : `SSH_AUTH_SERVER`.
4. **Username** : `ubuntu`.
5. **Private Key** : Collez le contenu du fichier `.pem` téléchargé.

> **Note** : Un seul credential `SSH_AUTH_SERVER` est utilisé pour les deux environnements (staging et production). Assurez-vous que la clé SSH associée a accès aux deux instances EC2.

#### Docker Hub

1. Ajoutez un credential de type **Username with password**.
2. **ID** : `dockerhub-credentials`.
3. Renseignez votre identifiant et mot de passe Docker Hub.

### Étape 7 : Mettre à jour les hostnames dans le Jenkinsfile

Remplacez les valeurs des variables d'environnement dans le `Jenkinsfile` par les DNS publics de vos instances EC2 :

```groovy
// Staging
environment {
    HOSTNAME_DEPLOY_STAGING = "ec2-18-208-200-72.compute-1.amazonaws.com"
}

// Production
environment {
    HOSTNAME_DEPLOY_PROD = "ec2-3-215-181-230.compute-1.amazonaws.com"
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
          ┌────────────────▼────────────────┐
          │             Build               │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │       SonarQube Analysis        │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │          Quality Gate           │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │          Code Quality           │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │             Package             │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │          Review / Tests         │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │     Login and Push to           │
          │          Docker Hub             │
          └────────────────┬────────────────┘
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
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
withCredentials([sshUserPrivateKey(credentialsId: 'SSH_AUTH_SERVER', keyFileVariable: 'SSH_KEY')]) {
    sh '''
        ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_STAGING} ...
    '''
}
```

**Note importante :** Assurez-vous que le credential `SSH_AUTH_SERVER` est bien configuré dans Jenkins avec :
- Type : **SSH Username with private key**
- ID : `SSH_AUTH_SERVER`
- Username : `ubuntu`
- Private Key : Le contenu de votre clé privée `.pem`

### Erreur SonarQube / Quality Gate

- Assurez-vous que le plugin **SonarQube Scanner** est installé dans Jenkins.
- Vérifiez que l'environnement SonarQube est bien nommé `SonarQube` dans **Manage Jenkins > Configure System**.
- Le timeout de la Quality Gate est de 5 minutes. Si l'analyse prend plus longtemps, augmentez la valeur dans le Jenkinsfile.

### Problèmes de connexion SSH

- Vérifiez que le Security Group autorise le port 22 depuis l'IP Jenkins.
- Vérifiez les permissions de la clé : `chmod 400 webapp-key.pem`.
- Assurez-vous que l'utilisateur SSH est bien `ubuntu`.

### Problèmes Docker

- Vérifiez que Docker est démarré : `sudo systemctl status docker`.
- Vérifiez que l'utilisateur est dans le groupe docker : `groups ubuntu`.

### Problèmes de déploiement

- Vérifiez les logs Jenkins pour identifier l'étape en échec.
- Testez la connexion SSH manuellement depuis le serveur Jenkins.
- Vérifiez que le port 8090 est accessible dans le Security Group.

---

## Accès au Site après le Déploiement

Une fois le déploiement terminé avec succès, le site web est accessible sur les serveurs de staging et production sur le **port 8090**.

### Accès à l'environnement Staging

Le site de staging est accessible à l'adresse suivante :

```
http://<DNS_PUBLIC_STAGING>:8090
```

Exemple avec le DNS de staging configuré dans le Jenkinsfile :
```
http://ec2-18-208-200-72.compute-1.amazonaws.com:8090
```

### Accès à l'environnement Production

Le site de production est accessible à l'adresse suivante :

```
http://<DNS_PUBLIC_PRODUCTION>:8090
```

Exemple avec le DNS de production configuré dans le Jenkinsfile :
```
http://ec2-3-215-181-230.compute-1.amazonaws.com:8090
```

### Configuration du Security Group

**Important** : Pour que le site soit accessible sur le port 8090, vous devez mettre à jour les règles du Security Group de vos instances EC2.

Ajoutez la règle suivante au Security Group :

| Type        | Protocole | Port | Source          | Description                    |
|-------------|-----------|------|-----------------|--------------------------------|
| Custom TCP  | TCP       | 8090 | 0.0.0.0/0       | Accès web public sur port 8090 |

#### Étapes pour ajouter la règle dans AWS Console :

1. Connectez-vous à la [Console AWS](https://console.aws.amazon.com)
2. Accédez au service **EC2**
3. Dans le menu de gauche, cliquez sur **Security Groups**
4. Sélectionnez le Security Group associé à vos instances
5. Cliquez sur l'onglet **Inbound rules**
6. Cliquez sur **Edit inbound rules**
7. Cliquez sur **Add rule**
8. Configurez la nouvelle règle :
   - **Type** : Custom TCP
   - **Port range** : 8090
   - **Source** : 0.0.0.0/0 (ou une plage IP spécifique pour plus de sécurité)
   - **Description** : Accès web public sur port 8090
9. Cliquez sur **Save rules**

### Vérification du Déploiement

Pour vérifier que le site fonctionne correctement, vous pouvez utiliser `curl` :

```bash
# Test du serveur staging
curl -I http://<DNS_PUBLIC_STAGING>:8090

# Test du serveur production
curl -I http://<DNS_PUBLIC_PRODUCTION>:8090
```

Une réponse avec le code `HTTP/1.1 200 OK` indique que le site est accessible et fonctionne correctement.

### Accès via Navigateur Web

Vous pouvez également accéder au site directement depuis votre navigateur web en entrant l'URL :

- **Staging** : `http://<DNS_PUBLIC_STAGING>:8090`
- **Production** : `http://<DNS_PUBLIC_PRODUCTION>:8090`

Le site web statique devrait s'afficher correctement dans votre navigateur.

---
## Resultat du pipeline

![alt text](<_result/image_3.png>)

---
## Apres deploiement

![alt text](<_result/image_II.png>)

![alt text](<_result/image_4.png>)
