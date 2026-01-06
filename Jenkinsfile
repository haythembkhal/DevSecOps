
pipeline {
  agent any

  environment {
    APP_NAME = 'devsecops-spring-boot'
    DOCKER_IMAGE = "${APP_NAME}:${env.GIT_COMMIT ?: 'local'}"
    DOCKER_IMAGE_LATEST = "${APP_NAME}:latest"
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build, Test & SCA (Dependency-Check)') {
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

    stage('SAST (SonarQube) + Quality Gate') {
      when { expression { return env.SONAR_HOST_URL && env.SONAR_TOKEN } }
      steps {
        // Run analysis via Maven (as before)
        sh '''#!/bin/bash -eux
        docker run --rm -e SONAR_HOST_URL -e SONAR_TOKEN \
          -v "$PWD":/wrk -w /wrk maven:3.9.9-eclipse-temurin-17 \
          mvn -B sonar:sonar \
            -Dsonar.projectKey=devsecops-spring-boot \
            -Dsonar.host.url=${SONAR_HOST_URL} \
            -Dsonar.login=${SONAR_TOKEN}
        '''
      }
      post {
        // Enforce Quality Gate using Jenkins server binding
        success {
          script {
            // Name must match your configured SonarQube server in Jenkins global config
            withSonarQubeEnv('SonarQubeServer') {
              // This waits for background task and fails the build if gate fails/timeouts
              def qg = waitForQualityGate()
              if (qg.status != 'OK') {
                error "Quality Gate failed: ${qg.status}"
              }
            }
          }
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''#!/bin/bash -eux
        docker build -t ${DOCKER_IMAGE} -t ${DOCKER_IMAGE_LATEST} .
        docker images | grep ${APP_NAME}
        '''
      }
    }

    stage('Docker Push (optional)') {
      when { expression { return env.DOCKER_REGISTRY && env.DOCKER_USERNAME && env.DOCKER_PASSWORD } }
      steps {
        sh '''#!/bin/bash -eux
        echo "Logging into ${DOCKER_REGISTRY}"
        echo "$DOCKER_PASSWORD" | docker login ${DOCKER_REGISTRY} -u "$DOCKER_USERNAME" --password-stdin
        docker tag ${DOCKER_IMAGE} ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT}
        docker tag ${DOCKER_IMAGE_LATEST} ${DOCKER_REGISTRY}/${APP_NAME}:latest
        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT}
        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
        '''
      }
    }

    stage('Upload artifact to Nexus (optional)') {
      when { expression { return env.NEXUS_URL && env.NEXUS_REPO_PATH && env.NEXUS_USER && env.NEXUS_PASSWORD } }
      steps {
        sh '''#!/bin/bash -eux
        JAR=$(ls -1 target/*-SNAPSHOT.jar 2>/dev/null || true)
        if [ -z "$JAR" ]; then JAR=$(ls -1 target/*-*.jar | head -n1); fi
        [ -n "$JAR" ]
        FILENAME=$(basename "$JAR")
        curl -u "$NEXUS_USER:$NEXUS_PASSWORD" --upload-file "$JAR" \
          "$NEXUS_URL/$NEXUS_REPO_PATH/devsecops/${BUILD_NUMBER}/${FILENAME}"
        '''
      }
    }

    // Optional: deploy to staging prior to DAST
    stage('Deploy to Staging (optional)') {
      when { expression { return env.STAGING_DEPLOY_SCRIPT || env.KUBE_CONTEXT || env.DOCKER_HOST } }
      steps {
        sh '''#!/bin/bash -eux
        # This stage is intentionally generic and gated by env vars.
        # Examples:
        # - If you deploy via a script provided by your team:
        #     $STAGING_DEPLOY_SCRIPT
        # - If you deploy to Docker host:
        #     docker pull ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT}
        #     docker stop ${APP_NAME} || true
        #     docker rm ${APP_NAME} || true
        #     docker run -d --name ${APP_NAME} -p 80:8080 ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT}
        # - If you deploy to Kubernetes (kubectl already configured via Jenkins agent/credentials):
        #     kubectl --context="${KUBE_CONTEXT}" set image deploy/${APP_NAME} ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT} --record
        #     kubectl --context="${KUBE_CONTEXT}" rollout status deploy/${APP_NAME} --timeout=120s

        if [ -n "${STAGING_DEPLOY_SCRIPT:-}" ]; then
          bash "${STAGING_DEPLOY_SCRIPT}"
        else
          echo "No staging deploy action configured. Skipping."
        fi
        '''
      }
    }

    stage('DAST (OWASP ZAP) against STAGING') {
      when { expression { return env.STAGING_URL } }
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
  }
}
