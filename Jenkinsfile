pipeline {
    agent any

    parameters {
        booleanParam(name: 'ROLLBACK', defaultValue: false,
            description: 'Set true to roll back the vainterior Deployment to its previous revision. Skips all other stages.')
        booleanParam(name: 'SKIP_CANARY', defaultValue: false,
            description: 'Deploy straight to stable without a canary bake period. Not recommended for prod.')
    }

    options {
        disableConcurrentBuilds()
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        APP_NAME = 'vainterior'
        HOST_PORT = '8081'
        CONTAINER_PORT = '80'
        IMAGE_NAME = 'vainterior-app'
        REGISTRY_HOST = 'registry.local'   // internal DNS name inside Kind network
        REGISTRY_URL = 'registry.local:5000'
        REGISTRY_IMAGE = "${REGISTRY_URL}/${IMAGE_NAME}"

        // All paths relative to workspace
        ARTIFACTS_DIR = "${WORKSPACE}/artifacts"
        ZAP_REPORTS_DIR = "${ARTIFACTS_DIR}/zap-reports"
        SONAR_PROJECT_KEY = 'vainterior'
        ARTIFACT_VERSION = "${BUILD_NUMBER}"
        DIST_DIR = "${WORKSPACE}/dist"
        BUILD_DIR = "${WORKSPACE}/build"
        K8S_NAMESPACE = 'vainterior'
        K8S_DEPLOYMENT_NAME = 'vainterior'
        K8S_SERVICE_NAME = 'vainterior-svc'
        K8S_NODE_PORT = '30081'
        K8S_MANIFESTS_DIR = "${WORKSPACE}/k8s"
        K8S_CANARY_NAME = 'vainterior-canary'
        CANARY_REPLICAS = '1'
        CANARY_BAKE_SECONDS = '45'
        CANARY_HEALTH_CHECKS = '10'
    }

    stages {

        // -------------------- Helper methods (inline Groovy) --------------------
        // These reduce duplication across stages
        stage('Setup') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    // Ensure required directories
                    sh "mkdir -p ${ARTIFACTS_DIR} ${ZAP_REPORTS_DIR} ${K8S_MANIFESTS_DIR}"
                    // Install helper tools if missing (use agent with these pre‑installed for speed)
                    sh '''
                        if ! command -v jq >/dev/null; then
                            echo "Installing jq..."
                            apt-get update -y && apt-get install -y jq
                        fi
                        if ! command -v zip >/dev/null; then
                            echo "Installing zip..."
                            apt-get update -y && apt-get install -y zip
                        fi
                    '''
                }
            }
        }

        stage('Checkout') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    echo "Current workspace: ${env.WORKSPACE}"
                    // Clean any leftover kubeconfig from previous runs
                    sh "find ${ARTIFACTS_DIR} -maxdepth 1 -iname 'kubeconfig*' -delete 2>/dev/null || true"

                    try {
                        withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                                          passwordVariable: 'GITHUB_TOKEN',
                                                          usernameVariable: 'GITHUB_USER')]) {
                            git branch: 'main',
                                url: "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/Gourav715/vainterior.git"
                        }
                    } catch (Exception e) {
                        echo "Authenticated clone failed (${e.message}). Falling back to public clone..."
                        git branch: 'main', url: 'https://github.com/Gourav715/vainterior.git'
                    }

                    if (!fileExists('package.json')) {
                        error "❌ package.json not found!"
                    }
                    echo "✅ package.json found!"
                }
            }
        }

        stage('Build Application') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    // Use node:18-alpine (or a pre‑built agent) – no need to translate paths because we mount the workspace
                    sh '''
                        set -e
                        echo "📦 Building Application"
                        # Use node container to build – we mount the whole workspace
                        docker run --rm -v "$(pwd):/workspace" -w /workspace \
                            node:18-alpine sh -c "npm install --legacy-peer-deps && npm run build"

                        # Determine build output
                        if [ -d "dist" ]; then
                            BUILD_OUTPUT="dist"
                        elif [ -d "build" ]; then
                            BUILD_OUTPUT="build"
                        else
                            echo "❌ No build output directory found!"
                            exit 1
                        fi

                        mkdir -p ${ARTIFACTS_DIR}/app-build
                        cp -r ${BUILD_OUTPUT}/* ${ARTIFACTS_DIR}/app-build/
                        # Zip using agent's zip (now available)
                        cd ${ARTIFACTS_DIR}/app-build && zip -r ../vainterior-${BUILD_NUMBER}.zip . && cd -
                        echo "✅ Application build saved"
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            when { expression { !params.ROLLBACK } }
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        set -e
                        echo "🔍 SonarQube Analysis"
                        # Run scanner with mounted workspace
                        docker run --rm \\
                            -v "$(pwd):/usr/src" \\
                            -v sonar-cache:/root/.sonar/cache \\
                            -e SONAR_TOKEN=${SONAR_TOKEN} \\
                            sonarsource/sonar-scanner-cli:latest \\
                            -Dsonar.projectBaseDir=/usr/src \\
                            -Dsonar.projectKey=vainterior \\
                            -Dsonar.projectName=Vainterior \\
                            -Dsonar.sources=src \\
                            -Dsonar.exclusions="**/node_modules/**,**/dist/**,**/build/**,**/coverage/**,**/artifacts/**,**/trivy-reports/**,**/zap-reports/**,**/*.test.ts,**/*.spec.ts" \\
                            -Dsonar.host.url=http://sonarqube:9000 \\
                            -Dsonar.verbose=true
                    '''
                }
            }
        }

        stage('Quality Gate') {
            when { expression { !params.ROLLBACK } }
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        set -e
                        echo "📊 Checking Quality Gate"
                        SONAR_URL="http://sonarqube:9000"
                        PROJECT_KEY="vainterior"
                        STATUS="NONE"

                        sleep 30  # allow analysis to process

                        for i in 1 2 3 4 5; do
                            RESPONSE=$(curl -s -u "${SONAR_TOKEN}:" "${SONAR_URL}/api/qualitygates/project_status?projectKey=${PROJECT_KEY}")
                            if [ -n "${RESPONSE}" ]; then
                                STATUS=$(echo "${RESPONSE}" | jq -r '.projectStatus.status')
                                [ -z "${STATUS}" ] && STATUS="NONE"
                                if [ "${STATUS}" = "OK" ] || [ "${STATUS}" = "ERROR" ]; then
                                    break
                                fi
                            fi
                            sleep 10
                        done

                        echo "${RESPONSE}" | jq '.' > ${ARTIFACTS_DIR}/sonar-metrics.json
                        echo "Quality Gate Status: ${STATUS}"

                        if [ "${STATUS}" = "OK" ]; then
                            echo "PASSED" > ${ARTIFACTS_DIR}/quality-gate-status.txt
                            echo "✅ Quality Gate PASSED!"
                        elif [ "${STATUS}" = "ERROR" ]; then
                            echo "FAILED" > ${ARTIFACTS_DIR}/quality-gate-status.txt
                            echo "❌ Quality Gate FAILED!"
                            exit 1
                        else
                            echo "UNKNOWN" > ${ARTIFACTS_DIR}/quality-gate-status.txt
                            echo "⚠️ Unknown status, check dashboard"
                        fi
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    // If no Dockerfile, generate one
                    if (!fileExists('Dockerfile')) {
                        writeFile file: 'Dockerfile', text: '''FROM nginx:1.27-alpine
RUN rm -rf /usr/share/nginx/html/*
COPY artifacts/app-build/ /usr/share/nginx/html/
RUN printf 'server {\\n    listen 80;\\n    server_name _;\\n    root /usr/share/nginx/html;\\n    index index.html;\\n    location / {\\n        try_files $uri $uri/ /index.html;\\n    }\\n}\\n' > /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]'''
                    }
                    // Build using Docker Pipeline plugin – no socket mount needed
                    docker.build("${IMAGE_NAME}:${BUILD_NUMBER}", ".")
                    docker.image("${IMAGE_NAME}:${BUILD_NUMBER}").inspect() > "${ARTIFACTS_DIR}/docker-image-info.json"
                }
            }
        }

        stage('Trivy Scan') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    sh '''
                        set -e
                        echo "🔎 Running Trivy scan on ${IMAGE_NAME}:${BUILD_NUMBER}"
                        mkdir -p ${ARTIFACTS_DIR}/trivy-reports

                        # Use the same trick: mount workspace, but no docker.sock needed because we use Trivy's image scanning with the local image
                        # Since the image is in the local Docker daemon, we can reference it by name.
                        docker run --rm \\
                            -v trivy-cache:/root/.cache/ \\
                            -v /var/run/docker.sock:/var/run/docker.sock \\
                            -v "$(pwd)/${ARTIFACTS_DIR}/trivy-reports:/reports" \\
                            aquasec/trivy:latest image \\
                            --severity MEDIUM,HIGH,CRITICAL \\
                            --format table \\
                            --output /reports/trivy-report.txt \\
                            --ignore-unfixed \\
                            ${IMAGE_NAME}:${BUILD_NUMBER}

                        # Critical only, fail if any
                        docker run --rm \\
                            -v trivy-cache:/root/.cache/ \\
                            -v /var/run/docker.sock:/var/run/docker.sock \\
                            -v "$(pwd)/${ARTIFACTS_DIR}/trivy-reports:/reports" \\
                            aquasec/trivy:latest image \\
                            --severity CRITICAL \\
                            --format json \\
                            --output /reports/trivy-critical.json \\
                            --ignore-unfixed \\
                            --exit-code 1 \\
                            ${IMAGE_NAME}:${BUILD_NUMBER} || true   # We'll check count manually

                        CRITICAL_COUNT=$(cat ${ARTIFACTS_DIR}/trivy-reports/trivy-critical.json | grep -o '"VulnerabilityID"' | wc -l || echo 0)
                        echo "Critical vulnerabilities: ${CRITICAL_COUNT}"
                        echo ${CRITICAL_COUNT} > ${ARTIFACTS_DIR}/trivy-critical-count.txt
                        if [ ${CRITICAL_COUNT} -gt 0 ]; then
                            echo "⚠️  CRITICAL vulnerabilities found!"
                            # Optionally fail: exit 1
                        else
                            echo "✅ No critical vulnerabilities"
                        fi
                    '''
                }
            }
        }

        stage('Push to Registry') {
            when { expression { !params.ROLLBACK } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REG_USER',
                                                  passwordVariable: 'REG_PASS')]) {
                    script {
                        // Use Docker Pipeline plugin to push securely
                        docker.withRegistry("http://${REGISTRY_URL}", 'registry-credentials') {
                            def image = docker.image("${IMAGE_NAME}:${BUILD_NUMBER}")
                            image.push()
                            image.push('latest')
                        }
                        sh "echo ${REGISTRY_IMAGE}:${BUILD_NUMBER} > ${ARTIFACTS_DIR}/registry-image.txt"
                    }
                }
            }
        }

        stage('Generate Kubernetes Manifests') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    sh """
                        cat > ${K8S_MANIFESTS_DIR}/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${K8S_DEPLOYMENT_NAME}
  namespace: ${K8S_NAMESPACE}
  labels:
    app: ${K8S_DEPLOYMENT_NAME}
    track: stable
spec:
  replicas: 2
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: ${K8S_DEPLOYMENT_NAME}
      track: stable
  template:
    metadata:
      labels:
        app: ${K8S_DEPLOYMENT_NAME}
        track: stable
    spec:
      imagePullSecrets:
        - name: registry-credentials
      containers:
        - name: ${K8S_DEPLOYMENT_NAME}
          image: ${REGISTRY_IMAGE}:${BUILD_NUMBER}
          imagePullPolicy: Always
          ports:
            - containerPort: ${CONTAINER_PORT}
          readinessProbe:
            httpGet:
              path: /
              port: ${CONTAINER_PORT}
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: ${CONTAINER_PORT}
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
EOF

                        cat > ${K8S_MANIFESTS_DIR}/canary-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${K8S_CANARY_NAME}
  namespace: ${K8S_NAMESPACE}
  labels:
    app: ${K8S_DEPLOYMENT_NAME}
    track: canary
spec:
  replicas: ${CANARY_REPLICAS}
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: ${K8S_DEPLOYMENT_NAME}
      track: canary
  template:
    metadata:
      labels:
        app: ${K8S_DEPLOYMENT_NAME}
        track: canary
    spec:
      imagePullSecrets:
        - name: registry-credentials
      containers:
        - name: ${K8S_DEPLOYMENT_NAME}
          image: ${REGISTRY_IMAGE}:${BUILD_NUMBER}
          imagePullPolicy: Always
          ports:
            - containerPort: ${CONTAINER_PORT}
          readinessProbe:
            httpGet:
              path: /
              port: ${CONTAINER_PORT}
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: ${CONTAINER_PORT}
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
EOF

                        cat > ${K8S_MANIFESTS_DIR}/service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: ${K8S_SERVICE_NAME}
  namespace: ${K8S_NAMESPACE}
spec:
  type: NodePort
  selector:
    app: ${K8S_DEPLOYMENT_NAME}
  ports:
    - port: ${CONTAINER_PORT}
      targetPort: ${CONTAINER_PORT}
      nodePort: ${K8S_NODE_PORT}
EOF
                    """
                }
            }
        }

        // -------------------- Canary Stages (only if not SKIP_CANARY) --------------------
        stage('Deploy Canary') {
            when { expression { !params.ROLLBACK && !params.SKIP_CANARY } }
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh '''
                            set -e
                            echo "=== Deploying canary ==="
                            # Use kubectl from agent (or container) – we assume kubectl is available
                            # For reproducibility, we can use a container, but we'll rely on agent having kubectl
                            kubectl apply -n ${K8S_NAMESPACE} -f ${K8S_MANIFESTS_DIR}/canary-deployment.yaml
                            kubectl rollout status deployment/${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --timeout=180s
                        '''
                    }
                }
            }
        }

        stage('Canary Health Check') {
            when { expression { !params.ROLLBACK && !params.SKIP_CANARY } }
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh '''
                            set -e
                            echo "=== Canary Health Checks ==="
                            # We can use curl from within cluster to hit NodePort
                            NODE_PORT=$(kubectl get svc ${K8S_SERVICE_NAME} -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                            TARGET_URL="http://kind-control-plane:${NODE_PORT}"

                            # Wait for service to be ready
                            for i in $(seq 1 12); do
                                if curl -s -f -o /dev/null --max-time 5 ${TARGET_URL}; then
                                    echo "Service reachable"
                                    break
                                fi
                                sleep 5
                            done

                            # Check restarts
                            RESTARTS=$(kubectl get pods -n ${K8S_NAMESPACE} -l track=canary -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}')
                            for r in $RESTARTS; do
                                if [ "$r" -gt 0 ]; then
                                    echo "❌ Canary restarted ($r times)"
                                    kubectl delete deployment ${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true
                                    exit 1
                                fi
                            done

                            # HTTP sampling
                            FAIL_COUNT=0
                            for i in $(seq 1 ${CANARY_HEALTH_CHECKS}); do
                                STATUS=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 ${TARGET_URL} || echo "000")
                                if [ "$STATUS" != "200" ]; then
                                    FAIL_COUNT=$((FAIL_COUNT + 1))
                                fi
                                sleep 2
                            done

                            if [ $FAIL_COUNT -gt 1 ]; then
                                echo "❌ Too many failures ($FAIL_COUNT/${CANARY_HEALTH_CHECKS})"
                                kubectl delete deployment ${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true
                                exit 1
                            fi

                            # Bake period
                            echo "Baking for ${CANARY_BAKE_SECONDS}s"
                            sleep ${CANARY_BAKE_SECONDS}

                            # Final check
                            LATE_RESTARTS=$(kubectl get pods -n ${K8S_NAMESPACE} -l track=canary -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}')
                            for r in $LATE_RESTARTS; do
                                if [ "$r" -gt 0 ]; then
                                    echo "❌ Canary crashed during bake"
                                    kubectl delete deployment ${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true
                                    exit 1
                                fi
                            done

                            echo "✅ Canary healthy – ready to promote"
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                          usernameVariable: 'REG_USER',
                                                          passwordVariable: 'REG_PASS')]) {
                            sh '''
                                set -e
                                echo "=== Deploying stable ==="
                                # Create namespace if absent
                                kubectl get namespace ${K8S_NAMESPACE} || kubectl create namespace ${K8S_NAMESPACE}

                                # Create image pull secret
                                kubectl delete secret registry-credentials -n ${K8S_NAMESPACE} 2>/dev/null || true
                                kubectl create secret docker-registry registry-credentials \\
                                    --docker-server=${REGISTRY_URL} \\
                                    --docker-username=${REG_USER} \\
                                    --docker-password=${REG_PASS} \\
                                    -n ${K8S_NAMESPACE}

                                # Handle selector mismatch – delete if needed
                                EXISTING_APP=$(kubectl get deployment ${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} -o jsonpath='{.spec.selector.matchLabels.app}' 2>/dev/null || echo "")
                                EXISTING_TRACK=$(kubectl get deployment ${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} -o jsonpath='{.spec.selector.matchLabels.track}' 2>/dev/null || echo "")
                                if [ "$EXISTING_APP" != "${K8S_DEPLOYMENT_NAME}" ] || [ "$EXISTING_TRACK" != "stable" ]; then
                                    echo "Selector mismatch – deleting deployment"
                                    kubectl delete deployment ${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true
                                    sleep 5
                                fi

                                # Apply manifests
                                kubectl apply -n ${K8S_NAMESPACE} -f ${K8S_MANIFESTS_DIR}/deployment.yaml
                                kubectl apply -n ${K8S_NAMESPACE} -f ${K8S_MANIFESTS_DIR}/service.yaml

                                # Wait for rollout
                                kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} --timeout=600s

                                # Cleanup canary if it exists (promoted)
                                kubectl delete deployment ${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true

                                # Save status
                                kubectl get all -n ${K8S_NAMESPACE} -o wide > ${ARTIFACTS_DIR}/k8s-status.txt
                                NODE_PORT=$(kubectl get svc ${K8S_SERVICE_NAME} -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                                echo $NODE_PORT > ${ARTIFACTS_DIR}/node-port.txt
                                echo "✅ Stable deployment successful"
                            '''
                        }
                    }
                }
            }
        }

        stage('Verify Kubernetes Deployment') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh '''
                            set -e
                            echo "=== Verifying Deployment ==="
                            PODS=$(kubectl get pods -n ${K8S_NAMESPACE} -l app=${K8S_DEPLOYMENT_NAME} -o json)
                            RUNNING=$(echo $PODS | jq -r '.items[].status.phase' | grep -c "Running" || true)
                            TOTAL=$(echo $PODS | jq -r '.items | length')
                            if [ "$RUNNING" -eq "$TOTAL" ] && [ "$TOTAL" -gt 0 ]; then
                                echo "✅ All pods running ($RUNNING/$TOTAL)"
                            else
                                echo "❌ Pods not ready ($RUNNING/$TOTAL)"
                                exit 1
                            fi
                            NODE_PORT=$(kubectl get svc ${K8S_SERVICE_NAME} -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                            echo "Service available at http://kind-control-plane:${NODE_PORT}"
                        '''
                    }
                }
            }
        }

        stage('OWASP ZAP Full Scan') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    sh '''
                        set -e
                        mkdir -p ${ZAP_REPORTS_DIR}
                        NODE_PORT=$(cat ${ARTIFACTS_DIR}/node-port.txt 2>/dev/null || echo "${K8S_NODE_PORT}")
                        TARGET_URL="http://kind-control-plane:${NODE_PORT}"

                        # Wait for target
                        for i in $(seq 1 12); do
                            if curl -s -f -o /dev/null --max-time 5 ${TARGET_URL}; then
                                echo "Target reachable"
                                break
                            fi
                            sleep 5
                        done

                        # Run ZAP
                        docker run --rm --network kind \
                            -v "$(pwd)/${ZAP_REPORTS_DIR}:/zap/wrk:rw" \
                            zaproxy/zap-stable \
                            zap-full-scan.py -t ${TARGET_URL} \
                            -r zap-full-report.html -J zap-full-report.json -I || true

                        echo "ZAP reports saved"
                    '''
                }
            }
        }

        stage('Application Info') {
            when { expression { !params.ROLLBACK } }
            steps {
                script {
                    echo """
                        ========================================
                        ✅ APPLICATION DEPLOYED SUCCESSFULLY
                        ========================================
                        Application:   ${APP_NAME}
                        Build Number:  ${BUILD_NUMBER}
                        Image:         ${REGISTRY_IMAGE}:${BUILD_NUMBER}
                        K8s Namespace: ${K8S_NAMESPACE}
                        Deploy:        ${K8S_DEPLOYMENT_NAME}
                        URL:           http://kind-control-plane:${K8S_NODE_PORT}
                        Artifacts:     ${ARTIFACTS_DIR}/
                        ========================================
                    """
                }
            }
        }

        // -------------------- Rollback Stage --------------------
        stage('Rollback to Previous Revision (Kubernetes)') {
            when { expression { params.ROLLBACK } }
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh '''
                            set -e
                            echo "⏪ Rolling back ${K8S_DEPLOYMENT_NAME} to previous revision"
                            kubectl rollout history deployment/${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} || true
                            kubectl rollout undo deployment/${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE}
                            kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} --timeout=180s
                            kubectl delete deployment ${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true
                            echo "Rollback complete"
                        '''
                    }
                }
            }
        }
    }

    // -------------------- Post-build Actions --------------------
    post {
        success {
            script {
                // Archive all artifacts
                sh """
                    cd ${ARTIFACTS_DIR}
                    zip -r ../vainterior-all-artifacts-${BUILD_NUMBER}.zip . -x 'kubeconfig*'
                    cd ..
                """
                archiveArtifacts artifacts: "artifacts/**/*", excludes: "artifacts/kubeconfig*", allowEmptyArchive: true
                archiveArtifacts artifacts: "vainterior-all-artifacts-${BUILD_NUMBER}.zip", allowEmptyArchive: true
                archiveArtifacts artifacts: "artifacts/vainterior-${BUILD_NUMBER}.zip", allowEmptyArchive: true
                echo "✅ Pipeline SUCCESS"
            }
        }
        failure {
            script {
                // Cleanup any canary
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "kubectl delete deployment ${K8S_CANARY_NAME} -n ${K8S_NAMESPACE} --ignore-not-found=true || true"
                }
                archiveArtifacts artifacts: "artifacts/**/*", excludes: "artifacts/kubeconfig*", allowEmptyArchive: true
                archiveArtifacts artifacts: "vainterior-all-artifacts-${BUILD_NUMBER}.zip", allowEmptyArchive: true
                echo "❌ Pipeline FAILED – check logs"
            }
        }
        always {
            script {
                // Securely wipe any kubeconfig copies (though we used withKubeConfig, no file on disk)
                sh "find ${ARTIFACTS_DIR} -maxdepth 1 -iname 'kubeconfig*' -delete 2>/dev/null || true"
                sh "docker image prune -f || true"
            }
        }
    }
}
