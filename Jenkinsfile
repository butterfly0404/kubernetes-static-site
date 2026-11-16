
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

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds-new',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    powershell '''
                        $tempDockerConfig = Join-Path $env:TEMP ("jenkins-docker-" + [guid]::NewGuid().ToString("N"))

                        try {
                            # Get the actual Docker Desktop engine endpoint
                            $dockerHost = docker context inspect desktop-linux --format "{{.Endpoints.docker.Host}}"

                            if ($LASTEXITCODE -ne 0 -or !$dockerHost) {
                                throw "Could not retrieve Docker Desktop endpoint"
                            }

                            $dockerHost = $dockerHost.Trim()

                            Write-Output "Docker username: $env:DOCKER_USER"
                            Write-Output "Docker engine endpoint: $dockerHost"
                            Write-Output "Temporary Docker configuration will be used"

                            # Create clean temporary Docker configuration
                            New-Item -ItemType Directory -Path $tempDockerConfig -Force | Out-Null

                            $configFile = Join-Path $tempDockerConfig "config.json"

                            [System.IO.File]::WriteAllText(
                                $configFile,
                                '{"auths":{}}',
                                (New-Object System.Text.UTF8Encoding($false))
                            )

                            # Safe token fingerprint
                            Write-Output "Docker token length: $($env:DOCKER_TOKEN.Length)"

                            $sha = [System.Security.Cryptography.SHA256]::Create()
                            $bytes = [System.Text.Encoding]::UTF8.GetBytes($env:DOCKER_TOKEN)

                            $hash = [BitConverter]::ToString(
                                $sha.ComputeHash($bytes)
                            ).Replace("-", "")

                            Write-Output "JENKINS TOKEN FINGERPRINT: $hash"

                            $sha.Dispose()
                            $bytes = $null
                            $hash = $null

                            # Docker login using direct engine endpoint
                            $psi = New-Object System.Diagnostics.ProcessStartInfo
                            $psi.FileName = "docker.exe"
                            $psi.Arguments = "--config `"$tempDockerConfig`" --host `"$dockerHost`" login --username $env:DOCKER_USER --password-stdin"
                            $psi.UseShellExecute = $false
                            $psi.RedirectStandardInput = $true
                            $psi.RedirectStandardOutput = $true
                            $psi.RedirectStandardError = $true

                            $process = New-Object System.Diagnostics.Process
                            $process.StartInfo = $psi

                            [void]$process.Start()

                            $stdoutTask = $process.StandardOutput.ReadToEndAsync()
                            $stderrTask = $process.StandardError.ReadToEndAsync()

                            $process.StandardInput.WriteLine($env:DOCKER_TOKEN)
                            $process.StandardInput.Close()

                            $process.WaitForExit()

                            $stdout = $stdoutTask.Result
                            $stderr = $stderrTask.Result

                            if ($stdout) {
                                Write-Output $stdout
                            }

                            if ($process.ExitCode -ne 0) {
                                throw "Docker login failed: $stderr"
                            }

                            Write-Output "Docker Hub login successful!"

                            # Tag image
                            $image = "$($env:DOCKER_USER)/static-website:$env:BUILD_NUMBER"

                            docker tag static-website:latest $image

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker tag failed"
                            }

                            # Push using the same endpoint and temporary config
                            docker --config $tempDockerConfig --host $dockerHost push $image

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker push failed"
                            }

                            Write-Output "Docker image pushed successfully!"
                        }
                        finally {
                            # Remove temporary credentials
                            if (Test-Path $tempDockerConfig) {
                                Remove-Item $tempDockerConfig -Recurse -Force
                            }

                            $env:DOCKER_TOKEN = $null
                        }
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                powershell '''
                    kubectl apply -f deployment.yaml

                    if ($LASTEXITCODE -ne 0) {
                        throw "Deployment configuration failed"
                    }

                    kubectl apply -f service.yaml

                    if ($LASTEXITCODE -ne 0) {
                        throw "Service configuration failed"
                    }

                    $image = "alishri/static-website:$env:BUILD_NUMBER"

                    kubectl set image deployment/static-website static-website=$image

                    if ($LASTEXITCODE -ne 0) {
                        throw "Image update failed"
                    }

                    kubectl rollout status deployment/static-website --timeout=180s

                    if ($LASTEXITCODE -ne 0) {
                        throw "Deployment failed"
                    }

                    Write-Output "Kubernetes deployment successful!"
                '''
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
