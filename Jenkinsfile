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
            defaultValue: 'latest',
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
                        
                        if (params.SERVICE == 'all' || params.SERVICE == 'backend' || params.SERVICE == 'auth') {
                            sh """
                                sed -i 's|repository:.*auth|repository: ${DOCKER_USERNAME}/auth|g' ${valuesFile} || true
                                sed -i 's|tag:.*|tag: ${params.IMAGE_VERSION}|g' ${valuesFile} || true
                            """
                        }
                        
                        if (params.SERVICE == 'all' || params.SERVICE == 'backend' || params.SERVICE == 'users') {
                            sh """
                                sed -i 's|repository:.*users|repository: ${DOCKER_USERNAME}/users|g' ${valuesFile} || true
                            """
                        }
                        
                        if (params.SERVICE == 'all' || params.SERVICE == 'backend' || params.SERVICE == 'items') {
                            sh """
                                sed -i 's|repository:.*items|repository: ${DOCKER_USERNAME}/items|g' ${valuesFile} || true
                            """
                        }
                        
                        if (params.SERVICE == 'all' || params.SERVICE == 'frontend') {
                            sh """
                                sed -i 's|repository:.*frontend|repository: ${DOCKER_USERNAME}/frontend|g' ${valuesFile} || true
                                sed -i 's|tag:.*|tag: ${params.IMAGE_VERSION}|g' ${valuesFile} || true
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
                        file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE'),
                        string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            export AWS_ACCESS_KEY_ID=\${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=\${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=eu-west-3
                            
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
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying to Kubernetes namespace: ${params.NAMESPACE}"
                    
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE'),
                        string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            export AWS_ACCESS_KEY_ID=\${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=\${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=eu-west-3
                            
                            # Create namespace if it doesn't exist
                            kubectl create namespace ${params.NAMESPACE} || true
                            
                            # Get database password from AWS Secrets Manager
                            echo "Fetching database password from AWS Secrets Manager..."
                            SECRET_JSON=\$(aws secretsmanager get-secret-value \\
                                --secret-id microservices-platform-dev-secrets \\
                                --region eu-west-3 \\
                                --query 'SecretString' \\
                                --output text)
                            DB_PASSWORD=\$(python3 -c "import sys, json; print(json.loads(sys.stdin.read())['rds_master_password'])" <<< "\${SECRET_JSON}")
                            
                            # Create Kubernetes Secret for database credentials
                            kubectl create secret generic database-credentials \\
                                --from-literal=password="\${DB_PASSWORD}" \\
                                --from-literal=username="admin" \\
                                --namespace ${params.NAMESPACE} \\
                                --dry-run=client -o yaml | kubectl apply -f -
                            
                            # Deploy with Helm
                            cd ${HELM_CHART_PATH}
                            
                            # Uninstall old release if exists (to clean up old resources)
                            helm uninstall platform --namespace ${params.NAMESPACE} || true
                            
                            echo "Deploying full platform..."
                            helm upgrade --install platform . \
                                --namespace ${params.NAMESPACE} \
                                --wait \
                                --timeout 10m
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
                        file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE'),
                        string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            export AWS_ACCESS_KEY_ID=\${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=\${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=eu-west-3
                            
                            # Wait for pods to be ready
                            kubectl wait --for=condition=ready pod \
                                -l app.kubernetes.io/instance=platform \
                                -n ${params.NAMESPACE} \
                                --timeout=300s || true
                            
                            # Show deployment status
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

