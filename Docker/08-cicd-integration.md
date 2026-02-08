# CI/CD Security Integration

## Why Security in CI/CD?

**Traditional Approach:**
```
Code → Build → Deploy → (Oops, vulnerability in production)
```

**DevSecOps Approach:**
```
Code → Build → Scan → Test → Deploy
           ↓      ↓      ↓
        Security checks at every stage
```

**Benefits:**
- Find vulnerabilities early
- Automated security checks
- Prevent insecure deployments
- Faster remediation

---

## Security Gates

### Build Stage

**1. Dockerfile Scan:**
```bash
# Scan Dockerfile for misconfigurations
trivy config Dockerfile

# Fail if issues found
trivy config --exit-code 1 Dockerfile
```

**2. Secret Scan:**
```bash
# Scan for hardcoded secrets
trivy fs --security-checks secret .

# Fail if secrets found
trivy fs --security-checks secret --exit-code 1 .
```

**3. Build Image:**
```bash
# Build with security
docker build \
  --no-cache \
  --pull \
  -t myapp:${VERSION} .
```

---

### Test Stage

**1. Image Vulnerability Scan:**
```bash
# Scan built image
trivy image --severity HIGH,CRITICAL myapp:${VERSION}

# Fail pipeline if vulnerabilities
trivy image \
  --exit-code 1 \
  --severity CRITICAL \
  myapp:${VERSION}
```

**2. Generate SBOM:**
```bash
# Create Software Bill of Materials
trivy image \
  --format cyclonedx \
  --output sbom.json \
  myapp:${VERSION}
```

**3. Security Tests:**
```bash
# Test as non-root
docker run --rm myapp:${VERSION} whoami
# Should output: appuser (not root)

# Test read-only works
docker run --rm --read-only myapp:${VERSION}
```

---

### Deploy Stage

**1. Sign Image:**
```bash
# Sign with Docker Content Trust
export DOCKER_CONTENT_TRUST=1
docker push myregistry/myapp:${VERSION}
```

**2. Deploy with Security:**
```bash
# Only deploy if all checks passed
kubectl apply -f secure-deployment.yaml
```

---

## GitLab CI Pipeline

**Complete .gitlab-ci.yml:**

```yaml
stages:
  - build
  - security
  - test
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  DOCKER_DRIVER: overlay2

# Build Stage
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Build image
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
  only:
    - main
    - merge_requests

# Security Stage
dockerfile_scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    # Scan Dockerfile
    - trivy config --exit-code 1 Dockerfile
  allow_failure: false

secret_scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    # Scan for secrets
    - trivy fs --security-checks secret --exit-code 1 .
  allow_failure: false

vulnerability_scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    # Scan image for vulnerabilities
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_NAME
    
    # Generate reports
    - trivy image --format json --output vulnerability-report.json $IMAGE_NAME
    - trivy image --format cyclonedx --output sbom.json $IMAGE_NAME
    - trivy image --format sarif --output trivy-results.sarif $IMAGE_NAME
  artifacts:
    reports:
      sast: trivy-results.sarif
    paths:
      - vulnerability-report.json
      - sbom.json
  allow_failure: false

# Test Stage
security_tests:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  script:
    # Pull image
    - docker pull $IMAGE_NAME
    
    # Test runs as non-root
    - |
      USER=$(docker run --rm $IMAGE_NAME whoami)
      if [ "$USER" == "root" ]; then
        echo "ERROR: Container runs as root!"
        exit 1
      fi
    
    # Test read-only filesystem
    - docker run --rm --read-only $IMAGE_NAME echo "Read-only test passed"
  allow_failure: false

# Deploy Stage
deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    # Update image in Kubernetes
    - kubectl set image deployment/myapp app=$IMAGE_NAME
    
    # Wait for rollout
    - kubectl rollout status deployment/myapp
  only:
    - main
  needs:
    - vulnerability_scan
    - security_tests
```

---

## GitHub Actions Pipeline

**.github/workflows/security.yml:**

```yaml
name: Security Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}:${{ github.sha }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Build image
        run: docker build -t $IMAGE_NAME .
      
      - name: Login to registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Push image
        run: docker push $IMAGE_NAME

  security:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Scan Dockerfile
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: 'Dockerfile'
          exit-code: '1'
      
      - name: Scan for secrets
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          scanners: 'secret'
          exit-code: '1'
      
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Upload results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Generate SBOM
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}
          format: 'cyclonedx'
          output: 'sbom.json'
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: sbom.json

  test:
    runs-on: ubuntu-latest
    needs: security
    steps:
      - name: Pull image
        run: docker pull $IMAGE_NAME
      
      - name: Test non-root user
        run: |
          USER=$(docker run --rm $IMAGE_NAME whoami)
          if [ "$USER" == "root" ]; then
            echo "ERROR: Container runs as root!"
            exit 1
          fi
      
      - name: Test read-only filesystem
        run: docker run --rm --read-only $IMAGE_NAME echo "Success"

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp app=$IMAGE_NAME
          kubectl rollout status deployment/myapp
```

---

## Jenkins Pipeline

**Jenkinsfile:**

```groovy
pipeline {
    agent any
    
    environment {
        IMAGE = "myregistry/myapp:${BUILD_NUMBER}"
    }
    
    stages {
        stage('Build') {
            steps {
                script {
                    docker.build(IMAGE)
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Dockerfile Scan') {
                    steps {
                        sh 'trivy config --exit-code 1 Dockerfile'
                    }
                }
                
                stage('Secret Scan') {
                    steps {
                        sh 'trivy fs --security-checks secret --exit-code 1 .'
                    }
                }
                
                stage('Vulnerability Scan') {
                    steps {
                        sh """
                            trivy image \
                              --exit-code 1 \
                              --severity CRITICAL,HIGH \
                              ${IMAGE}
                        """
                        
                        // Generate reports
                        sh "trivy image --format json -o report.json ${IMAGE}"
                        sh "trivy image --format cyclonedx -o sbom.json ${IMAGE}"
                        
                        archiveArtifacts artifacts: 'report.json,sbom.json'
                    }
                }
            }
        }
        
        stage('Security Tests') {
            steps {
                sh """
                    # Test non-root
                    USER=\$(docker run --rm ${IMAGE} whoami)
                    if [ "\$USER" == "root" ]; then
                        echo "ERROR: Runs as root!"
                        exit 1
                    fi
                    
                    # Test read-only
                    docker run --rm --read-only ${IMAGE}
                """
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    docker push ${IMAGE}
                    kubectl set image deployment/myapp app=${IMAGE}
                    kubectl rollout status deployment/myapp
                """
            }
        }
    }
    
    post {
        failure {
            mail to: 'security@company.com',
                 subject: "Security scan failed: ${env.JOB_NAME}",
                 body: "Build ${env.BUILD_NUMBER} failed security checks"
        }
    }
}
```

---

## Local Development Workflow

**Pre-commit Checks:**

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running security checks..."

# Scan for secrets
echo "Checking for secrets..."
trivy fs --security-checks secret .
if [ $? -ne 0 ]; then
    echo "ERROR: Secrets found!"
    exit 1
fi

# Scan Dockerfile
echo "Checking Dockerfile..."
trivy config Dockerfile
if [ $? -ne 0 ]; then
    echo "ERROR: Dockerfile has issues!"
    exit 1
fi

echo "All checks passed!"
```

**Make executable:**
```bash
chmod +x .git/hooks/pre-commit
```

---

## Security Policy Enforcement

### OPA (Open Policy Agent)

**Policy: Only allow scanned images**

```rego
# policy.rego
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  image := input.request.object.spec.containers[_].image
  not scanned_image(image)
  msg := sprintf("Image %v has not been scanned", [image])
}

scanned_image(image) {
  startswith(image, "registry.company.com/scanned/")
}
```

**Deploy policy:**
```bash
kubectl apply -f opa-policy.yaml
```

---

## Monitoring and Alerts

### Prometheus Metrics

```yaml
# metrics.yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: container-security
spec:
  selector:
    matchLabels:
      app: security-scanner
  endpoints:
  - port: metrics
    interval: 5m
```

### Alerting Rules

```yaml
groups:
  - name: container_security
    rules:
      - alert: CriticalVulnerabilities
        expr: vulnerabilities_critical > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Critical vulnerabilities detected"
          description: "{{ $value }} critical vulnerabilities found"
      
      - alert: ContainerRunningAsRoot
        expr: container_root_user == 1
        labels:
          severity: warning
        annotations:
          summary: "Container running as root"
```

---

## Best Practices

### 1. Fail Fast

```yaml
# Fail pipeline immediately on critical issues
script:
  - trivy image --exit-code 1 --severity CRITICAL $IMAGE
```

### 2. Multiple Scans

```yaml
# Scan at different stages
stages:
  - dockerfile_scan  # Before build
  - image_scan       # After build
  - runtime_scan     # In production
```

### 3. Generate Evidence

```bash
# Keep audit trail
trivy image --format json --output scan-${BUILD_NUMBER}.json $IMAGE

# Store SBOMs
trivy image --format cyclonedx --output sbom-${BUILD_NUMBER}.json $IMAGE
```

### 4. Automate Everything

```yaml
# No manual security checks
# Everything in pipeline
security_scan:
  script:
    - trivy config Dockerfile
    - trivy fs --security-checks secret .
    - trivy image $IMAGE
```

### 5. Notify Security Team

```yaml
# Alert on failures
on_failure:
  - send_notification
  - create_jira_ticket
  - notify_security_team
```

---

## Complete Example: Node.js App

**Dockerfile:**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
RUN adduser -D appuser
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY --from=builder --chown=appuser:appuser /app/dist ./dist
USER appuser
HEALTHCHECK CMD wget -q --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

**.gitlab-ci.yml:**
```yaml
build_and_scan:
  stage: build
  script:
    - docker build -t $IMAGE .
    - trivy config --exit-code 1 Dockerfile
    - trivy fs --security-checks secret --exit-code 1 .
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE
    - docker push $IMAGE
```

---

## Key Takeaways

1. **Shift left** - Security early in pipeline
2. **Automate everything** - No manual checks
3. **Fail fast** - Stop on critical issues
4. **Multiple gates** - Check at every stage
5. **Generate evidence** - SBOMs, reports
6. **Monitor continuously** - Don't stop at deployment
7. **Alert on failures** - Notify security team

**Result:** Secure containers from code to production.
