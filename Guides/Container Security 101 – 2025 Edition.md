# Container Security 101 – 2025 Edition

Secure containers = minimal + hardened + verified + monitored

## Core Principles (Never Skip These)

- **Use minimal base images**  
  Prefer `alpine`, `distroless`, `scratch`, or `chainguard` images  
  → Dramatically reduces attack surface

- **Run as non-root user**  
  ```dockerfile
  USER 10001:10001
  # or
  USER nobody:nogroup

- **Multi-stage builds**  
  Keep only runtime dependencies, drop compilers, package managers, etc.

- **Never bake secrets into images**  
  No API keys, passwords, certs, tokens → use secrets management (Kubernetes Secrets, AWS Secrets Manager, HashiCorp Vault, etc.)

- **Scan images regularly**  
  Tools: Trivy, Grype, Snyk, Clair, Syft  
  Integrate into CI/CD (fail on critical/high vulnerabilities)

- **Keep images updated**  
  Automate base image and dependency updates (Renovate, Dependabot, GitHub Actions)

## Hardening Best Practices

```dockerfile
--cap-drop=ALL
--cap-add=NET_BIND_SERVICE   # only if needed
--security-opt=no-new-privileges
--read-only                         # root filesystem
--tmpfs=/tmp                        # writable tmp
--pids-limit=100
--memory=512m --cpu-shares=256
```

- Pin images by digest (not tag) in production  
  `ghcr.io/chainguard/nginx@sha256:abc123...`

- Sign images with cosign / sigstore  
  Enforce verification via policy (Kyverno, OPA Gatekeeper, Cosign Policy Controller)

- Use restricted seccomp, AppArmor, or SELinux profiles

- Avoid `--privileged` and host namespaces (`--pid=host`, `--network=host`, etc.)

## Dockerfile & CI Security

- Lint Dockerfiles: `hadolint`, `checkov`, `dockerfile-lint`
- Scan IaC and policies: `checkov`, `tfsec`, `terrascan`
- Fail builds on critical vulnerabilities

## Runtime Security

- Behavioral monitoring: Falco, Tracee, Tetragon (eBPF-powered)
- Stronger isolation when needed: gVisor, Firecracker, Kata Containers
- Network policies (Kubernetes) or `--icc=false` (Docker)

**The no-fluff, battle-tested cheat sheet**

| Rule | Why it matters | One-liner fix |
|------|----------------|--------------|
| Use minimal base images | 95 % of CVEs live in OS packages you don’t need | `cgr.dev/chainguard/python`, `gcr.io/distroless/...`, `scratch` |
| Run as non-root | Root in container = root on host if breakout | `USER 65532` or `USER nonroot` |
| Multi-stage builds | Build tools = free backdoors | Copy only the final binary/artifacts |
| Never bake secrets | Secrets in layers are forever (even if deleted) | Use Kubernetes Secrets, Vault, AWS SSM, etc. |
| Pin images by digest | “latest” = surprise CVEs tomorrow | `image@sha256:abc123...` |
| Scan every build | One outdated lib can sink the ship | `trivy image --severity HIGH,CRITICAL --exit-code 1` |
| Sign + verify images | Supply-chain attacks are the #1 threat in 2025 | `cosign sign` + Kyverno/Cosign Policy Controller |
| Drop all capabilities | Containers don’t need CAP_SYS_ADMIN | `--cap-drop=ALL` + add only what you need |
| Read-only filesystem | Can’t write malware if there’s no write | `readOnlyRootFilesystem: true` + `tmpfs` on /tmp |
| Resource + PID limits | Stops crypto-miners and fork bombs | `resources.limits` + `pids-limit=100` |

## Image Size Reality Check (Dec 2025)

| Language       | Base                  | Final Image (compressed) | CVEs (typical) |
|----------------|-----------------------|--------------------------|----------------|
| Python (FastAPI) | chainguard/python   | 45–65 MB                 | 0              |
| Node.js (Express) | chainguard/node    | 140–160 MB               | 0–2            |
| Java (Spring)  | chainguard/jre        | 90–120 MB                | 0              |

(Compared to official images: 800–1500 MB and 200–800 CVEs)

## One-liner Dockerfiles (copy-paste)

### Python
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt
COPY . .

FROM cgr.dev/chainguard/python:latest-dev
WORKDIR /app
COPY --from=build --chown=65532:65532 /root/.local /home/nonroot/.local
COPY --from=build --chown=65532:65532 /app .
USER 65532
ENV PATH="/home/nonroot/.local/bin:$PATH"
CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
```

### Node.js
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json .
RUN npm ci --omit=dev
COPY . .

FROM cgr.dev/chainguard/node:latest
WORKDIR /app
COPY --from=build --chown=65532:65532 /app .
USER 65532
ENV NODE_ENV=production
CMD ["node", "server.js"]
```
#### Java (Spring Boot / Quarkus)
```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline
COPY src ./src
RUN ./mvnw package -DskipTests

FROM cgr.dev/chainguard/jre:latest
WORKDIR /app
COPY --from=build --chown=65532:65532 /app/target/*.jar app.jar
USER 65532
EXPOSE 8080
HEALTHCHECK CMD wget --spider -q http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

## Quick Checklist 

```markdown
[ ] Minimal base image (distroless/alpine/chainguard)
[ ] Non-root user
[ ] Multi-stage build
[ ] No secrets in image
[ ] Image scanned & no critical vulns
[ ] Base image pinned by digest
[ ] Image signed with cosign
[ ] Capabilities dropped
[ ] no-new-privileges enabled
[ ] Read-only filesystem + tmpfs /tmp
[ ] Resource limits set
[ ] Runtime monitoring enabled
```
Happy and secure containerizing!
