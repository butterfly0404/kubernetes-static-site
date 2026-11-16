
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
                        Write-Output "===== JENKINS DOCKER ENVIRONMENT ====="

                        Write-Output "Docker username: $env:DOCKER_USER"
                        Write-Output "USERPROFILE: $env:USERPROFILE"
                        Write-Output "DOCKER_CONFIG: $env:DOCKER_CONFIG"
                        Write-Output "DOCKER_CONTEXT: $env:DOCKER_CONTEXT"

                        Write-Output "Docker context:"
                        docker context show

                        $config = if ($env:DOCKER_CONFIG) {
                            Join-Path $env:DOCKER_CONFIG "config.json"
                        } else {
                            Join-Path $env:USERPROFILE ".docker\config.json"
                        }

                        Write-Output "Docker config path: $config"

                        if (Test-Path $config) {
                            $json = Get-Content $config -Raw | ConvertFrom-Json

                            Write-Output "Credential store: $($json.credsStore)"
                            Write-Output "Credential helpers: $($json.credHelpers | ConvertTo-Json -Compress)"
                        } else {
                            Write-Output "Docker config file not found"
                        }

                        Write-Output "===== END ENVIRONMENT CHECK ====="

                        # Safe credential diagnostics
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

                        # Docker Hub login
                        $psi = New-Object System.Diagnostics.ProcessStartInfo
                        $psi.FileName = "docker.exe"
                        $psi.Arguments = "login --username $env:DOCKER_USER --password-stdin"
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
                        docker tag static-website:latest "$($env:DOCKER_USER)/static-website:$($env:BUILD_NUMBER)"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker tag failed"
                        }

                        # Push image
                        docker push "$($env:DOCKER_USER)/static-website:$($env:BUILD_NUMBER)"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker push failed"
                        }

                        Write-Output "Docker image pushed successfully!"
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

                    kubectl set image deployment/static-website static-website="alishri/static-website:$env:BUILD_NUMBER"

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
