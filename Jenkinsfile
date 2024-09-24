pipeline {
  agent any
  tools {
    jdk 'jdk17'
    nodejs 'node16'
  }
  environment {
    SCANNER_HOME = tool 'sonar-scanner'
  }
  stages {
    stage('clean workspace') {
      steps {
        cleanWs()
      }
    }
    stage('Checkout from Git') {
      steps {
        git branch: 'master', url: 'https://github.com/anil8055/2048.git'
      }
    }
    stage("Sonarqube Analysis ") {
      steps {
        withSonarQubeEnv('sonar-server') {
          sh ''
          ' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Game \
                    -Dsonar.projectKey=Game '
          ''
        }
      }
    }
    stage("quality gate") {
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
      environment {
        NVD_API_KEY = 'f55157f3-eaff-43ff-91c6-3c3a694c091c' // Your NVD API Key
      }
      steps {
        dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }
    stage("Docker Build & Push") {
      steps {
        script {
          withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
            sh "docker build -t 2048 ."
            sh "docker tag 2048 jenkinsgame:latest "
            sh "docker push jenkinsgame:latest "
          }
        }
      }
    }
    stage("TRIVY") {
      steps {
        sh "trivy image jenkinsgame:latest > trivy.txt"
      }
    }
    stage('TRIVY FS SCAN') {
      steps {
        sh "trivy fs . > trivyfs.txt"
      }
    }
  }
}
