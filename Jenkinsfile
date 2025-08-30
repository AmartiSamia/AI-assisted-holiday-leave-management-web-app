pipeline {
  agent any

  // Auto build on push + poll every minute
  triggers {
    githubPush()
    pollSCM('H/1 * * * *') // every minute (hashed per job to avoid thundering herd)
  }

  options {
    disableConcurrentBuilds()
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    durabilityHint('MAX_SURVIVABILITY')
  }

  parameters {
    string(name: 'GITHUB_URL',   defaultValue: '', description: 'GitHub repo URL (https://github.com/you/repo)')
    string(name: 'PROJECT_NAME', defaultValue: '', description: 'Project slug (ACR repo + K8s app name)')
    string(name: 'DEPLOYMENT_ID', defaultValue: '', description: 'Optional deployment ID for callbacks')
    string(name: 'LB_RG', defaultValue: 'MC_devops-monitoring-rg_devops-aks_eastus', description: 'AKS managed RG for Azure LB annotation')
    string(name: 'LB_IP', defaultValue: '', description: 'Optional static public IP')
  }

  environment {
    // EDIT ONLY IF YOUR ACR SERVER CHANGES
    ACR_SERVER   = 'devopsmonitoracrrt2y5a.azurecr.io'

    // Will be (re)set in the auto-fill step
    PROJECT_NAME = "${params.PROJECT_NAME}"
    GITHUB_URL   = "${params.GITHUB_URL}"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    NAMESPACE    = "${params.PROJECT_NAME}-dev"

    // ScrumOps backend for callbacks - Jenkins should be able to reach this
    BACKEND_BASE = 'http://20.246.34.90:8081'
  }

  stages {

    stage('Auto-fill parameters (webhook runs)') {
      steps {
        script {
          // Discover Git URL from Jenkins env or local repo (when available)
          def discoveredGit = env.GIT_URL
          if (!discoveredGit?.trim()) {
            try {
              discoveredGit = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
            } catch (ignored) { /* first run may not have .git yet */ }
          }

          // If param empty, use discovered one
          env.GITHUB_URL = (params.GITHUB_URL?.trim()) ? params.GITHUB_URL.trim() : (discoveredGit ?: '')

          // Derive project name if missing (repo name or job base name)
          def derivedName = env.JOB_BASE_NAME
          if (env.GITHUB_URL?.trim()) {
            def bits = env.GITHUB_URL.split('/')
            derivedName = bits ? bits[-1].replaceAll(/\.git$/, '') : derivedName
          }
          env.PROJECT_NAME = (params.PROJECT_NAME?.trim()) ?: (derivedName ?: '')
          env.NAMESPACE    = env.PROJECT_NAME ? "${env.PROJECT_NAME}-dev" : "unknown-dev"
        }
      }
    }

    stage('Validate Parameters') {
      steps {
        echo "Validating parameters"
        script {
          if (!env.GITHUB_URL?.trim()) {
            error('GITHUB_URL is required and could not be auto-detected. Set it in the job or ensure the repo remote is available.')
          }
          if (!env.PROJECT_NAME?.trim()) {
            error('PROJECT_NAME is required and could not be inferred (e.g., from repo name or JOB_BASE_NAME).')
          }

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"validate","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Checkout') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"checkout","status":"running"}' || true"""
          }
        }

        echo "Cloning repository: ${env.GITHUB_URL}"
        deleteDir()

        script {
          try {
            git branch: 'main', url: env.GITHUB_URL
          } catch (err) {
            echo "Branch 'main' not found, trying 'master'"
            try {
              git branch: 'master', url: env.GITHUB_URL
            } catch (err2) {
              echo "Branch 'master' not found, trying 'develop'"
              git branch: 'develop', url: env.GITHUB_URL
            }
          }

          env.GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.GIT_AUTHOR      = sh(script: 'git log -1 --pretty=%an',    returnStdout: true).trim()

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"checkout","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Detect Project Type') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"analyze","status":"running"}' || true"""
          }

          sh 'echo "Repository file listing:" && ls -la'

          if (fileExists('package.json')) {
            env.PROJECT_TYPE = 'nodejs'
            env.PORT = '3000'
            echo "Detected project type: Node.js"
          } else if (fileExists('pom.xml')) {
            env.PROJECT_TYPE = 'maven'
            env.PORT = '8080'
            echo "Detected project type: Maven (Spring Boot)"
          } else if (fileExists('requirements.txt')) {
            env.PROJECT_TYPE = 'python'
            env.PORT = '8000'
            echo "Detected project type: Python"
          } else if (fileExists('index.html')) {
            env.PROJECT_TYPE = 'static'
            env.PORT = '80'
            echo "Detected project type: Static"
          } else {
            env.PROJECT_TYPE = 'static'
            env.PORT = '80'
            echo "Project type unknown; defaulting to Static"
          }

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"analyze","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Generate Dockerfile') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"prepare","status":"running"}' || true"""
          }

          if (!fileExists('Dockerfile')) {
            echo "Creating Dockerfile for ${env.PROJECT_TYPE}"
            def df = ''
            if (env.PROJECT_TYPE == 'nodejs') {
              df = '''FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production || npm install
COPY . .
EXPOSE 3000
CMD ["npm","start"]'''
            } else if (env.PROJECT_TYPE == 'maven') {
              df = '''FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java","-jar","app.jar"]'''
            } else if (env.PROJECT_TYPE == 'python') {
              df = '''FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt* ./
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python","app.py"]'''
            } else {
              df = '''FROM nginx:alpine
COPY . /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx","-g","daemon off;"]'''
            }
            writeFile file: 'Dockerfile', text: df
          } else {
            echo 'Using existing Dockerfile'
          }

          sh 'echo "---- Dockerfile ----"; cat Dockerfile; echo "-------------------"'

          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"prepare","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"build","status":"running"}' || true"""
          }
        }

        script {
          // Compile/build application if needed (before Docker build)
          if (env.PROJECT_TYPE == 'maven') {
            echo "Building Maven project (tests skipped)"
            sh 'mvn clean package -DskipTests'
          } else if (env.PROJECT_TYPE == 'nodejs') {
            echo "Building Node.js project"
            sh 'npm run build || echo "No build script found; proceeding"'
          }
        }

        sh """
          docker build -t ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} .
          docker tag   ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} ${ACR_SERVER}/${PROJECT_NAME}:latest
        """

        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"build","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Push to ACR') {
      steps {
        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"push","status":"running"}' || true"""
          }
        }

        withCredentials([usernamePassword(credentialsId: 'acr-credentials',
                                          usernameVariable: 'ACR_USERNAME',
                                          passwordVariable: 'ACR_PASSWORD')]) {
          sh """
            echo "${ACR_PASSWORD}" | docker login "${ACR_SERVER}" -u "${ACR_USERNAME}" --password-stdin
            docker push ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG}
            docker push ${ACR_SERVER}/${PROJECT_NAME}:latest
          """
        }

        script {
          if (params.DEPLOYMENT_ID?.trim()) {
            sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                 -H "Content-Type: application/json" \
                 -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"push","status":"success"}' || true"""
          }
        }
      }
    }

    stage('Setup kubectl (if needed)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            if ! command -v kubectl >/dev/null 2>&1 && [ ! -x "./kubectl" ]; then
              echo "Installing kubectl locally"
              VER=$(curl -fsSL https://dl.k8s.io/release/stable.txt)
              curl -fsSL -o kubectl "https://dl.k8s.io/release/${VER}/bin/linux/amd64/kubectl"
              chmod +x kubectl
            fi
            export PATH="$PWD:$PATH"
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl version --client
          '''
        }
      }
    }

    stage('Deploy to AKS (Rolling)') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')
        ]) {
          script {
            if (params.DEPLOYMENT_ID?.trim()) {
              sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"deploy","status":"running"}' || true"""
            }

            String k8sYaml = """
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
        image: ${env.ACR_SERVER}/${env.PROJECT_NAME}:${env.IMAGE_TAG}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: ${env.PORT}
        readinessProbe:
          httpGet:
            path: /
            port: ${env.PORT}
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
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
""".stripIndent()

            sh '''
              set -e
              export PATH="$PWD:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              # Namespace and image pull secret
              kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"

              kubectl -n "${NAMESPACE}" create secret docker-registry acr-auth \
                --docker-server="${ACR_SERVER}" \
                --docker-username="${ACR_USERNAME}" \
                --docker-password="${ACR_PASSWORD}" \
                --dry-run=client -o yaml | kubectl apply -f -
            '''

            writeFile file: 'k8s.yaml', text: k8sYaml

            timeout(time: 15, unit: 'MINUTES') {
              retry(2) {
                sh 'export PATH="$PWD:$PATH"; export KUBECONFIG="${KUBECONFIG_FILE}"; kubectl apply -f k8s.yaml'
              }
            }

            timeout(time: 10, unit: 'MINUTES') {
              retry(1) {
                sh 'export PATH="$PWD:$PATH"; export KUBECONFIG="${KUBECONFIG_FILE}"; kubectl rollout status deployment/${PROJECT_NAME} -n ${NAMESPACE} --timeout=300s'
              }
            }

            if (params.DEPLOYMENT_ID?.trim()) {
              sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"deploy","status":"success"}' || true"""
            }
          }
        }
      }
    }

    stage('Get External URL') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
          script {
            if (params.DEPLOYMENT_ID?.trim()) {
              sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"health-check","status":"running"}' || true"""
            }
          }

          sh '''
            set -e
            export PATH="$PWD:$PATH"
            export KUBECONFIG="${KUBECONFIG_FILE}"

            # Prefer Ingress host as the live URL
            for i in {1..30}; do
              INGRESS_HOST=$(kubectl get ingress ${PROJECT_NAME}-ingress -n ${NAMESPACE} -o jsonpath='{.spec.rules[0].host}' 2>/dev/null || echo "")
              if [ -n "$INGRESS_HOST" ] && [ "$INGRESS_HOST" != "null" ]; then
                LIVE_URL="http://$INGRESS_HOST"
                echo "Application URL: $LIVE_URL"

                # Update ScrumOps
                curl -s -X POST "${BACKEND_BASE}/api/devops/internal/projects/${PROJECT_NAME}/url" \
                     -H "Content-Type: application/json" \
                     -d "{\"external_url\":\"$LIVE_URL\"}" || true

                if [ -n "${DEPLOYMENT_ID}" ]; then
                  curl -s -X POST "${BACKEND_BASE}/api/devops/internal/deployments/${DEPLOYMENT_ID}/url" \
                       -H "Content-Type: application/json" \
                       -d "{\"external_url\":\"$LIVE_URL\"}" || true
                fi
                break
              fi
              echo "Waiting for ingress to be ready (${i}/30)"
              sleep 10
            done

            echo "Kubernetes resources (namespace: ${NAMESPACE})"
            kubectl get all -n ${NAMESPACE}
          '''

          script {
            if (params.DEPLOYMENT_ID?.trim()) {
              sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/stages" \
                   -H "Content-Type: application/json" \
                   -d '{"deployment_id":"${params.DEPLOYMENT_ID}","stage_name":"health-check","status":"success"}' || true"""
            }
          }
        }
      }
    }
  }

  post {
    always {
      // Cleanup local images to save space (ignore errors)
      sh 'docker rmi ${ACR_SERVER}/${PROJECT_NAME}:${IMAGE_TAG} 2>/dev/null || true'
      sh 'docker rmi ${ACR_SERVER}/${PROJECT_NAME}:latest 2>/dev/null || true'
    }
    success {
      script {
        if (params.DEPLOYMENT_ID?.trim()) {
          sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/deployments/${params.DEPLOYMENT_ID}/status" \
               -H "Content-Type: application/json" \
               -d '{"status":"success"}' || true"""
        }
      }
      echo "Deployment completed successfully. Check ScrumOps dashboard for the live URL."
    }
    failure {
      withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG_FILE')]) {
        sh '''
          export PATH="$PWD:$PATH"
          export KUBECONFIG="${KUBECONFIG_FILE}"
          NS="${NAMESPACE:-unknown-dev}"
          APP="${PROJECT_NAME:-unknown}"
          echo "Describe deployment:"
          kubectl -n "$NS" describe deploy/"$APP" || true
          echo "Recent events:"
          kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 50 || true
          echo "Pods:"
          kubectl -n "$NS" get pods -o wide || true
        '''
      }
      script {
        if (params.DEPLOYMENT_ID?.trim()) {
          sh """curl -s -X POST "${BACKEND_BASE}/api/devops/internal/deployments/${params.DEPLOYMENT_ID}/status" \
               -H "Content-Type: application/json" \
               -d '{"status":"failed"}' || true"""
        }
      }
      echo "Deployment failed. Review Jenkins logs and the ScrumOps dashboard for details."
    }
  }
}
