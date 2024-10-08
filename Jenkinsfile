pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'master', url: 'https://github.com/anil8055/2048.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Game \
                    -Dsonar.projectKey=Game '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build -t 2048 ."
                       sh "docker tag 2048 rapidgenius/2048:latest "
                       sh "docker push rapidgenius/2048:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image rapidgenius/2048:latest > trivy.txt"
            }
        }
      stage('DeployToProduction') {
    steps {
        kubeconfig(caCertificate: 'MIIDBjCCAe6gAwIBAgIBATANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwptaW5pa3ViZUNBMB4XDTI0MDkyMjE0NDQ1OVoXDTM0MDkyMTE0NDQ1OVowFTETMBEGA1UEAxMKbWluaWt1YmVDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJ+IYVfa84PbhYVU2XF2jyodmAwvcLC3gvCNAztt+94Psuno7WiTlTGZm6Bhllb49E4HjL62wgKwfi29D0Fl2N4tr3Rktq4an8bmAK01mw6iDm5S7gitu/vmpD/r0TSfzFW8vbd48vDA2KtNThJDdoCYfHT6dozD0XM6H9ujhcrJE7mABfIW7C4vE/CTTmhiZ2FbVI33vKBr4GTqWONnQhqaMczGpoDvoYO5lxP+UigAbqdZnLOJM2DZc7Q793kddhL2K0SiioY+oMbKsP+2KqL+MJqFuzNmc5VP10sOc7gDj/cjC+yOGPBQrpqdV5MJ/PqLXhpxWQzeZrCYP7HFZM0CAwEAAaNhMF8wDgYDVR0PAQH/BAQDAgKkMB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBRSGB2RCw7zNj5jAxdPO4IdWOligjANBgkqhkiG9w0BAQsFAAOCAQEAJ2sU9+/m0aDAwqg0ziWU1nPZDVsOo8mOIv0uT55L3eqXFjfaYw79H9OBTm8zZTIsoCqkMc9/eFVgLoMUDn11NtYu6nx/YuRXQpW6FNQPUt+DeRVRw9d3bSVFD8M1EYVakxmpE8VSm5wEy6qazLLlF8GxJnyvfZpd2n7GWD94ntu0EOXhHFN+uFvoa37Ws8jloU/dtGTN0NS4oAf/2c1bsa/fgJb6zLKIQgxX6dRZqEv/1D+RDZFC3OkC3itewf9QC8hXdJAnhp7SFDVgj9/3nfKGrS5NMB4IIjVhCJgD6vY+77GMOOWwE77zeIqriV3U05RdQfr9CxjfjMFMJcDUjw==', credentialsId: 'ac4c6002-37d8-4172-bff2-232ab87dbf76', serverUrl: 'https://192.168.49.2:8443') {
            // Apply the manifests
            sh 'kubectl apply -f app.yaml'
            sh 'kubectl apply -f app-service.yaml'
            sh 'kubectl rollout restart deployment rapidgenius-2048'

            // Wait for the pod to be ready
            script {
                def podReady = false
                timeout(time: 5, unit: 'MINUTES') { // Set a timeout to avoid indefinite waiting
                    while (!podReady) {
                        sleep(10) // Wait for 10 seconds before checking again
                        def podStatus = sh(script: 'kubectl get pods -l app=ab -o jsonpath="{.items[0].status.phase}"', returnStdout: true).trim()
                        
                        if (podStatus == 'Running') {
                            podReady = true
                        } else if (podStatus == 'CrashLoopBackOff' || podStatus == 'Failed') {
                            error "Pod is in an error state: ${podStatus}. Check the logs for details."
                        }

                        echo "Current pod status: ${podStatus}"
                    }
                }
            }

            // Port forward once the pod is ready
            script {
                // Start port forwarding in the background
                def portForwardProcess = sh(script: 'kubectl port-forward service/node-app-service 3000:3000 &', returnStdout: true).trim()
                echo "Started port forwarding process with PID: ${portForwardProcess}"

                // Check if the port forwarding is established
                sleep(5) // Wait a few seconds for the port forward to establish

                // Check if port forwarding is still active
                def isForwardingActive = sh(script: "lsof -i :3000", returnStatus: true) == 0
                if (isForwardingActive) {
                    echo "Port forwarding is established successfully."
                } else {
                    error "Port forwarding failed. Check the service and pod status."
                }
            }
        }
    }
}


    }
}
