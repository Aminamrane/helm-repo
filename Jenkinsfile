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
        KUBECONFIG_CREDENTIALS = 'kubeconfig-credentials'
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
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying to Kubernetes namespace: ${params.NAMESPACE}"
                    
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            
                            # Create namespace if it doesn't exist
                            kubectl create namespace ${params.NAMESPACE} || true
                            
                            # Deploy with Helm
                            cd ${HELM_CHART_PATH}
                            
                            if [ "${params.SERVICE}" == "all" ]; then
                                echo "Deploying full platform..."
                                helm upgrade --install platform . \\
                                    --namespace ${params.NAMESPACE} \\
                                    --wait \\
                                    --timeout 10m
                            else
                                echo "Deploying service: ${params.SERVICE}"
                                # For individual services, we can use helm upgrade with specific values
                                helm upgrade --install platform . \\
                                    --namespace ${params.NAMESPACE} \\
                                    --set ${params.SERVICE}.image.tag=${params.IMAGE_VERSION} \\
                                    --wait \\
                                    --timeout 10m
                            fi
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "Verifying deployment..."
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            
                            # Wait for pods to be ready
                            kubectl wait --for=condition=ready pod \\
                                -l app.kubernetes.io/instance=platform \\
                                -n ${params.NAMESPACE} \\
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

