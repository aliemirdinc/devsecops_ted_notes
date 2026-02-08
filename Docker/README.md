# Docker Security Guide

A comprehensive, beginner-friendly guide to Docker and container security.

## Reading Order

Start from the beginning and read in sequence:

1. **[What is Docker?](01-what-is-docker.md)**
   - Basic concepts and terminology
   - Containers vs VMs
   - Docker architecture

2. **[Images and Containers](02-images-and-containers.md)**
   - Image structure and lifecycle
   - Container creation and management
   - Working with registries

3. **[Dockerfile Basics](03-dockerfile-basics.md)**
   - Writing Dockerfiles
   - Core instructions (FROM, RUN, COPY, CMD, etc.)
   - Build optimization

4. **[Docker Layers](04-docker-layers.md)**
   - Layer architecture
   - Caching strategy
   - Multi-stage builds
   - Layer optimization

5. **[Security Risks](05-security-risks.md)**
   - Running as root
   - Public image trust
   - Shared kernel vulnerabilities
   - Secrets in images
   - Common attack scenarios

6. **[Security Best Practices](06-security-best-practices.md)**
   - Minimal base images
   - Non-root users
   - Multi-stage builds
   - Secret management
   - Image scanning

7. **[Runtime Security](07-runtime-security.md)**
   - Secure container execution
   - Security contexts
   - Network isolation
   - Monitoring and intrusion detection

8. **[CI/CD Integration](08-cicd-integration.md)**
   - Security gates in pipelines
   - GitLab CI examples
   - GitHub Actions examples
   - Automated scanning

## Quick Reference

**Build Secure Image:**
```bash
docker build -t myapp:v1 .
trivy image --severity HIGH,CRITICAL myapp:v1
```

**Run Securely:**
```bash
docker run \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --user 1000:1000 \
  --memory="512m" \
  myapp:v1
```

**Scan Everything:**
```bash
trivy config Dockerfile                    # Scan Dockerfile
trivy fs --security-checks secret .        # Scan for secrets
trivy image myapp:v1                       # Scan image
```

## Target Audience

- Beginners learning Docker
- Developers implementing security
- DevOps engineers building pipelines
- Security professionals auditing containers

## Prerequisites

- Basic Linux command line knowledge
- Understanding of applications and dependencies
- No prior Docker experience required

## Learning Path

**Beginner (Files 1-2):** Understand what Docker is and how it works

**Intermediate (Files 3-4):** Learn to build optimized images

**Advanced (Files 5-6):** Master security concepts and best practices

**Expert (Files 7-8):** Implement production-grade security

## Key Concepts

- **Image:** Template for containers
- **Container:** Running instance of an image
- **Layer:** Building block of images
- **Dockerfile:** Recipe to build images
- **Registry:** Storage for images
- **Security:** Not default, must be implemented

## Security Checklist

- [ ] Run as non-root user
- [ ] Use minimal base images
- [ ] Scan for vulnerabilities
- [ ] No secrets in images
- [ ] Multi-stage builds
- [ ] Resource limits set
- [ ] Read-only filesystem
- [ ] Drop capabilities
- [ ] Network isolation
- [ ] Regular updates

## Additional Resources

- [Official Docker Documentation](https://docs.docker.com/)
- [Trivy Scanner](https://github.com/aquasecurity/trivy)
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)

## Contributing

This is a learning resource. Feel free to add notes, examples, or improvements.
