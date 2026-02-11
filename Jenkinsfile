pipeline {
  agent any

  environment {
    APP_NAME = 'devsecops-spring-boot'
    IMAGE_NAME = "haytembk/${APP_NAME}"
    DOCKER_IMAGE = "${IMAGE_NAME}:${env.GIT_COMMIT ?: 'local'}"
    DOCKER_IMAGE_LATEST = "${IMAGE_NAME}:latest"

    SONAR_PROJECT_KEY = 'devsecops-spring-boot'
    SONARQUBE_SERVER_NAME = 'SonarQubeServer'

    STAGING_HOST = 'http://127.0.0.1'
    STAGING_PORT = '8080'
    STAGING_URL = "${STAGING_HOST}:${STAGING_PORT}"

    NEXUS_URL = 'http://127.0.0.1:8081'
    NEXUS_REPO_PATH = 'repository/releases'
  }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build, Test & Dependency-Check') {
      steps {
        sh '''#!/bin/bash -eux
        docker run --rm -v "$PWD":/wrk -w /wrk maven:3.9.9-eclipse-temurin-17 \
          mvn -B -DskipTests=false clean verify \
            org.owasp:dependency-check-maven:check
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/dependency-check-report.*', onlyIfSuccessful: false
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER_NAME}") {
          withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
            sh '''#!/bin/bash -eux
            docker run --rm \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" -e SONAR_TOKEN="$SONAR_TOKEN" \
              -v "$PWD":/wrk -w /wrk maven:3.9.9-eclipse-temurin-17 \
              mvn -B sonar:sonar \
                -Dsonar.projectKey=''' + "${SONAR_PROJECT_KEY}" + ''' \
                -Dsonar.host.url=${SONAR_HOST_URL} \
                -Dsonar.login=${SONAR_TOKEN}
            '''
          }
        }
      }
      post {
        success {
          script {
            withSonarQubeEnv("${SONARQUBE_SERVER_NAME}") {
              def qg = waitForQualityGate()
              if (qg.status != 'OK') { error "Quality Gate failed: ${qg.status}" }
            }
          }
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''#!/bin/bash -eux
        docker build -t ${DOCKER_IMAGE} -t ${DOCKER_IMAGE_LATEST} .
        docker images | grep ${IMAGE_NAME}
        '''
      }
    }

    stage('Deploy to Staging') {
      steps {
        sh '''#!/bin/bash -eux
        docker rm -f ${APP_NAME} || true
        docker run -d --name ${APP_NAME} -p ${STAGING_PORT}:8080 ${DOCKER_IMAGE}
        for i in {1..30}; do
          curl -fsS ${STAGING_URL} && exit 0 || true
          sleep 2
        done
        exit 1
        '''
      }
    }

    stage('OWASP ZAP DAST') {
      steps {
        sh '''#!/bin/bash -eux
        docker run --rm --network host -v "$PWD":/zap/wrk:rw \
          owasp/zap2docker-stable zap-baseline.py \
            -t "${STAGING_URL}" \
            -r zap-report.html \
            -x zap-report.xml \
            -m 10
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'zap-report.*', onlyIfSuccessful: false
        }
      }
    }

    stage('Docker Push') {
      when { expression { return env.PUBLISH_DOCKER?.toBoolean() } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
          sh '''#!/bin/bash -eux
          echo "$DOCKER_PASSWORD" | docker login docker.io -u "$DOCKER_USERNAME" --password-stdin
          docker tag ${DOCKER_IMAGE} ${IMAGE_NAME}:${GIT_COMMIT}
          docker tag ${DOCKER_IMAGE_LATEST} ${IMAGE_NAME}:latest
          docker push ${IMAGE_NAME}:${GIT_COMMIT}
          docker push ${IMAGE_NAME}:latest
          '''
        }
      }
    }

    stage('Upload to Nexus') {
      when { expression { return env.PUBLISH_NEXUS?.toBoolean() } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
          sh '''#!/bin/bash -eux
          JAR=$(ls -1 target/*-SNAPSHOT.jar 2>/dev/null || ls -1 target/*.jar | head -n1)
          FILENAME=$(basename "$JAR")
          curl -f -u "$NEXUS_USER:$NEXUS_PASSWORD" --upload-file "$JAR" \
            "${NEXUS_URL}/${NEXUS_REPO_PATH}/devsecops-spring-boot/${BUILD_NUMBER}/${FILENAME}"
          '''
        }
      }
    }
  }

  post {
    always {
      sh 'docker rm -f ${APP_NAME} || true'
      cleanWs()
    }
  }
}
