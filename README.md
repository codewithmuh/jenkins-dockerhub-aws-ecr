# 🐳 Jenkins Docker CI/CD Pipeline – Multi-Registry + Trivy Scan

This project demonstrates a **Jenkins-based CI/CD pipeline** to:

- ✅ Build Docker images  
- 📦 Push to **DockerHub** and **AWS ECR**  
- 🔍 Scan images using **Trivy**  
- 📧 Send email notifications with scan reports  

---

---

## ✅ Prerequisites

Make sure you have the following:

- Jenkins installed with Docker on the same host
- Jenkins user added to the `docker` group
- AWS CLI installed on Jenkins machine
- Jenkins plugins:
  - Pipeline
  - Docker Pipeline
  - Email Extension
- Jenkins Credentials:
  - `dockerhub-creds`: DockerHub username/password
  - `aws-access-key`, `aws-secret-key`
  - `smtp-email` (for sending scan reports)

---

## 🧪 Jenkinsfile Pipeline

```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME = "myapp"
    IMAGE_TAG = ""
    AWS_REGION = "us-east-1"
    ECR_REPO = "<aws_account_id>.dkr.ecr.${AWS_REGION}.amazonaws.com/myapp"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        """
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
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
          sh """
            aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
            aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
            aws configure set default.region ${AWS_REGION}

            aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${ECR_REPO}

            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
            docker push ${ECR_REPO}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Scan Image with Trivy') {
      steps {
        sh """
          apt-get update && apt-get install wget apt-transport-https gnupg lsb-release -y
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add -
          echo "deb https://aquasecurity.github.io/trivy-repo/deb \$(lsb_release -sc) main" | \
            tee -a /etc/apt/sources.list.d/trivy.list
          apt-get update && apt-get install trivy -y

          docker pull ${IMAGE_NAME}:${IMAGE_TAG}
          trivy image --format json -o trivy-report.json ${IMAGE_NAME}:${IMAGE_TAG}
        """
      }
    }

    stage('Send Email Report') {
      steps {
        mail bcc: '',
             body: 'Docker image scan report attached.',
             from: 'jenkins@example.com',
             replyTo: '',
             subject: "Trivy Scan Report: ${IMAGE_NAME}:${IMAGE_TAG}",
             to: 'your-team@example.com',
             attachmentsPattern: 'trivy-report.json'
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}
