/**
 * ScrumOps CI/CD Pipeline
 * 
 * Simplified Jenkins pipeline for automated application deployment to Azure Kubernetes Service.
 * This pipeline handles source code checkout, containerization, and deployment with minimal overhead.
 * 
 * Key Features:
 * - Automatic project type detection (Node.js, Maven, Python, Static)
 * - Dynamic Dockerfile generation for containerization
 * - Azure Container Registry integration for image storage
 * - Azure Kubernetes Service deployment with ingress configuration
 * - Simplified configuration with essential parameters only
 */
pipeline {
  agent any

  options {
    disableConcurrentBuilds() // Prevent concurrent builds of the same job
    timestamps() // Add timestamps to console output
    buildDiscarder(logRotator(numToKeepStr: '30')) // Keep only last 30 builds
  }

  parameters {
    string(name: 'GITHUB_URL', defaultValue: '', description: 'GitHub repository URL')
    string(name: 'PROJECT_NAME', defaultValue: '', description: 'Project name for deployment')
    string(name: 'IMAGE_NAME', defaultValue: '', description: 'Custom Docker image name (optional)')
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to deploy')
  }

  environment {
    // Azure Container Registry configuration
    ACR_SERVER = 'devopsmonitoracrrt2y5a.azurecr.io'

    // Dynamic environment variables set during pipeline execution
    PROJECT_NAME = "${params.PROJECT_NAME}"
    GITHUB_URL = "${params.GITHUB_URL}"
    IMAGE_TAG = "${BUILD_NUMBER}"
    NAMESPACE = "${params.PROJECT_NAME}-dev"
  }

  stages {
    stage('Validate Parameters') {
      steps {
        script {
          // Validate required parameters
          if (!params.GITHUB_URL?.trim()) {
            error('GitHub URL is required for deployment')
          }
          if (!params.PROJECT_NAME?.trim()) {
            error('Project name is required for deployment')
          }

          echo "Starting deployment for project: ${params.PROJECT_NAME}"
          echo "Source repository: ${params.GITHUB_URL}"
          echo "Target branch: ${params.BRANCH}"
        }
      }
    }

    stage('Checkout Source Code') {
      steps {
        echo "Checking out source code from: ${params.GITHUB_URL}"
        deleteDir() // Clean workspace before checkout

        script {
          try {
            // Attempt to checkout specified branch
            git branch: params.BRANCH, url: params.GITHUB_URL
          } catch (err) {
            echo "Branch '${params.BRANCH}' not found, trying 'main'"
            try {
              git branch: 'main', url: params.GITHUB_URL
            } catch (err2) {
              echo "Branch 'main' not found, trying 'master'"
              git branch: 'master', url: params.GITHUB_URL
            }
          }

          // Capture git metadata for deployment tracking
          env.GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          echo "Checked out commit: ${env.GIT_COMMIT_HASH}"
        }
      }
    }

    stage('Detect Project Type') {
      steps {
        script {
          echo "Analyzing project structure for deployment configuration"

          // Detect project type based on configuration files
          if (fileExists('package.json')) {
            env.PROJECT_TYPE = 'nodejs'
            env.PORT = '3000'
            echo "Detected Node.js project"
          } else if (fileExists('pom.xml')) {
            env.PROJECT_TYPE = 'maven'
            env.PORT = '8080'
            echo "Detected Maven Spring Boot project"
          } else if (fileExists('requirements.txt')) {
            env.PROJECT_TYPE = 'python'
            env.PORT = '8000'
            echo "Detected Python project"
          } else if (fileExists('index.html')) {
            env.PROJECT_TYPE = 'static'
            env.PORT = '80'
            echo "Detected static website project"
          } else {
            env.PROJECT_TYPE = 'static'
            env.PORT = '80'
            echo "Project type unknown, defaulting to static hosting"
          }
        }
      }
    }

    stage('Prepare Container Configuration') {
      steps {
        script {
          if (!fileExists('Dockerfile')) {
            echo "Generating optimized Dockerfile for ${env.PROJECT_TYPE} project"

            def dockerfileContent = ''

            switch(env.PROJECT_TYPE) {
              case 'nodejs':
                // NOTE: Use npm ci if package-lock.json exists; otherwise fallback to npm install
                dockerfileContent = '''
FROM node:18-alpine
WORKDIR /app

# Copy package manifests first for better layer caching
COPY package*.json ./

# If a lockfile exists, prefer a clean, reproducible install.
# Otherwise, do a regular install (use --omit=dev to mirror previous --only=production).
RUN if [ -f package-lock.json ]; then \
      npm ci --omit=dev; \
    else \
      npm install --omit=dev; \
    fi

# Copy the rest of the source
COPY . .

EXPOSE 3000
CMD ["npm", "start"]
'''
                break

              case 'maven':
                dockerfileContent = '''
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
'''
                break

              case 'python':
                dockerfileContent = '''
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
'''
                break

              default: // static
                dockerfileContent = '''
FROM nginx:alpine
COPY . /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''
            }

            writeFile file: 'Dockerfile', text: dockerfileContent
            echo "Generated Dockerfile for containerization"
          } else {
            echo "Using existing Dockerfile from repository"
          }
        }
      }
    }

    stage('Build Application') {
      steps {
        script {
          // Build application based on project type before containerization
          switch(env.PROJECT_TYPE) {
            case 'maven':
              echo "Building Maven project"
              sh 'mvn clean package -DskipTests'
              break

            case 'nodejs':
              // No npm on agent; installs happen in Dockerfile

              echo "Skipping Node.js prebuild; handled inside Docker image"
              break

            default:
              echo "No build step required for ${env.PROJECT_TYPE} project"
          }
        }
      }
    }

    stage('Build Container Image') {
      steps {
        script {
          // Use custom image name if provided, otherwise use project name
          def imageName = params.IMAGE_NAME?.trim() ? params.IMAGE_NAME : params.PROJECT_NAME
          def fullImageName = "${env.ACR_SERVER}/${imageName}:${env.IMAGE_TAG}"

          echo "Building container image: ${fullImageName}"

          sh """
            docker build -t ${fullImageName} .
            docker tag ${fullImageName} ${env.ACR_SERVER}/${imageName}:latest
          """

          // Store image name for later stages
          env.FULL_IMAGE_NAME = fullImageName
          env.IMAGE_NAME_USED = imageName
        }
      }
    }

    stage('Push to Container Registry') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'acr-credentials',
                                          usernameVariable: 'ACR_USERNAME',
                                          passwordVariable: 'ACR_PASSWORD')]) {
          script {
            echo "Pushing container image to Azure Container Registry"

            sh """
              echo "${ACR_PASSWORD}" | docker login "${ACR_SERVER}" -u "${ACR_USERNAME}" --password-stdin
              docker push ${env.FULL_IMAGE_NAME}
              docker push ${env.ACR_SERVER}/${env.IMAGE_NAME_USED}:latest
            """
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')
        ]) {
          script {
            echo "Deploying application to Azure Kubernetes Service"

            // Ensure kubectl is available
            sh '''
              # Install kubectl locally in workspace if not available globally
              if ! command -v kubectl >/dev/null 2>&1; then
                echo "Installing kubectl locally..."
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                # Add to PATH for this session
                export PATH="$PWD:$PATH"
              fi
              
              # Verify kubectl installation
              kubectl version --client || ./kubectl version --client
            '''

            // Generate Kubernetes deployment manifests
            def k8sManifests = """
apiVersion: v1
kind: Namespace
metadata:
  name: ${env.NAMESPACE}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${env.PROJECT_NAME}
  namespace: ${env.NAMESPACE}
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: ${env.PROJECT_NAME}
  template:
    metadata:
      labels:
        app: ${env.PROJECT_NAME}
    spec:
      imagePullSecrets:
      - name: acr-auth
      containers:
      - name: ${env.PROJECT_NAME}
        image: ${env.FULL_IMAGE_NAME}
        ports:
        - containerPort: ${env.PORT}
        readinessProbe:
          httpGet:
            path: /
            port: ${env.PORT}
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: ${env.PORT}
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: ${env.PROJECT_NAME}-service
  namespace: ${env.NAMESPACE}
spec:
  selector:
    app: ${env.PROJECT_NAME}
  ports:
  - port: 80
    targetPort: ${env.PORT}
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${env.PROJECT_NAME}-ingress
  namespace: ${env.NAMESPACE}
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: ${env.PROJECT_NAME}.172.191.181.1.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${env.PROJECT_NAME}-service
            port:
              number: 80
"""

            // Setup Kubernetes environment and deploy
            sh '''
              export KUBECONFIG="${KUBECONFIG_FILE}"
              export PATH="$PWD:$PATH"
              
              # Use kubectl or local kubectl
              KUBECTL_CMD="kubectl"
              if ! command -v kubectl >/dev/null 2>&1 && [ -x "./kubectl" ]; then
                KUBECTL_CMD="./kubectl"
              fi
              
              # Create namespace if it doesn't exist
              $KUBECTL_CMD get ns "${NAMESPACE}" >/dev/null 2>&1 || $KUBECTL_CMD create ns "${NAMESPACE}"
              
              # Create or update container registry secret
              $KUBECTL_CMD -n "${NAMESPACE}" create secret docker-registry acr-auth \
                --docker-server="${ACR_SERVER}" \
                --docker-username="${ACR_USERNAME}" \
                --docker-password="${ACR_PASSWORD}" \
                --dry-run=client -o yaml | $KUBECTL_CMD apply -f -
            '''

            // Write and apply manifests
            writeFile file: 'k8s-deployment.yaml', text: k8sManifests

            timeout(time: 10, unit: 'MINUTES') {
              sh '''
                export KUBECONFIG="${KUBECONFIG_FILE}"
                export PATH="$PWD:$PATH"
                
                # Use kubectl or local kubectl
                KUBECTL_CMD="kubectl"
                if ! command -v kubectl >/dev/null 2>&1 && [ -x "./kubectl" ]; then
                  KUBECTL_CMD="./kubectl"
                fi
                
                $KUBECTL_CMD apply -f k8s-deployment.yaml
                $KUBECTL_CMD rollout status deployment/${PROJECT_NAME} -n ${NAMESPACE} --timeout=300s
              '''
            }
          }
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
          script {
            echo "Verifying deployment and generating access URL"

            sh '''
              export KUBECONFIG="${KUBECONFIG_FILE}"
              export PATH="$PWD:$PATH"
              
              # Use kubectl or local kubectl
              KUBECTL_CMD="kubectl"
              if ! command -v kubectl >/dev/null 2>&1 && [ -x "./kubectl" ]; then
                KUBECTL_CMD="./kubectl"
              fi
              
              # Wait for ingress to be ready and get URL
              for i in {1..20}; do
                INGRESS_HOST=$($KUBECTL_CMD get ingress ${PROJECT_NAME}-ingress -n ${NAMESPACE} -o jsonpath='{.spec.rules[0].host}' 2>/dev/null || echo "")
                if [ -n "$INGRESS_HOST" ] && [ "$INGRESS_HOST" != "null" ]; then
                  LIVE_URL="http://$INGRESS_HOST"
                  echo "✅ Application successfully deployed and accessible at: $LIVE_URL"
                  break
                fi
                echo "Waiting for ingress configuration (${i}/20)..."
                sleep 10
              done
              
              # Display deployment status
              echo "=== Deployment Summary ==="
              $KUBECTL_CMD get all -n ${NAMESPACE}
            '''
          }
        }
      }
    }
  }

  post {
    always {
      // Clean up local Docker images to save disk space
      script {
        if (env.FULL_IMAGE_NAME) {
          sh "docker rmi ${env.FULL_IMAGE_NAME} 2>/dev/null || true"
          sh "docker rmi ${env.ACR_SERVER}/${env.IMAGE_NAME_USED}:latest 2>/dev/null || true"
        }
      }
    }

    success {
      echo " Deployment completed successfully!"
      echo "Your application is now running on Azure Kubernetes Service."
    }

    failure {
      echo " Deployment failed. Check the logs above for details."
      withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
        sh '''
          # Ensure kubectl is available for troubleshooting
          if ! command -v kubectl >/dev/null 2>&1 && [ ! -x "./kubectl" ]; then
            echo "Installing kubectl for troubleshooting..."
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" || true
            chmod +x kubectl || true
          fi
          
          export KUBECONFIG="${KUBECONFIG_FILE}"
          export PATH="$PWD:$PATH"
          echo "=== Troubleshooting Information ==="
          
          # Use kubectl or local kubectl
          KUBECTL_CMD="kubectl"
          if ! command -v kubectl >/dev/null 2>&1 && [ -x "./kubectl" ]; then
            KUBECTL_CMD="./kubectl"
          fi
          
          # Only run kubectl commands if kubectl is available
          if command -v $KUBECTL_CMD >/dev/null 2>&1 || [ -x "./$KUBECTL_CMD" ]; then
            $KUBECTL_CMD -n "${NAMESPACE}" describe deployment "${PROJECT_NAME}" || true
            $KUBECTL_CMD -n "${NAMESPACE}" get pods -o wide || true
            $KUBECTL_CMD -n "${NAMESPACE}" get events --sort-by=.lastTimestamp | tail -20 || true
          else
            echo "kubectl not available for troubleshooting"
            echo "Please check Jenkins server configuration and ensure kubectl is installed"
          fi
        '''
      }
    }
  }
}
