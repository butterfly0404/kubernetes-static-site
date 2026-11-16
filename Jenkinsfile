
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
                '''
            }
        }

        stage('Test Application') {
            steps {
                powershell '''
                    docker run -d -p 8083:80 --name ci-test static-website:latest

                    try {
                        $response = Invoke-WebRequest http://localhost:8083 -UseBasicParsing
                        if ($response.StatusCode -ne 200) {
                            throw "Test failed"
                        }
                    }
                    finally {
                        docker rm -f ci-test
                    }
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    powershell '''
                        $env:DOCKER_TOKEN | docker login -u $env:DOCKER_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) { throw "Login failed" }

                        docker tag static-website:latest "$($env:DOCKER_USER)/static-website:$($env:BUILD_NUMBER)"
                        docker push "$($env:DOCKER_USER)/static-website:$($env:BUILD_NUMBER)"
                        if ($LASTEXITCODE -ne 0) { throw "Push failed" }
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    powershell '''
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml

                        kubectl set image deployment/static-website static-website="$($env:DOCKER_USER)/static-website:$($env:BUILD_NUMBER)"

                        if ($LASTEXITCODE -ne 0) { throw "Image update failed" }

                        kubectl rollout status deployment/static-website --timeout=180s
                        if ($LASTEXITCODE -ne 0) { throw "Deployment failed" }
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }
        failure {
            echo 'CI/CD pipeline failed. Check the console output.'
        }
    }
}
