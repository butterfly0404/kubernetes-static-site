
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    docker build -t static-website:latest .

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image build failed"
                    }

                    Write-Output "Docker image built successfully!"
                '''
            }
        }

        stage('Test Application') {
            steps {
                powershell '''
                    docker rm -f ci-test 2>$null

                    docker run -d -p 8083:80 --name ci-test static-website:latest

                    if ($LASTEXITCODE -ne 0) {
                        throw "Container startup failed"
                    }

                    try {
                        $response = Invoke-WebRequest http://localhost:8083 -UseBasicParsing

                        if ($response.StatusCode -ne 200) {
                            throw "Application test failed"
                        }

                        Write-Output "Application test passed!"
                    }
                    finally {
                        docker rm -f ci-test
                    }
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                powershell '''
                    Write-Output "Checking Kubernetes connection..."

                    kubectl get nodes

                    if ($LASTEXITCODE -ne 0) {
                        throw "Kubernetes is not accessible"
                    }

                    Write-Output "Applying Kubernetes deployment..."

                    kubectl apply -f deployment.yaml

                    if ($LASTEXITCODE -ne 0) {
                        throw "Deployment configuration failed"
                    }

                    kubectl apply -f service.yaml

                    if ($LASTEXITCODE -ne 0) {
                        throw "Service configuration failed"
                    }

                    Write-Output "Updating application image..."

                    kubectl set image deployment/static-website static-website=static-website:latest

                    if ($LASTEXITCODE -ne 0) {
                        throw "Image update failed"
                    }

                    Write-Output "Restarting deployment..."

                    kubectl rollout restart deployment/static-website

                    if ($LASTEXITCODE -ne 0) {
                        throw "Deployment restart failed"
                    }

                    kubectl rollout status deployment/static-website --timeout=180s

                    if ($LASTEXITCODE -ne 0) {
                        throw "Kubernetes rollout failed"
                    }

                    Write-Output "Checking pods..."

                    kubectl get pods -o wide

                    Write-Output "Checking services..."

                    kubectl get services

                    Write-Output "Kubernetes deployment completed successfully!"
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: CI/CD pipeline completed!'
            echo 'Application built, tested and deployed to Kubernetes.'
        }

        failure {
            echo 'Pipeline failed. Check the stage that reported the error.'
        }
    }
}
