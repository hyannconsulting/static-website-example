pipeline {
    agent none
    environment {
        DOCKERHUB_AUTH = credentials('dockerhub-credentials')
        ID_DOCKER = "${DOCKERHUB_AUTH_USR}"
        PORT_EXPOSED = "8090"
        IMAGE_NAME = 'static-website'
        IMAGE_TAG = 'latest'
        APP_NAME = 'hyann-consulting'
        IMAGE_URL = 'http://192.168.56.100'
    }
    stages {

        //=========================================
        // CI Stages
        //=========================================

        stage('Build') {
            agent any
            steps {
                script {
                    sh 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }

        stage('Code Quality') {
            agent any
            steps {
                script {
                    sh '''
                        echo "Running code quality checks..."
                        echo "Validating HTML structure..."
                        docker run --rm ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} nginx -t
                        echo "Code quality checks passed."
                    '''
                }
            }
        }

        stage('Package') {
            agent any
            steps {
                script {
                    sh '''
                        echo "Clean Environment"
                        docker rm -f $IMAGE_NAME || echo "container does not exist"
                        docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:80 ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                        sleep 5
                    '''
                }
            }
        }

        //=========================================
        // Review / Tests
        //=========================================

        stage('Review / Tests') {
            agent any
            steps {
                script {
                    sh '''
                        echo "Running acceptance tests..."
                        curl -s -o /dev/null -w "%{http_code}" ${IMAGE_URL}:${PORT_EXPOSED} | grep -q "200"
                        echo "Acceptance tests passed."
                    '''
                }
            }
            post {
                always {
                    script {
                        sh '''
                            echo "Cleaning up test container..."
                            docker stop $IMAGE_NAME || true
                            docker rm $IMAGE_NAME || true
                        '''
                    }
                }
            }
        }

        //=========================================
        // Push Image to Docker Hub
        //=========================================

        stage('Login and Push Image on Docker Hub') {
            agent any
            steps {
                script {
                    sh '''
                        echo $DOCKERHUB_AUTH_PSW | docker login -u $DOCKERHUB_AUTH_USR --password-stdin
                        docker push ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        //=========================================
        // CD Stages
        //=========================================

        stage('Deploy in Staging') {
            agent any
            environment {
                HOSTNAME_DEPLOY_STAGING = "ec2-54-226-234-15.compute-1.amazonaws.com"
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_AUTH_SERVER', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp || echo 'app does not exist'"
                        command4="docker run -d -p 8090:80 --name static-website $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_STAGING} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4"
                    '''
                }
            }
        }

        stage('Deploy in Production') {
            agent any
            environment {
                HOSTNAME_DEPLOY_PROD = "ec2-13-222-142-249.compute-1.amazonaws.com"
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_AUTH_SERVER', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp || echo 'app does not exist'"
                        command4="docker run -d -p 8090:80 --name static-website $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_PROD} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4"
                    '''
                }
            }
        }
    }
}
