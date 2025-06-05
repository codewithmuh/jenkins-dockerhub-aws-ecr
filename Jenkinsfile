pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "myapp"
    SHA_TAG = ""
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          SHA_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker build -t ${DOCKER_IMAGE}:${SHA_TAG} .
        """
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
          sh """
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
            docker tag ${DOCKER_IMAGE}:${SHA_TAG} $DOCKER_USERNAME/${DOCKER_IMAGE}:${SHA_TAG}
            docker push $DOCKER_USERNAME/${DOCKER_IMAGE}:${SHA_TAG}
          """
        }
      }
    }

    stage('Push to AWS ECR') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          script {
            def region = 'us-east-1' // change to your AWS region
            def accountId = '123456789012' // change to your AWS account ID
            def repoUri = "${accountId}.dkr.ecr.${region}.amazonaws.com/${DOCKER_IMAGE}"

            sh """
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws configure set default.region ${region}

              aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${repoUri}
              docker tag ${DOCKER_IMAGE}:${SHA_TAG} ${repoUri}:${SHA_TAG}
              docker push ${repoUri}:${SHA_TAG}
            """
          }
        }
      }
    }

    stage('(Optional) Trivy Scan') {
      when {
        expression { return fileExists('Dockerfile') }
      }
      steps {
        sh """
          trivy image -f json -o trivy-report.json ${DOCKER_IMAGE}:${SHA_TAG} || true
        """
      }
    }

    stage('(Optional) Send Email Report') {
      when {
        expression { return fileExists('trivy-report.json') }
      }
      steps {
        emailext(
          subject: "Trivy Scan Report for ${DOCKER_IMAGE}:${SHA_TAG}",
          body: "Please find attached the security scan report.",
          to: "dev-team@example.com",
          attachmentsPattern: 'trivy-report.json'
        )
      }
    }
  }

  post {
    failure {
      mail to: 'dev-team@example.com',
           subject: "Pipeline Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
           body: "Something went wrong in the Jenkins pipeline.\nCheck the logs."
    }
  }
}
