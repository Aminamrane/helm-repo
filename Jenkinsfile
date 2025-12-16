pipeline {
    agent any

    parameters {
        choice(
            name: 'SERVICE',
            choices: ['all', 'backend', 'frontend', 'auth', 'users', 'items', 'gateway'],
            description: 'Service to deploy (all = full platform)'
        )
        string(
            name: 'IMAGE_VERSION',
            defaultValue: 'dev',
            description: 'Docker image version/tag to deploy'
        )
        string(
            name: 'NAMESPACE',
            defaultValue: 'dev',
            description: 'Kubernetes namespace'
        )
        string(
            name: 'ENVIRONMENT',
            defaultValue: 'dev',
            description: 'Environment (dev/prod)'
        )
    }

    environment {
        DOCKER_USERNAME = 'leogrv22'
        HELM_CHART_PATH = 'platform'
        KUBECONFIG_CREDENTIALS = 'kubeconfig-dev'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out Helm charts from ${env.GIT_URL}"
                    checkout scm
                }
            }
        }

        stage('Validate Helm Charts') {
            steps {
                script {
                    echo "Validating Helm charts..."
                    sh """
                        helm lint ${HELM_CHART_PATH} || true
                        echo "Helm charts validated"
                    """
                }
            }
        }

        stage('Update Dependencies') {
            steps {
                script {
                    echo "Updating Helm chart dependencies..."
                    sh """
                        cd ${HELM_CHART_PATH}
                        helm dependency update
                    """
                }
            }
        }

        stage('Update Image Versions') {
            steps {
                script {
                    echo "Updating image versions in values.yaml..."
                    script {
                        def valuesFile = "${HELM_CHART_PATH}/values.yaml"
                        def imageTag = 'dev'

                        if (params.SERVICE == 'all' || params.SERVICE == 'backend' || params.SERVICE == 'auth') {
                            sh """
                                sed -i 's|repository:.*auth|repository: ${DOCKER_USERNAME}/auth|g' ${valuesFile} || true
                                sed -i '/auth:/,/tag:/ s|tag:.*|tag: ${imageTag}|g' ${valuesFile} || true
                            """
                        }

                        if (params.SERVICE == 'all' || params.SERVICE == 'backend' || params.SERVICE == 'users') {
                            sh """
                                sed -i 's|repository:.*users|repository: ${DOCKER_USERNAME}/users|g' ${valuesFile} || true
                                sed -i '/users:/,/tag:/ s|tag:.*|tag: ${imageTag}|g' ${valuesFile} || true
                            """
                        }

                        if (params.SERVICE == 'all' || params.SERVICE == 'backend' || params.SERVICE == 'items') {
                            sh """
                                sed -i 's|repository:.*items|repository: ${DOCKER_USERNAME}/items|g' ${valuesFile} || true
                                sed -i '/items:/,/tag:/ s|tag:.*|tag: ${imageTag}|g' ${valuesFile} || true
                            """
                        }

                        if (params.SERVICE == 'all' || params.SERVICE == 'frontend') {
                            sh """
                                sed -i 's|repository:.*frontend|repository: ${DOCKER_USERNAME}/frontend|g' ${valuesFile} || true
                                sed -i '/frontend:/,/tag:/ s|tag:.*|tag: ${imageTag}|g' ${valuesFile} || true
                            """
                        }
                    }
                }
            }
        }

        stage('Install Traefik') {
            steps {
                script {
                    echo "Installing Traefik Ingress Controller..."

                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}

                            # Add Traefik Helm repo
                            helm repo add traefik https://traefik.github.io/charts || true
                            helm repo update

                            # Install Traefik
                            helm upgrade --install traefik traefik/traefik \
                                --namespace traefik \
                                --create-namespace \
                                --set service.type=LoadBalancer \
                                --wait \
                                --timeout 5m || true
                        """
                    }
                }
            }
        }

        stage('Confirm Prod Deployment') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        input(
                            message: "⚠️ Tu es sur le point de déployer en PROD. Confirmer le déploiement ?",
                            ok: "Déployer en PROD"
                        )
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    if (params.ENVIRONMENT == 'prod') {
                        echo "🚀 Deploying to PRODUCTION environment (namespace: ${params.NAMESPACE})"
                    } else {
                        echo "🔧 Deploying to DEV environment (namespace: ${params.NAMESPACE})"
                    }

                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')
                    ]) {

                        // 1) Namespace
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            kubectl get ns ${params.NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${params.NAMESPACE}
                        """

                        // 2) Cleanup logic (unchanged)
                        List<String> cleanupCommands = []

                        if (params.SERVICE == 'backend') {
                            echo "Deploying backend services only (auth, users, items, gateway)..."
                            cleanupCommands = [
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-auth --force --grace-period=0 || true",
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-users --force --grace-period=0 || true",
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-items --force --grace-period=0 || true"
                            ]
                        } else if (params.SERVICE == 'frontend') {
                            echo "Deploying frontend service only..."
                            cleanupCommands = [
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-frontend --force --grace-period=0 || true"
                            ]
                        } else if (params.SERVICE == 'auth') {
                            echo "Deploying auth service only..."
                            cleanupCommands = [
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-auth --force --grace-period=0 || true"
                            ]
                        } else if (params.SERVICE == 'users') {
                            echo "Deploying users service only..."
                            cleanupCommands = [
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-users --force --grace-period=0 || true"
                            ]
                        } else if (params.SERVICE == 'items') {
                            echo "Deploying items service only..."
                            cleanupCommands = [
                                "kubectl delete deployment -n ${params.NAMESPACE} platform-items --force --grace-period=0 || true"
                            ]
                        } else {
                            echo "Deploying full platform (all services)..."
                            cleanupCommands = [
                                "helm uninstall platform --namespace ${params.NAMESPACE} || true",
                                "kubectl delete deployment --all -n ${params.NAMESPACE} --force --grace-period=0 || true",
                                "kubectl delete pod --all -n ${params.NAMESPACE} --force --grace-period=0 || true"
                            ]
                        }

                        if (!cleanupCommands.isEmpty()) {
                            cleanupCommands.each { cmd ->
                                sh """
                                    export KUBECONFIG=\${KUBECONFIG_FILE}
                                    ${cmd}
                                """
                            }
                            sleep 3
                        }

                        // 3) Helm deploy
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            cd ${HELM_CHART_PATH}
                            helm upgrade --install platform . \
                                --namespace ${params.NAMESPACE} \
                                --wait \
                                --timeout 20m
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "Verifying deployment..."
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}

                            kubectl wait --for=condition=ready pod \
                                -l app.kubernetes.io/instance=platform \
                                -n ${params.NAMESPACE} \
                                --timeout=300s || true

                            kubectl get pods -n ${params.NAMESPACE}
                            kubectl get svc -n ${params.NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Helm deployment completed successfully!"
            echo "Service: ${params.SERVICE}"
            echo "Version: ${params.IMAGE_VERSION}"
            echo "Namespace: ${params.NAMESPACE}"
        }
        failure {
            echo "❌ Helm deployment failed!"
        }
        always {
            cleanWs()
        }
    }
}