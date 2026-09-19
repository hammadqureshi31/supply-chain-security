# Supply Chain Security Lab

A practical DevSecOps CI/CD lab demonstrating **shift-left security, container security, SBOM generation, and software supply-chain integrity** using GitHub Actions.

The project builds a small Node.js application into a hardened Docker image and subjects it to multiple security gates before publishing and signing the final container image.

The pipeline combines:

**Secret Scanning → SCA → SAST → Container Build → Container Scanning → SBOM → Registry → Keyless Signing → Signature Verification**

---

## 1. Project Overview

Modern CI/CD pipelines should not treat security as a final step after deployment.

This project demonstrates how security checks can be integrated directly into the software delivery workflow so that insecure code, leaked credentials, vulnerable dependencies, or vulnerable container artifacts can prevent an artifact from progressing through the pipeline.

The project intentionally remains small at the application level so that the focus stays on **DevSecOps controls and supply-chain security rather than application complexity**.

### Objectives

* Implement shift-left security controls in CI/CD
* Detect accidentally committed secrets
* Scan application dependencies for known vulnerabilities
* Perform static application security testing
* Build a hardened container image
* Scan the final container artifact for vulnerabilities
* Generate an SBOM
* Publish the validated image to GitHub Container Registry
* Sign the image using Cosign and GitHub OIDC
* Verify the image signature
* Practice deliberate security failures and recovery
* Investigate security findings instead of blindly suppressing them

---

# 2. DevSecOps Pipeline

```text
                    Developer Push
                          │
                          ▼
                 ┌─────────────────┐
                 │     Checkout    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    Gitleaks     │
                 │ Secret Scanning │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    npm audit    │
                 │      SCA        │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │     Semgrep     │
                 │      SAST       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   Docker Build  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │      Trivy      │
                 │ Container Scan  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  Syft / SBOM    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │      GHCR       │
                 │ Image Registry  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │     Cosign      │
                 │  Keyless Sign   │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Cosign Verify   │
                 │ Artifact Trust  │
                 └─────────────────┘
```

The pipeline is designed so that security validation happens **before the container artifact is considered trusted**.

---

# 3. Security Controls

| Stage              | Tool        | Purpose                                           |
| ------------------ | ----------- | ------------------------------------------------- |
| Secret scanning    | Gitleaks    | Detect accidentally committed credentials/secrets |
| SCA                | npm audit   | Detect vulnerable application dependencies        |
| SAST               | Semgrep     | Detect insecure source-code patterns              |
| Container build    | Docker      | Build the application artifact                    |
| Container scanning | Trivy       | Detect vulnerabilities in the final image         |
| SBOM               | Syft        | Generate software inventory                       |
| Registry           | GHCR        | Store the validated image                         |
| Signing            | Cosign      | Sign the container image                          |
| Identity           | GitHub OIDC | Provide keyless signing identity                  |
| Verification       | Cosign      | Verify image authenticity/integrity               |

---

# 4. Application

The application is intentionally minimal.

It is an Express application with two endpoints.

### `/`

Returns:

```json
{
  "message": "supply chain security lab"
}
```

### `/healthz`

Returns:

```json
{
  "status": "ok"
}
```

The application is deliberately simple because the primary objective is to demonstrate the security pipeline rather than application functionality.

---

# 5. Technology Stack

* Node.js
* Express
* Docker
* GitHub Actions
* Gitleaks
* npm audit
* Semgrep
* Trivy
* Syft
* GitHub Container Registry
* Cosign
* GitHub OIDC

---

# 6. Repository Structure

```text
supply-chain-security/
├── .github/
│   └── workflows/
│       └── security.yml
│
├── app/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   └── server.js
│
├── scripts/
│   └── local-scan.sh
│
├── Makefile
├── project.yaml
├── .gitignore
└── README.md
```

Generated SBOM files are intentionally ignored from Git and generated during validation/CI instead.

---

# 7. Docker Image

The application is packaged into a lightweight Node.js Alpine container.

The Dockerfile follows several container-hardening principles:

* Uses an Alpine-based Node.js image
* Installs only production dependencies
* Copies only the required application files
* Runs the application as the non-root `node` user
* Removes unnecessary runtime tooling
* Exposes only the application port
* Starts the application directly with Node.js

Example runtime configuration:

```dockerfile
USER node

EXPOSE 3000

CMD ["node", "server.js"]
```

---

# 8. Container Hardening Investigation

One of the most useful lessons from this project came from comparing application dependency scanning with container scanning.

Initially:

```text
npm audit
    ↓
0 application vulnerabilities
```

But:

```text
Trivy container scan
    ↓
4 HIGH vulnerabilities
```

Instead of assuming the scanners were contradicting each other, the container image was investigated.

The vulnerable packages were traced to tooling bundled inside the Node.js base image, including npm/corepack/yarn-related runtime tooling.

These tools were not required to execute the application.

The Dockerfile was therefore hardened by removing unnecessary runtime tooling:

```dockerfile
RUN rm -rf \
    /usr/local/lib/node_modules/npm \
    /usr/local/lib/node_modules/corepack \
    /opt/yarn-v1.22.22
```

The application continued to run successfully because the runtime only required Node.js and the installed application dependencies.

A subsequent Trivy scan reported no HIGH/CRITICAL vulnerabilities.

### Lesson

`npm audit` and Trivy answer different questions.

```text
npm audit
    ↓
Are the application's Node.js dependencies vulnerable?

Trivy
    ↓
Is the shipped container artifact vulnerable?
```

A clean application dependency audit does not automatically mean that the final container artifact is secure.

---

# 9. Local Setup

## Prerequisites

Install:

* Node.js
* npm
* Docker
* Trivy
* Syft
* Gitleaks
* Git

Cosign is also required for local signing/verification exercises.

Semgrep can be executed locally through its official Docker image.

---

# 10. Validate the Project

The Makefile provides a basic validation target:

```bash
make validate
```

This performs:

```text
Node.js syntax validation
        +
GitHub Actions workflow YAML validation
```

Expected result:

```text
node --check app/server.js
workflow yaml ok
```

---

# 11. Run the Application Locally

Build and run:

```bash
make up
```

The application listens on:

```text
http://localhost:8080
```

The container internally listens on port `3000`.

Test:

```bash
curl http://localhost:8080/
```

Expected:

```json
{"message":"supply chain security lab"}
```

Health check:

```bash
curl http://localhost:8080/healthz
```

Expected:

```json
{"status":"ok"}
```

Stop the container:

```bash
make down
```

---

# 12. Gitleaks — Secret Scanning

Gitleaks scans the repository for accidentally committed credentials and other sensitive values.

Run locally:

```bash
gitleaks detect --source . --verbose
```

A clean scan produced:

```text
3 commits scanned.
scan completed
no leaks found
```

## Deliberate Failure

A fake AWS access key was introduced into a temporary file:

```text
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
```

Gitleaks detected it as:

```text
RuleID: aws-access-token
leaks found: 1
```

The temporary test file was removed and the repository was scanned again.

Result:

```text
no leaks found
```

### Lesson

Secret scanning should happen before an artifact is built or published.

---

# 13. Software Composition Analysis — npm audit

Application dependencies are checked with:

```bash
npm --prefix app audit --omit=dev
```

The actual project dependency tree currently reports:

```text
found 0 vulnerabilities
```

## Deliberate Failure

A temporary project was created using an intentionally vulnerable version of lodash.

Running:

```bash
npm audit --omit=dev
```

produced:

```text
1 high severity vulnerability
```

The vulnerable dependency was then removed from the temporary test environment.

The real project remained unchanged and continued to report:

```text
found 0 vulnerabilities
```

### Lesson

Dependency scanning provides an early security gate before vulnerable third-party packages become part of the build artifact.

---

# 14. SAST — Semgrep

Semgrep performs static analysis against the source code.

The project can be scanned locally using the official Semgrep container:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep \
  semgrep --config=p/ci /src/app
```

The real application produced:

```text
Findings: 0
0 blocking
```

## Deliberate Failure

A temporary JavaScript file was created containing:

```javascript
eval(req.query.code);
```

Semgrep detected the user-controlled data flowing into `eval()` and reported a **blocking security finding**.

The finding identified the risk of arbitrary code execution when untrusted request data reaches `eval()`.

After removing the temporary insecure code, the real project was scanned again.

Result:

```text
0 findings
0 blocking
```

### Lesson

SAST can detect dangerous coding patterns before they reach the container build stage.

---

# 15. Container Security — Trivy

The container image is scanned after building.

Example:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  supply-chain-demo:local
```

The initial scan identified four HIGH vulnerabilities associated with tooling included in the base image.

The findings were investigated rather than simply suppressed.

After removing unnecessary runtime tooling from the Docker image, the image was rebuilt and rescanned.

Final result:

```text
Alpine vulnerabilities: 0
Node package vulnerabilities: 0
HIGH: 0
CRITICAL: 0
```

### Important Principle

The goal was not:

> "Make Trivy ignore the vulnerability."

The goal was:

> "Determine why the vulnerability exists and remove unnecessary vulnerable components from the artifact."

---

# 16. SBOM Generation

Syft is used to generate an SPDX SBOM.

Run:

```bash
make sbom
```

The generated file is:

```text
sbom.spdx.json
```

The local image was cataloged successfully, producing an SPDX 2.3 SBOM.

The generated SBOM is ignored by Git because the CI pipeline generates it as part of the build process.

In GitHub Actions, the SBOM is uploaded as a workflow artifact.

### Why SBOM?

An SBOM provides visibility into the software components contained in an artifact.

It can be used for:

* Dependency visibility
* Vulnerability correlation
* Software inventory
* Incident response
* Compliance workflows
* Supply-chain analysis

---

# 17. GitHub Container Registry

After passing the security gates, the container image is published to GitHub Container Registry (GHCR).

The image is tagged using the Git commit SHA rather than relying only on a mutable tag.

Conceptually:

```text
ghcr.io/<repository>/supply-chain-demo:<commit-sha>
```

Using an immutable commit-based identifier makes it easier to associate:

```text
Source Commit
      ↓
Container Image
      ↓
SBOM
      ↓
Signature
```

---

# 18. Cosign Keyless Signing

The final container image is signed using Cosign.

The workflow uses GitHub's OIDC identity instead of storing a long-lived Cosign private key in GitHub Secrets.

The signing flow is:

```text
GitHub Actions
      │
      ▼
GitHub OIDC Identity
      │
      ▼
Cosign Keyless Signing
      │
      ▼
Container Image Signature
```

This avoids maintaining a traditional long-lived signing private key inside the repository or CI environment.

---

# 19. Signature Verification

After signing, the workflow verifies the image signature.

Conceptually:

```text
Container Image
      │
      ▼
Cosign Verify
      │
      ├── Valid identity
      ├── Valid OIDC issuer
      └── Valid signature
```

The workflow verifies that the image was signed by the expected GitHub Actions identity associated with the repository workflow.

This creates a trust relationship between:

```text
GitHub Repository
       ↓
GitHub Actions Workflow
       ↓
Container Image
       ↓
Cosign Signature
```

---

# 20. GitHub Actions Pipeline

The CI workflow performs the following sequence:

```text
1. Checkout
       ↓
2. Gitleaks
       ↓
3. npm ci
       ↓
4. npm audit
       ↓
5. Semgrep SAST
       ↓
6. Docker Build
       ↓
7. Trivy Container Scan
       ↓
8. Generate SBOM
       ↓
9. Upload SBOM
       ↓
10. Login to GHCR
       ↓
11. Push Container
       ↓
12. Install Cosign
       ↓
13. Keyless Sign
       ↓
14. Verify Signature
```

Pull requests run the security analysis but do not publish or sign container images.

The `main` branch performs the complete artifact publication and signing flow.

---

# 21. Security Gate Philosophy

Each security layer has a different responsibility.

```text
Gitleaks
    ↓
"Did someone commit a secret?"

npm audit
    ↓
"Are application dependencies vulnerable?"

Semgrep
    ↓
"Does the source contain dangerous coding patterns?"

Trivy
    ↓
"Is the final container artifact vulnerable?"

Syft
    ↓
"What software is actually inside the artifact?"

Cosign
    ↓
"Can we establish artifact authenticity/integrity?"
```

No single scanner provides complete supply-chain security.

The controls complement one another.

---

# 22. Deliberate Failure Labs

This project intentionally included failure scenarios rather than only demonstrating successful scans.

### Failure 1 — Secret Detection

```text
Fake AWS credential
        ↓
Gitleaks
        ↓
Secret detected
        ↓
Security gate failure
        ↓
Credential removed
        ↓
Clean scan
```

### Failure 2 — Vulnerable Dependency

```text
Vulnerable lodash
        ↓
npm audit
        ↓
HIGH vulnerability
        ↓
Security gate failure
        ↓
Dependency removed/fixed
        ↓
Clean audit
```

### Failure 3 — Insecure Source Code

```text
eval(req.query.code)
        ↓
Semgrep
        ↓
Blocking finding
        ↓
Security gate failure
        ↓
Insecure code removed
        ↓
Clean SAST scan
```

### Failure 4 — Container Vulnerabilities

```text
Clean npm audit
        ↓
Trivy finds container vulnerabilities
        ↓
Investigate image contents
        ↓
Trace findings to unnecessary runtime tooling
        ↓
Remove tooling
        ↓
Rebuild
        ↓
Trivy clean
```

These exercises demonstrate that security work involves **investigation and remediation**, not simply running scanners.

---

# 23. Makefile Commands

Available commands:

```bash
make validate
```

Validate application syntax and workflow YAML.

```bash
make up
```

Build and run the local application container.

```bash
make logs
```

Follow container logs.

```bash
make down
```

Stop and remove the local container.

```bash
make scan
```

Build the image, run Trivy, and generate an SBOM.

```bash
make sbom
```

Generate an SPDX SBOM from the local image.

---

# 24. Local Scan Script

The project includes:

```text
scripts/local-scan.sh
```

The script automates:

```text
Docker Build
     ↓
Trivy Scan
     ↓
Syft SBOM
```

This provides a lightweight local equivalent of the core container-security portion of the CI pipeline.

---

# 25. Security Design Decisions

### Non-root container

The application runs as the built-in Node.js non-root user.

This reduces the impact of a potential application compromise compared with running the process as root.

### Production dependencies only

The Docker image installs:

```bash
npm install --omit=dev
```

Development dependencies are not required at runtime.

### Remove unnecessary runtime tooling

npm, Corepack, and Yarn tooling are not required after the application image is built.

Removing unnecessary tooling reduces the attack surface and eliminated the vulnerabilities discovered during the initial Trivy investigation.

### Immutable image identification

The CI pipeline uses the Git commit SHA to identify the published image.

This creates a direct relationship between source revision and artifact.

### Keyless signing

Cosign uses GitHub OIDC rather than a long-lived private signing key stored in repository secrets.

### Security gates before publication

The container is not published until the relevant security stages succeed.

---

# 26. Lessons Learned

## 1. Different scanners answer different questions

A clean dependency audit does not mean the container is automatically secure.

Application dependencies and the final container artifact must be evaluated separately.

---

## 2. Security findings require investigation

A vulnerability report is only the beginning.

The important question is:

```text
Why does this vulnerability exist?
```

In this project, the answer was unnecessary runtime tooling inherited from the base image.

---

## 3. Minimize runtime contents

Every package included in a production container increases the potential attack surface.

If software is not required at runtime, removing it can be both a security and operational improvement.

---

## 4. Security gates should fail loudly

The pipeline uses non-zero exit codes for security failures.

This makes security findings actionable instead of informational.

---

## 5. SBOMs provide artifact visibility

A container image should not be treated as an opaque binary.

Generating an SBOM provides an inventory of the software components inside the artifact.

---

## 6. Artifact integrity matters

Vulnerability scanning answers:

> "Does this artifact contain known vulnerabilities?"

Signing answers a different question:

> "Can we establish trust in this artifact's identity and integrity?"

Both are useful controls.

---

## 7. Deliberate failure is better than theoretical configuration

The security controls were not only configured.

They were deliberately tested with:

* Fake credentials
* Vulnerable dependencies
* Insecure source code
* Vulnerable container contents

This demonstrated that the gates actually behaved as expected.

---

# 27. Current Scope

This project intentionally focuses on **CI/CD supply-chain security**.

It does not attempt to implement every possible DevSecOps capability.

The project does not currently include:

* Kubernetes deployment
* Infrastructure as Code
* Production hosting
* DAST against a deployed environment
* TLS scanning
* Production database migrations
* Multi-environment promotion
* Runtime SIEM integration

These are intentionally outside the scope of this compact supply-chain security lab.

---

# 28. Possible Production Extensions

A larger production platform could extend this pipeline with:

```text
SAST
  +
SCA
  +
Secret Scanning
  +
Container Scanning
  +
SBOM
  +
Image Signing
  +
Admission Verification
  +
DAST
  +
Infrastructure Scanning
  +
Policy Enforcement
  +
Environment Promotion
```

For example, Kubernetes admission policies could require container images to have valid signatures before they are allowed to run.

That would extend the trust model from:

```text
CI Security
```

to:

```text
CI Security
      ↓
Artifact Signing
      ↓
Registry
      ↓
Admission Verification
      ↓
Runtime
```

Such functionality is intentionally left outside this project's current scope.

---

# 29. Interview Talking Points

### "Why did you use multiple security scanners?"

Because each tool addresses a different layer.

Gitleaks detects secrets, npm audit analyzes application dependencies, Semgrep analyzes source code, and Trivy analyzes the final container artifact.

---

### "Why wasn't npm audit enough?"

`npm audit` only evaluates the Node.js dependency tree.

The final container can still contain vulnerabilities from the operating system or other packages/tools included in the image.

That happened during this project.

---

### "What did you do when Trivy reported vulnerabilities?"

I investigated where the vulnerable packages were coming from instead of immediately suppressing the findings.

They were associated with unnecessary npm/Corepack/Yarn tooling inherited from the Node base image.

Since those tools were not required at runtime, I removed them, rebuilt the image, and rescanned it.

---

### "Why generate an SBOM?"

An SBOM provides visibility into the software components contained in an artifact.

It makes dependency inventory and vulnerability correlation easier and provides useful information for incident response and compliance processes.

---

### "Why sign the image?"

Scanning tells us whether known vulnerabilities exist.

Signing provides a mechanism to establish trust in the artifact's origin and integrity.

---

### "Why use Cosign keyless signing?"

Keyless signing avoids maintaining a long-lived private signing key in CI secrets.

GitHub Actions can obtain an OIDC identity, and Cosign can use that identity as part of the signing and verification process.

---

### "How did you test the security gates?"

I deliberately introduced failures.

I tested a fake AWS credential with Gitleaks, a vulnerable lodash dependency with npm audit, and an `eval()` vulnerability with Semgrep.

I also investigated actual container vulnerabilities reported by Trivy.

After remediation, the corresponding scans returned clean results.

---

# 30. Final Outcome

This project demonstrates a compact but realistic DevSecOps supply-chain workflow:

```text
Source Code
    │
    ├── Secret Security
    ├── Dependency Security
    └── Static Analysis
           │
           ▼
      Docker Artifact
           │
           ├── Vulnerability Scan
           └── SBOM
           │
           ▼
          GHCR
           │
           ▼
      Keyless Signing
           │
           ▼
    Signature Verification
```

The main objective was not to collect security tools.

The objective was to build a workflow where:

**insecure source or dependencies can be detected early, the final artifact is scanned, its contents are inventoried, and the resulting artifact can be cryptographically signed and verified.**

---

## Project Status

**Status:** Complete

**Classification:** DevSecOps / Supply-Chain Security Lab

**Deployment:** CI/CD ready

**Cloud infrastructure:** None required

**Primary platform:** GitHub Actions

**Container Registry:** GitHub Container Registry

**Signing:** Cosign + GitHub OIDC

**SBOM format:** SPDX JSON

