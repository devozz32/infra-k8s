@Library('jenkins-share-lib') _

pipeline {
    agent { label 'jenkins-agent-pod' }

    environment {
        // Registry Configuration
        REGISTRY_URL   = "docker.io"
        DOCKERHUB_USER = "talko32"

        // Infrastructure Repository
        INFRA_REPO     = "https://github.com/devozz32/infra-k8s.git"
        INFRA_BRANCH   = "main"

        // Project Configuration
        PROJECT_NAME   = ""
    }

    stages {
        stage('Debug Environment') {
            steps {
                script {
                    env.DEPLOY_ENV = (
                        env.BRANCH_NAME.endsWith('dev')   ? 'dev'   :
                        env.BRANCH_NAME.endsWith('stage') ? 'stage' :
                        env.BRANCH_NAME.endsWith('main')  ? 'prod'  : 'none'
                    )

                    env.K8S_NAMESPACE = env.DEPLOY_ENV

                    echo """
                    ============================================
                    🔍 Pipeline Configuration
                    ============================================
                    Branch     : ${env.BRANCH_NAME}
                    Environment: ${env.DEPLOY_ENV}
                    Namespace  : ${env.K8S_NAMESPACE}
                    Build      : #${env.BUILD_NUMBER}
                    ============================================
                    """
                }
            }
        }

        stage('Checkout Source') {
            steps {
                checkout scm
                script {
                    env.PROJECT_NAME = getprojectnamefromgit()
                    echo "✅ Detected PROJECT_NAME = ${env.PROJECT_NAME}"
                    echo "Branch: ${env.BRANCH_NAME}, Commit: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Get Versions') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                script {
                    env.BACKEND_VERSION  = getversion('backend/VERSION.txt')
                    env.AUTH_VERSION     = getversion('auth-service/VERSION.txt')
                    env.FRONTEND_VERSION = getversion('jewelry-store/VERSION.txt')

                    env.BACKEND_TAG  = "${env.DOCKERHUB_USER}/store-backend:${env.BACKEND_VERSION}.${env.BUILD_NUMBER}"
                    env.AUTH_TAG     = "${env.DOCKERHUB_USER}/store-auth:${env.AUTH_VERSION}.${env.BUILD_NUMBER}"
                    env.FRONTEND_TAG = "${env.DOCKERHUB_USER}/store-frontend:${env.FRONTEND_VERSION}.${env.BUILD_NUMBER}"

                    echo """
                    ============================================
                    📦 Version Information
                    ============================================
                    Backend  : ${env.BACKEND_TAG}
                    Auth     : ${env.AUTH_TAG}
                    Frontend : ${env.FRONTEND_TAG}
                    ============================================
                    """
                }
            }
        }

        stage('Build & Push Images') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "🔐 Docker login..."
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                        echo "🔨 Building images..."
                        docker build -t ${env.BACKEND_TAG} ./backend
                        docker build -t ${env.AUTH_TAG} ./auth-service
                        docker build -t ${env.FRONTEND_TAG} ./jewelry-store

                        echo "📤 Pushing images..."
                        docker push ${env.BACKEND_TAG}
                        docker push ${env.AUTH_TAG}
                        docker push ${env.FRONTEND_TAG}

                        echo "✅ All images pushed successfully!"
                    """
                }
            }
        }

        stage('Checkout Infrastructure Repo') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                script {
                    echo "📥 Cloning infra repository..."
                    dir('infra-k8s') {
                        git branch: "${env.INFRA_BRANCH}", url: "${env.INFRA_REPO}"
                    }
                    echo "✅ Infrastructure repo cloned"
                }
            }
        }

        stage('Install Helm') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                sh '''
                    if ! command -v helm >/dev/null 2>&1; then
                        echo "🪄 Installing Helm..."
                        curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                    else
                        echo "✅ Helm already installed."
                    fi
                    helm version
                '''
            }
        }

        stage('Manual Approval for Production') {
            when { expression { env.BRANCH_NAME.endsWith("main") } }
            steps {
                input message: '🚨 Deploy to PRODUCTION?', ok: 'Deploy!'
            }
        }

        stage('Prepare JWT Secret Values') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                withCredentials([string(credentialsId: 'JWT_SECRET_KEY', variable: 'JWT_SECRET_KEY')]) {
                    sh '''
                        cp values.yaml values-temp.yaml
                        yq e ".jwtSecret.key = \"$JWT_SECRET_KEY\"" -i values-temp.yaml
                    '''
                }
            }
        }

        stage('Deploy via Helm') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                script {
                    env.HELM_DIR = "infra-k8s/jewelry-store"
                    env.VALUES_FILE = "${env.HELM_DIR}/values-temp.yaml"

                    sh """
                        echo "🚀 Deploying to ${env.DEPLOY_ENV.toUpperCase()} environment..."
                        echo "Namespace: ${env.K8S_NAMESPACE}"
                        echo ""

                        helm upgrade --install jewelry-store ${env.HELM_DIR} \
                            -f ${env.VALUES_FILE} \
                            --namespace ${env.K8S_NAMESPACE} \
                            --create-namespace \
                            --wait \
                            --timeout 10m \
                            --atomic

                        echo ""
                        echo "✅ ${env.DEPLOY_ENV.toUpperCase()} deployment completed!"
                    """
                }
            }
        }

        stage('Verify Deployment') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                sh """
                    echo "📊 Verifying deployment in namespace: ${env.K8S_NAMESPACE}"
                    helm status jewelry-store -n ${env.K8S_NAMESPACE} || true
                    kubectl get pods -n ${env.K8S_NAMESPACE} -o wide || true
                    kubectl get svc -n ${env.K8S_NAMESPACE} || true
                    kubectl get ingress -n ${env.K8S_NAMESPACE} || echo "No Ingress"
                """
            }
        }

        stage('Health Check') {
            when { expression { env.BRANCH_NAME =~ /(dev|stage|main)$/ } }
            steps {
                sh """
                    echo "🔍 Checking pod readiness..."
                    kubectl wait --for=condition=ready pod --all -n ${env.K8S_NAMESPACE} --timeout=5m || echo "⚠️ Some pods are not ready"
                    kubectl get pods -n ${env.K8S_NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo """
            ============================================
            ✅ DEPLOYMENT SUCCESSFUL!
            ============================================
            Environment : ${env.DEPLOY_ENV.toUpperCase()}
            Namespace   : ${env.K8S_NAMESPACE}
            Build       : #${env.BUILD_NUMBER}
            Branch      : ${env.BRANCH_NAME}
            ============================================
            """
        }

        failure {
            echo """
            ============================================
            ❌ DEPLOYMENT FAILED!
            ============================================
            """
            sh """
                echo "🔍 Debug info:"
                kubectl get pods -n ${env.K8S_NAMESPACE} || true
                helm history jewelry-store -n ${env.K8S_NAMESPACE} || true
            """
        }

        always {
            sh """
                echo "🏁 Pipeline finished for branch: ${env.BRANCH_NAME}"
                docker system prune -f || true
            """
        }
    }
}
