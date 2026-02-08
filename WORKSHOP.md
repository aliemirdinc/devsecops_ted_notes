# DevSecOps Hands-On Workshop

## Workshop Overview
This workshop provides practical exercises to test your DevSecOps knowledge. Each lab simulates real-world security scenarios.

---

## Lab Setup

### Prerequisites
```bash
# Install Docker
sudo apt-get update
sudo apt-get install docker.io

# Install Trivy
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Verify installations
docker --version
trivy --version
```

### Workshop Environment Setup
```bash
# Create workshop directory
mkdir -p ~/devsecops-workshop
cd ~/devsecops-workshop

# Create subdirectories for each lab
mkdir -p lab1-image-scanning
mkdir -p lab2-secret-detection
mkdir -p lab3-iac-security
mkdir -p lab4-sbom-generation
mkdir -p lab5-cicd-integration
mkdir -p lab6-vulnerability-remediation
```

---

## Lab 1: Container Image Vulnerability Scanning

### Objective
Identify and fix vulnerabilities in container images.

### Scenario
Your team uses an older nginx image in production. Scan it for vulnerabilities and remediate.

### Tasks

**Task 1: Scan vulnerable image**
```bash
cd ~/devsecops-workshop/lab1-image-scanning

# Pull and scan nginx:1.19
docker pull nginx:1.19
trivy image nginx:1.19
```

**Questions:**
- How many CRITICAL vulnerabilities are found?
- How many HIGH vulnerabilities are found?
- What is the most severe CVE?

**Task 2: Filter by severity**
```bash
# Show only CRITICAL
trivy image --severity CRITICAL nginx:1.19

# Show CRITICAL and HIGH
trivy image --severity CRITICAL,HIGH nginx:1.19
```

**Task 3: Generate JSON report**
```bash
trivy image --format json --output nginx-1.19-report.json nginx:1.19

# View report
cat nginx-1.19-report.json | jq '.Results[].Vulnerabilities[] | select(.Severity=="CRITICAL") | {VulnerabilityID, PkgName, InstalledVersion, FixedVersion}'
```

**Task 4: Find safer version**
```bash
# Test different versions
trivy image --severity CRITICAL nginx:1.20
trivy image --severity CRITICAL nginx:1.21
trivy image --severity CRITICAL nginx:latest
```

**Task 5: Ignore unfixed vulnerabilities**
```bash
trivy image --ignore-unfixed --severity CRITICAL,HIGH nginx:1.19
```

**Expected Outcome:**
- Understand vulnerability severity levels
- Know how to generate reports
- Learn version comparison for remediation

---

## Lab 2: Secret Detection in Code

### Objective
Detect hardcoded secrets before they reach production.

### Tasks

**Task 1: Create vulnerable files**
```bash
cd ~/devsecops-workshop/lab2-secret-detection

# Create config file with secrets
cat > config.yaml <<EOF
database:
  host: localhost
  username: admin
  password: SuperSecret123!
  
aws:
  access_key: AKIA_EXAMPLE_KEY_ID_HERE
  secret_key: wJalrXUtnFEMI_EXAMPLE_SECRET_KEY
  
github_token: ghp_EXAMPLE1234567890abcdefghij

stripe_key: sk_test_EXAMPLE_4eC39HqLyjWDar
EOF

# Create .env file
cat > .env <<EOF
DB_PASSWORD=password123
API_KEY=1234567890abcdef
JWT_SECRET=my-secret-jwt-key
STRIPE_SECRET=sk_live_EXAMPLE_real_key_here
EOF

# Create Python script with secrets
cat > app.py <<EOF
import requests

# Bad practice: Hardcoded credentials
AWS_ACCESS_KEY = "AKIA_EXAMPLE_KEY_ID"
AWS_SECRET_KEY = "wJalrXUtn_EXAMPLE_SECRET"

# Bad practice: API keys in code
OPENAI_API_KEY = "sk-proj_EXAMPLE1234567890abcdef"

def connect_database():
    password = "admin123"  # Hardcoded password
    return f"postgresql://user:{password}@localhost/db"
EOF
```

**Task 2: Scan for secrets**
```bash
# Scan current directory
trivy fs --security-checks secret .

# Scan specific file
trivy fs --security-checks secret config.yaml
trivy fs --security-checks secret .env
trivy fs --security-checks secret app.py
```

**Questions:**
- How many secrets were detected?
- What types of secrets were found?
- What files contain secrets?

**Task 3: Review findings**
```bash
# Generate detailed report
trivy fs --security-checks secret --format json --output secrets-report.json .

# View specific secret types
cat secrets-report.json | jq '.Results[].Secrets[] | {RuleID, Category, Severity, Title, Match}'
```

**Task 4: Remediation**

Create secure versions:
```bash
# Create .gitignore
cat > .gitignore <<EOF
.env
config.yaml
secrets/
*.key
*.pem
EOF

# Create config template
cat > config.yaml.template <<EOF
database:
  host: \${DB_HOST}
  username: \${DB_USERNAME}
  password: \${DB_PASSWORD}
  
aws:
  access_key: \${AWS_ACCESS_KEY}
  secret_key: \${AWS_SECRET_KEY}
EOF

# Rewrite app.py securely
cat > app_secure.py <<EOF
import os
import requests

# Good practice: Use environment variables
AWS_ACCESS_KEY = os.getenv("AWS_ACCESS_KEY")
AWS_SECRET_KEY = os.getenv("AWS_SECRET_KEY")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")

def connect_database():
    password = os.getenv("DB_PASSWORD")
    return f"postgresql://user:{password}@localhost/db"
EOF
```

**Task 5: Verify remediation**
```bash
# Scan again
trivy fs --security-checks secret app_secure.py
trivy fs --security-checks secret config.yaml.template
```

**Expected Outcome:**
- Detect various secret types
- Understand remediation strategies
- Implement secure configuration management

---

## Lab 3: Infrastructure as Code Security

### Objective
Identify misconfigurations in Kubernetes and Docker files.

### Tasks

**Task 1: Create vulnerable Dockerfile**
```bash
cd ~/devsecops-workshop/lab3-iac-security

cat > Dockerfile.vulnerable <<EOF
FROM ubuntu:18.04

# Running as root
USER root

# Installing packages without cleanup
RUN apt-get update && apt-get install -y curl wget

# Exposing unnecessary ports
EXPOSE 22 3306 6379

# Hardcoded secrets
ENV DB_PASSWORD=secret123
ENV API_KEY=1234567890abcdef

# Using ADD instead of COPY
ADD https://example.com/script.sh /tmp/script.sh

# No healthcheck
COPY app.sh /app.sh
CMD ["/app.sh"]
EOF
```

**Task 2: Scan Dockerfile**
```bash
trivy config Dockerfile.vulnerable
```

**Questions:**
- What misconfigurations were found?
- What are the severity levels?
- What are the security implications?

**Task 3: Create secure Dockerfile**
```bash
cat > Dockerfile.secure <<EOF
FROM ubuntu:22.04

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install packages with cleanup
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Only expose necessary ports
EXPOSE 8080

# Use build args for configuration
ARG APP_VERSION=1.0.0

# Use COPY instead of ADD
COPY --chown=appuser:appuser app.sh /app.sh

# Switch to non-root user
USER appuser

# Add healthcheck
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["/app.sh"]
EOF
```

**Task 4: Compare scans**
```bash
# Scan vulnerable version
trivy config Dockerfile.vulnerable --format json --output dockerfile-vulnerable.json

# Scan secure version
trivy config Dockerfile.secure --format json --output dockerfile-secure.json

# Compare
echo "=== Vulnerable Dockerfile Issues ==="
trivy config Dockerfile.vulnerable

echo "=== Secure Dockerfile Issues ==="
trivy config Dockerfile.secure
```

**Task 5: Kubernetes misconfiguration**
```bash
# Create vulnerable deployment
cat > deployment.vulnerable.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        securityContext:
          privileged: true
          runAsUser: 0
          allowPrivilegeEscalation: true
        env:
        - name: DB_PASSWORD
          value: "hardcoded-password"
EOF

# Scan deployment
trivy config deployment.vulnerable.yaml
```

**Task 6: Create secure deployment**
```bash
cat > deployment.secure.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: webapp
        image: myapp:v1.2.3
        ports:
        - containerPort: 8080
        securityContext:
          privileged: false
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: db-password
EOF

# Scan secure deployment
trivy config deployment.secure.yaml
```

**Expected Outcome:**
- Identify Dockerfile best practices
- Understand Kubernetes security contexts
- Learn secure IaC configuration

---

## Lab 4: SBOM Generation and Analysis

### Objective
Generate and analyze Software Bill of Materials.

### Tasks

**Task 1: Generate SBOM for images**
```bash
cd ~/devsecops-workshop/lab4-sbom-generation

# Pull test images
docker pull nginx:1.21
docker pull python:3.9-slim
docker pull node:16-alpine

# Generate SBOMs
trivy image --format cyclonedx --output nginx-sbom.json nginx:1.21
trivy image --format cyclonedx --output python-sbom.json python:3.9-slim
trivy image --format cyclonedx --output node-sbom.json node:16-alpine
```

**Task 2: Analyze SBOM contents**
```bash
# View component count
echo "=== Nginx Components ==="
cat nginx-sbom.json | jq '.components | length'

echo "=== Python Components ==="
cat python-sbom.json | jq '.components | length'

echo "=== Node Components ==="
cat node-sbom.json | jq '.components | length'

# List all components
cat nginx-sbom.json | jq '.components[] | {name: .name, version: .version}'
```

**Task 3: Search for specific package**
```bash
# Search for openssl in all SBOMs
echo "=== Searching for OpenSSL ==="
cat nginx-sbom.json | jq '.components[] | select(.name | contains("openssl"))'
cat python-sbom.json | jq '.components[] | select(.name | contains("openssl"))'
cat node-sbom.json | jq '.components[] | select(.name | contains("openssl"))'
```

**Task 4: Scan SBOM for vulnerabilities**
```bash
# Scan SBOMs
trivy sbom nginx-sbom.json --severity HIGH,CRITICAL
trivy sbom python-sbom.json --severity HIGH,CRITICAL
trivy sbom node-sbom.json --severity HIGH,CRITICAL
```

**Task 5: Simulate vulnerability response**
```bash
# Create script to check all SBOMs for specific vulnerability
cat > check-vulnerability.sh <<'EOF'
#!/bin/bash

PACKAGE_NAME=$1
echo "Searching for package: $PACKAGE_NAME"
echo "=================================="

for sbom in *.json; do
  echo "Checking: $sbom"
  found=$(cat $sbom | jq -r ".components[] | select(.name | contains(\"$PACKAGE_NAME\")) | .name + \" \" + .version")
  if [ ! -z "$found" ]; then
    echo "FOUND: $found"
  else
    echo "Not found"
  fi
  echo "---"
done
EOF

chmod +x check-vulnerability.sh

# Test the script
./check-vulnerability.sh openssl
./check-vulnerability.sh curl
```

**Expected Outcome:**
- Generate SBOM in different formats
- Analyze SBOM contents
- Use SBOM for vulnerability tracking

---

## Lab 5: CI/CD Integration

### Objective
Integrate Trivy into CI/CD pipeline with automated security gates.

### Tasks

**Task 1: Create sample application**
```bash
cd ~/devsecops-workshop/lab5-cicd-integration

# Create simple Python app
cat > app.py <<EOF
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, DevSecOps!"

@app.route('/health')
def health():
    return {"status": "healthy"}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF

# Create requirements
cat > requirements.txt <<EOF
Flask==2.0.1
requests==2.25.1
EOF

# Create Dockerfile
cat > Dockerfile <<EOF
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

USER 1000

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["python", "app.py"]
EOF
```

**Task 2: Create GitLab CI pipeline**
```bash
cat > .gitlab-ci.yml <<EOF
stages:
  - build
  - security
  - deploy

variables:
  IMAGE_NAME: \$CI_REGISTRY_IMAGE:\$CI_COMMIT_SHA
  
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t \$IMAGE_NAME .
    - docker push \$IMAGE_NAME
  only:
    - main
    - merge_requests

trivy_scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    # Scan for vulnerabilities
    - trivy image --severity HIGH,CRITICAL --exit-code 1 \$IMAGE_NAME
    
    # Scan for secrets
    - trivy fs --security-checks secret --exit-code 1 .
    
    # Scan IaC
    - trivy config --exit-code 1 .
    
    # Generate reports
    - trivy image --format json --output vulnerability-report.json \$IMAGE_NAME
    - trivy image --format sarif --output trivy-results.sarif \$IMAGE_NAME
  artifacts:
    reports:
      sast: trivy-results.sarif
    paths:
      - vulnerability-report.json
  allow_failure: false
  only:
    - main
    - merge_requests

deploy:
  stage: deploy
  script:
    - echo "Deploying \$IMAGE_NAME"
    # Deployment commands here
  only:
    - main
  needs:
    - trivy_scan
EOF
```

**Task 3: Create GitHub Actions workflow**
```bash
mkdir -p .github/workflows

cat > .github/workflows/security-scan.yml <<EOF
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Build Docker image
      run: docker build -t myapp:\${{ github.sha }} .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:\${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Scan for secrets
      run: |
        docker run --rm -v \$PWD:/workspace aquasec/trivy:latest \
          fs --security-checks secret --exit-code 1 /workspace
    
    - name: Scan IaC
      run: |
        docker run --rm -v \$PWD:/workspace aquasec/trivy:latest \
          config --exit-code 1 /workspace
EOF
```

**Task 4: Local pipeline simulation**
```bash
# Simulate CI/CD locally
cat > run-pipeline.sh <<'EOF'
#!/bin/bash

set -e

echo "=== Stage 1: Build ==="
docker build -t myapp:test .

echo "=== Stage 2: Security Scan ==="
echo "--- Vulnerability Scan ---"
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:test

echo "--- Secret Scan ---"
trivy fs --security-checks secret --exit-code 1 .

echo "--- IaC Scan ---"
trivy config --exit-code 1 .

echo "=== Stage 3: Generate Reports ==="
trivy image --format json --output reports/vulnerability-report.json myapp:test
trivy image --format sarif --output reports/trivy-results.sarif myapp:test

echo "=== Pipeline Complete ==="
echo "All security checks passed!"
EOF

chmod +x run-pipeline.sh
mkdir -p reports

# Run the pipeline
./run-pipeline.sh
```

**Task 5: Test security gate failure**
```bash
# Introduce a vulnerability
cat > requirements.txt <<EOF
Flask==2.0.1
requests==2.20.0
EOF

# Rebuild and scan (should fail)
docker build -t myapp:vulnerable .
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:vulnerable

# Fix it
cat > requirements.txt <<EOF
Flask==2.3.0
requests==2.31.0
EOF

docker build -t myapp:fixed .
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:fixed
```

**Expected Outcome:**
- Create automated security pipelines
- Implement security gates
- Generate security reports
- Handle failures appropriately

---

## Lab 6: Vulnerability Remediation Workflow

### Objective
Practice complete vulnerability identification and remediation cycle.

### Scenario
You inherit a legacy application with multiple security issues. Your task is to identify and fix all vulnerabilities.

### Tasks

**Task 1: Setup vulnerable application**
```bash
cd ~/devsecops-workshop/lab6-vulnerability-remediation

# Create legacy Dockerfile
cat > Dockerfile.legacy <<EOF
FROM ubuntu:18.04

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    curl \
    wget \
    netcat

COPY app.py /app/
COPY requirements.txt /app/

WORKDIR /app

RUN pip3 install -r requirements.txt

EXPOSE 8080

CMD ["python3", "app.py"]
EOF

# Create requirements with old versions
cat > requirements.txt <<EOF
Flask==1.0.0
requests==2.18.0
urllib3==1.24.0
Jinja2==2.10
EOF

# Create simple app
cat > app.py <<EOF
from flask import Flask
import requests

app = Flask(__name__)

@app.route('/')
def home():
    return "Legacy App"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF
```

**Task 2: Initial security assessment**
```bash
# Build legacy image
docker build -f Dockerfile.legacy -t legacy-app:v1.0 .

# Comprehensive scan
trivy image legacy-app:v1.0 > initial-scan.txt

# Count vulnerabilities by severity
echo "=== Vulnerability Summary ==="
echo "CRITICAL: $(grep -c "CRITICAL" initial-scan.txt || echo 0)"
echo "HIGH: $(grep -c "HIGH" initial-scan.txt || echo 0)"
echo "MEDIUM: $(grep -c "MEDIUM" initial-scan.txt || echo 0)"
echo "LOW: $(grep -c "LOW" initial-scan.txt || echo 0)"
```

**Task 3: Create remediation plan**
```bash
# Generate detailed report
trivy image --format json --output remediation-plan.json legacy-app:v1.0

# Extract fixable vulnerabilities
cat remediation-plan.json | jq -r '
  .Results[].Vulnerabilities[] | 
  select(.FixedVersion != null and .FixedVersion != "") | 
  "\(.PkgName): \(.InstalledVersion) -> \(.FixedVersion) (Severity: \(.Severity))"
' > fixable-vulns.txt

echo "=== Fixable Vulnerabilities ==="
cat fixable-vulns.txt
```

**Task 4: Step-by-step remediation**

**Step 1: Update base image**
```bash
cat > Dockerfile.step1 <<EOF
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    curl \
    wget \
    netcat

COPY app.py /app/
COPY requirements.txt /app/

WORKDIR /app

RUN pip3 install -r requirements.txt

EXPOSE 8080

CMD ["python3", "app.py"]
EOF

docker build -f Dockerfile.step1 -t legacy-app:v1.1 .
trivy image --severity CRITICAL,HIGH legacy-app:v1.1
```

**Step 2: Update Python dependencies**
```bash
cat > requirements.txt <<EOF
Flask==2.3.0
requests==2.31.0
urllib3==2.0.7
Jinja2==3.1.2
EOF

docker build -f Dockerfile.step1 -t legacy-app:v1.2 .
trivy image --severity CRITICAL,HIGH legacy-app:v1.2
```

**Step 3: Remove unnecessary packages**
```bash
cat > Dockerfile.step2 <<EOF
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY app.py /app/
COPY requirements.txt /app/

WORKDIR /app

RUN pip3 install --no-cache-dir -r requirements.txt

EXPOSE 8080

CMD ["python3", "app.py"]
EOF

docker build -f Dockerfile.step2 -t legacy-app:v1.3 .
trivy image --severity CRITICAL,HIGH legacy-app:v1.3
```

**Step 4: Add security best practices**
```bash
cat > Dockerfile.final <<EOF
FROM ubuntu:22.04

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy and install requirements
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appuser app.py .

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/ || exit 1

# Run application
CMD ["python3", "app.py"]
EOF

docker build -f Dockerfile.final -t legacy-app:v2.0 .
trivy image --severity CRITICAL,HIGH,MEDIUM legacy-app:v2.0
```

**Task 5: Compare before and after**
```bash
# Scan all versions
trivy image --severity CRITICAL,HIGH legacy-app:v1.0 > scan-v1.0.txt
trivy image --severity CRITICAL,HIGH legacy-app:v2.0 > scan-v2.0.txt

# Compare
echo "=== Version 1.0 (Legacy) ==="
echo "Critical: $(grep -c "CRITICAL" scan-v1.0.txt || echo 0)"
echo "High: $(grep -c "HIGH" scan-v1.0.txt || echo 0)"

echo "=== Version 2.0 (Remediated) ==="
echo "Critical: $(grep -c "CRITICAL" scan-v2.0.txt || echo 0)"
echo "High: $(grep -c "HIGH" scan-v2.0.txt || echo 0)"

# Generate comparison report
cat > remediation-summary.md <<EOF
# Remediation Summary

## Before (v1.0)
- Base Image: ubuntu:18.04
- Python Dependencies: Outdated
- Security Practices: None
- User: root

## After (v2.0)
- Base Image: ubuntu:22.04
- Python Dependencies: Updated
- Security Practices: Implemented
- User: appuser (non-root)

## Results
- Critical Vulnerabilities: $(grep -c "CRITICAL" scan-v1.0.txt || echo 0) → $(grep -c "CRITICAL" scan-v2.0.txt || echo 0)
- High Vulnerabilities: $(grep -c "HIGH" scan-v1.0.txt || echo 0) → $(grep -c "HIGH" scan-v2.0.txt || echo 0)
EOF

cat remediation-summary.md
```

**Expected Outcome:**
- Complete vulnerability remediation workflow
- Measure security improvement
- Document remediation process
- Implement security best practices

---

## Challenge: Real-World Scenario

### The Challenge
Build a secure microservice from scratch with all security measures in place.

### Requirements

1. **Application:**
   - Simple REST API with health endpoint
   - Uses external dependencies
   - Connects to database

2. **Security Requirements:**
   - No CRITICAL or HIGH vulnerabilities
   - No hardcoded secrets
   - Runs as non-root user
   - Minimal attack surface
   - Proper resource limits
   - Configuration validation

3. **CI/CD:**
   - Automated security scanning
   - Security gates in pipeline
   - SBOM generation
   - Report artifacts

### Evaluation Criteria

```bash
# Run all checks
cd ~/devsecops-workshop/challenge

# 1. Vulnerability scan - must have 0 CRITICAL/HIGH
trivy image --severity CRITICAL,HIGH --exit-code 1 my-service:latest

# 2. Secret scan - must have 0 secrets
trivy fs --security-checks secret --exit-code 1 .

# 3. IaC scan - must pass all checks
trivy config --exit-code 1 .

# 4. SBOM generation - must complete
trivy image --format cyclonedx --output sbom.json my-service:latest

# 5. Security best practices
trivy image --format json my-service:latest | jq '.Results[].Misconfigurations[]'
```

### Success Criteria
- All scans pass
- SBOM generated successfully
- Dockerfile follows security best practices
- No secrets in code or configuration
- Documentation complete

---

## Workshop Completion Checklist

- [ ] Lab 1: Container vulnerability scanning completed
- [ ] Lab 2: Secret detection and remediation completed
- [ ] Lab 3: IaC security scanning completed
- [ ] Lab 4: SBOM generation and analysis completed
- [ ] Lab 5: CI/CD integration implemented
- [ ] Lab 6: Vulnerability remediation workflow completed
- [ ] Challenge: Secure microservice built and validated

## Additional Resources

**Practice Environments:**
- [TryHackMe DevSecOps](https://tryhackme.com)
- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
- [Damn Vulnerable Web Application](https://github.com/digininja/DVWA)

**Documentation:**
- [Trivy Official Docs](https://aquasecurity.github.io/trivy/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)

**Tools to Explore:**
- Snyk
- Grype
- Checkov
- Kubesec
- Hadolint

---

## Next Steps

1. Complete all labs in sequence
2. Document your findings
3. Create your own vulnerable/secure scenarios
4. Practice integrating into real projects
5. Share results with your team