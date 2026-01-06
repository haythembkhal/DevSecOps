pipeline {
  agent any

  tools {
    jdk 'jdk-17'         // CHANGE_ME: Jenkins JDK name
    maven 'maven-3.9'    // CHANGE_ME: Jenkins Maven name
  }

  environment {
    APP_NAME = 'spring-boot-app'
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    STAGING_COMPOSE_FILE = 'docker-compose.staging.yml'

    // Nexus (Docker registry + Maven repo)
    NEXUS_DOCKER_REGISTRY = 'https://nexus.example.com'             // CHANGE_ME
    NEXUS_DOCKER_REPO     = 'nexus.example.com/repo/spring-boot-app'// CHANGE_ME
    NEXUS_MAVEN_REPO_URL  = 'https://nexus.example.com/repository/maven-releases' // CHANGE_ME

    // SonarQube
    SONARQUBE_SERVER = 'sonarqube-server'  // CHANGE_ME (Jenkins global config)

    // Staging app URL for ZAP
    STAGING_BASE_URL = 'http://localhost:8080' // CHANGE_ME if exposed elsewhere
  }

  options {
    timeout(time: 45, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '25'))
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Unit Tests') {
      steps {
        sh '''
          mvn -B -q versions:set -DnewVersion=${IMAGE_TAG}
          mvn -B clean verify
        '''
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage('SAST - SonarQube') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh '''
            mvn -B sonar:sonar \
              -Dsonar.projectKey=${APP_NAME} \
              -Dsonar.projectName=${APP_NAME} \
              -Dsonar.projectVersion=${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          timeout(time: 10, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Quality Gate failed: ${qg.status}"
            }
          }
        }
      }
    }

    stage('SCA - OWASP Dependency-Check') {
      steps {
        sh '''
          mvn -B org.owasp:dependency-check-maven:check \
             -Dformat=HTML \
             -DoutputDirectory=target/dependency-check-report \
             -DfailOnCVSS=7
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/dependency-check-report/*.html', fingerprint: true
        }
      }
    }

    stage('Publish Artifact to Nexus (Maven)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-maven', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            mvn -B -DskipTests deploy \
              -Dnexus.url=${NEXUS_MAVEN_REPO_URL} \
              -DrepositoryId=nexus-releases \
              -Dnexus.username=$NEXUS_USER \
              -Dnexus.password=$NEXUS_PASS
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          docker build -t ${NEXUS_DOCKER_REPO}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Docker Push (Nexus)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login ${NEXUS_DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin
            docker push ${NEXUS_DOCKER_REPO}:${IMAGE_TAG}
            docker logout ${NEXUS_DOCKER_REGISTRY}
          '''
        }
      }
    }

    stage('Deploy to Staging (Docker Compose)') {
      steps {
        sh '''
          docker compose -f ${STAGING_COMPOSE_FILE} down || true
          IMAGE=${NEXUS_DOCKER_REPO}:${IMAGE_TAG} docker compose -f ${STAGING_COMPOSE_FILE} up -d
          curl -sS --retry 10 --retry-delay 3 ${STAGING_BASE_URL}/actuator/health || (docker compose -f ${STAGING_COMPOSE_FILE} logs && exit 1)
        '''
      }
    }

    stage('DAST - OWASP ZAP (Staging)') {
      steps {
        sh '''
          mkdir -p target/zap
          docker run --rm -v $(pwd)/target/zap:/zap/wrk owasp/zap2docker-stable \
            zap-baseline.py -t ${STAGING_BASE_URL} -r zap_baseline.html -J zap_baseline.json -I -m 100
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/zap/*', fingerprint: true
        }
        unsuccessful {
          error "DAST failed (ZAP baseline). Check target/zap reports."
        }
      }
    }
  }

  post {
    success { echo "CI/CD passed. Image pushed: ${NEXUS_DOCKER_REPO}:${IMAGE_TAG}" }
    failure { echo "Pipeline failed. Inspect logs & archived reports." }
  }
}
