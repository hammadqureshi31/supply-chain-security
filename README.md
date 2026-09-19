# Supply Chain Security Lab

A hands-on DevSecOps project that secures a small Node.js container from source code to signed artifact: secret scanning, dependency audit, static analysis, container build and scan, SBOM generation, registry publication, keyless signing, and signature verification, all automated in GitHub Actions.

[![Security pipeline](https://github.com/hammadqureshi31/supply-chain-security/actions/workflows/security.yml/badge.svg)](https://github.com/hammadqureshi31/supply-chain-security/actions/workflows/security.yml)
![CI](https://img.shields.io/badge/CI-GitHub_Actions-2088FF?logo=githubactions&logoColor=white)
![Container](https://img.shields.io/badge/container-Docker-2496ED?logo=docker&logoColor=white)
![Scanner](https://img.shields.io/badge/image_scan-Trivy-1904DA)
![SBOM](https://img.shields.io/badge/SBOM-SPDX_2.3-blue)
![Signing](https://img.shields.io/badge/signing-Cosign_keyless-4A4A9F)

The application is intentionally tiny (an Express app with two endpoints) so the focus stays on the **security controls around the artifact**, not on application complexity.

---

## At a Glance

**The problem:** a CI/CD pipeline that only builds and pushes an image tells you nothing about whether the code leaks secrets, the dependencies are vulnerable, the image is clean, or the published artifact is the one your pipeline actually produced.

**What this project does about it:** every push runs a chain of security gates before the image is trusted, then the published image is signed and the signature is verified.

**The most interesting part:** a clean `npm audit` (0 vulnerabilities) sat next to a Trivy scan reporting **4 HIGH** findings on the same project. Instead of suppressing the findings, I traced them to tooling bundled in the base image, removed what the runtime did not need, rebuilt, and rescanned to **0 HIGH / 0 CRITICAL**. See [Case Study: Container Hardening Investigation](#case-study-container-hardening-investigation).

### What this demonstrates

| Area | Implementation |
| --- | --- |
| Secret detection | Gitleaks |
| Dependency security (SCA) | npm audit |
| Static analysis (SAST) | Semgrep |
| Container build and hardening | Docker, Alpine-based Node.js image, non-root user |
| Container vulnerability scanning | Trivy |
| Software inventory (SBOM) | Syft, SPDX JSON |
| Artifact registry | GitHub Container Registry (GHCR) |
| Artifact signing | Cosign, keyless |
| Workload identity | GitHub OIDC (no long-lived signing key) |
| Signature verification | Cosign verify in the pipeline |
| CI/CD | GitHub Actions |
| Security testing of the gates | Deliberate failure labs for secrets, dependencies, source code, and container contents |

### Architecture

Three groups of stages run in GitHub Actions. Pull requests run the scans; `main` runs the full publish-sign-verify flow.

```mermaid
flowchart TD
    DEV[Developer push or PR] --> PRE

    subgraph PRE[1. Pre-build gates]
        direction LR
        G["Gitleaks<br/>secrets"] --> A["npm audit<br/>dependencies"] --> S["Semgrep<br/>source code"]
    end

    subgraph BLD[2. Build and inspect]
        direction LR
        B["Docker<br/>build"] --> T["Trivy<br/>image scan"] --> SB["Syft<br/>SBOM"]
    end

    subgraph PUB[3. Publish and trust - main only]
        direction LR
        P["Push<br/>to GHCR"] --> C["Cosign<br/>keyless sign"] --> V["Cosign<br/>verify"]
    end

    PRE --> BLD --> PUB
```

### Key engineering outcomes

- Automated security gates in CI/CD covering secrets, dependencies, source code, and the final container image
- Each gate was tested by **deliberately breaking it** (fake AWS key, vulnerable lodash, `eval()` on request input) and confirming it fails, then recovering to a clean state
- Container attack surface reduced by investigating and removing unneeded runtime tooling: **4 HIGH to 0 HIGH / 0 CRITICAL**
- SPDX SBOM generated from the built image and uploaded as a workflow artifact
- Image published to GHCR with an immutable, commit-SHA-based identifier
- Image signed with Cosign using GitHub OIDC identity, with **no long-lived private key** stored in CI secrets
- Signature verified inside the pipeline against the expected GitHub Actions identity

### Results

| Metric | Before | After |
| --- | --- | --- |
| Trivy HIGH findings on the image | 4 | 0 |
| Trivy CRITICAL findings | none reported in the initial scan | 0 |
| Alpine package vulnerabilities (final scan) | not itemised | 0 |
| Node package vulnerabilities (final scan) | not itemised | 0 |
| npm / Corepack / Yarn tooling in runtime image | present (inherited from base image) | removed |
| `npm audit` application dependency findings | 0 | 0 (unchanged, real project) |
| Application still runs after hardening | n/a | Yes |

### Technology stack

| Group | Technologies |
| --- | --- |
| Application | Node.js, Express |
| Container | Docker, GitHub Container Registry |
| CI/CD | GitHub Actions |
| Security scanning | Gitleaks, npm audit, Semgrep, Trivy |
| Supply chain | Syft (SBOM), Cosign (signing and verification), GitHub OIDC (identity) |
| Local automation | Make, shell script |

### Contents

[Overview](#overview) · [Security Pipeline](#security-pipeline) · [Controls in Depth](#controls-in-depth) · [Application and Image](#application-and-container-image) · [Case Study: Container Hardening](#case-study-container-hardening-investigation) · [SBOM](#sbom) · [Registry, Signing, Verification](#registry-signing-and-verification) · [Failure Labs](#deliberate-failure-labs) · [Design Decisions and Lessons](#design-decisions-and-lessons-learned) · [Quick Start](#quick-start) · [Repository Structure](#repository-structure) · [Scope and Extensions](#scope-and-extensions) · [Design Q&A](#design-qa) · [Project Metadata](#project-metadata)

---

## Overview

Security is often bolted on after deployment. This project shows the opposite, **shift-left** approach: security checks are wired into the delivery workflow itself, so leaked credentials, vulnerable dependencies, insecure code, or a vulnerable image can stop an artifact before it is published or trusted.

The pipeline covers this progression:

**Secret Scanning → SCA → SAST → Container Build → Container Scanning → SBOM → Registry → Keyless Signing → Signature Verification**

### Objectives

- Implement shift-left security controls in CI/CD
- Detect accidentally committed secrets
- Scan application dependencies for known vulnerabilities
- Perform static application security testing
- Build a hardened container image
- Scan the final container artifact for vulnerabilities
- Generate an SBOM
- Publish the validated image to GitHub Container Registry
- Sign the image using Cosign and GitHub OIDC
- Verify the image signature
- Practice deliberate security failures and recovery
- Investigate security findings instead of blindly suppressing them

---

## Security Pipeline

**Simple explanation:** each stage asks a different security question. If any answer is "no", the pipeline fails and nothing further is published.

**Technical explanation:** stages are sequential steps in a single workflow (`.github/workflows/security.yml`). Security tools return non-zero exit codes on findings, which fails the job and prevents later steps (push, sign) from running.

### Stage responsibilities

| Stage | Tool | Question it answers | What it protects | If it fails |
| --- | --- | --- | --- | --- |
| Secret scanning | Gitleaks | Did someone commit a secret? | Credentials and tokens in the repo | Pipeline stops; nothing is built or published |
| SCA | npm audit | Are application dependencies vulnerable? | Third-party Node.js packages | Pipeline stops |
| SAST | Semgrep | Does the source contain dangerous coding patterns? | First-party source code | Pipeline stops on blocking findings |
| Container build | Docker | Can the hardened artifact be built? | Build reproducibility | Pipeline stops |
| Container scan | Trivy | Is the final container artifact vulnerable? | OS packages and packages inside the image | Pipeline stops; image is not published |
| SBOM | Syft | What software is actually inside the artifact? | Visibility and inventory | Pipeline stops |
| Registry | GHCR | Where is the validated image stored? | Artifact storage after gates pass | Pipeline stops |
| Signing | Cosign + GitHub OIDC | Can authenticity and integrity be established? | Artifact identity and tamper evidence | Image stays unsigned |
| Verification | Cosign | Does the signature match the expected identity? | Trust in the published artifact | Pipeline stops |

No single scanner provides complete supply-chain security. The controls complement one another.

### Security gate flow

Any failed gate stops the job, so an artifact never progresses to publication or signing.

```mermaid
flowchart TD
    subgraph GATES[Security gates in order]
        direction LR
        G1[Gitleaks] --> G2[npm audit] --> G3[Semgrep] --> G4["Build +<br/>Trivy"]
    end
    G4 -->|all pass| OK["SBOM, push to GHCR,<br/>sign, verify"]
    GATES -.->|any failure| X["Non-zero exit<br/>job fails, nothing published"]
```

### Workflow sequence

The CI workflow runs these steps in order:

| # | Step | # | Step |
| --- | --- | --- | --- |
| 1 | Checkout | 8 | Generate SBOM |
| 2 | Gitleaks | 9 | Upload SBOM |
| 3 | `npm ci` | 10 | Login to GHCR |
| 4 | `npm audit` | 11 | Push container |
| 5 | Semgrep SAST | 12 | Install Cosign |
| 6 | Docker build | 13 | Keyless sign |
| 7 | Trivy container scan | 14 | Verify signature |

### Pull requests vs `main`

```mermaid
flowchart LR
    E[Push or PR] --> Q{Which?}
    Q -->|Pull request| R["Security analysis only<br/>no publish, no sign"]
    Q -->|main| F["Full flow: scans, SBOM,<br/>push, sign, verify"]
```

### Security controls matrix

| Layer | Control | Tool | Detects / protects | Pipeline stage |
| --- | --- | --- | --- | --- |
| Source | Secret detection | Gitleaks | Committed credentials | Early, before build |
| Source | SAST | Semgrep | Insecure code patterns (e.g. untrusted input reaching `eval()`) | Before build |
| Dependencies | SCA | npm audit | Known-vulnerable npm packages | Before build |
| Container | Image scan | Trivy | Vulnerabilities in OS and packages inside the image | After build, before publish |
| Container | Hardening | Dockerfile | Attack surface (non-root user, production deps only, tooling removed) | Build |
| Inventory | SBOM | Syft | What components the artifact contains | After scan |
| Distribution | Registry | GHCR | Stores validated image, tagged by commit SHA | After gates pass |
| Integrity | Signing | Cosign | Artifact authenticity and tamper evidence | After push |
| Identity | Workload identity | GitHub OIDC | Who/what signed, without a long-lived key | At signing |
| Trust | Verification | Cosign verify | Signature matches expected identity and issuer | After signing |

### Defense in depth

```mermaid
flowchart TD
    L1["Source<br/>Gitleaks · Semgrep"] --> L2["Dependencies<br/>npm audit"]
    L2 --> L3["Container<br/>Trivy"]
    L3 --> L4["Inventory<br/>Syft SBOM"]
    L4 --> L5["Integrity + identity<br/>Cosign · GitHub OIDC"]
    L5 --> L6["Verification<br/>cosign verify"]
```

Each layer catches a different class of problem. A clean result at one layer says nothing about the others, which is why the layers are stacked rather than substituted for one another.

---

## Controls in Depth

Each section: what it is, why an engineer should care, how it was used here, and the evidence.

### Gitleaks: secret scanning

**Why it matters:** a credential committed once can live in Git history indefinitely. Catching it in CI, before anything is built or published, is far cheaper than rotating keys after a leak.

Gitleaks scans the repository for accidentally committed credentials and other sensitive values.

```bash
gitleaks detect --source . --verbose
```

Clean scan result:

```text
3 commits scanned.
scan completed
no leaks found
```

*(Commit count is from the time the lab was run.)*

### npm audit: software composition analysis

**Why it matters:** most of an application's code is third-party. Dependency scanning is an early gate before vulnerable packages become part of the build artifact.

```bash
npm --prefix app audit --omit=dev
```

The real project dependency tree reports:

```text
found 0 vulnerabilities
```

### Semgrep: static analysis (SAST)

**Why it matters:** SAST finds dangerous coding patterns in your own source before they reach the container build, such as untrusted request data flowing into code execution.

Semgrep can be run locally through its official container:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep \
  semgrep --config=p/ci /src/app
```

The real application produces:

```text
Findings: 0
0 blocking
```

### Trivy: container vulnerability scanning

**Why it matters:** the shipped image contains more than your application: OS packages and base-image tooling too. Trivy scans the final artifact rather than just the dependency manifest.

```bash
trivy image \
  --severity HIGH,CRITICAL \
  supply-chain-demo:local
```

The initial scan identified four HIGH vulnerabilities associated with tooling included in the base image. They were investigated rather than suppressed; see the [case study](#case-study-container-hardening-investigation). Final scan result:

```text
Alpine vulnerabilities: 0
Node package vulnerabilities: 0
HIGH: 0
CRITICAL: 0
```

### Syft, Cosign, GitHub OIDC

Covered in [SBOM](#sbom) and [Registry, Signing, and Verification](#registry-signing-and-verification).

---

## Application and Container Image

### The application

An Express application with two endpoints. It is deliberately simple because the objective is the security pipeline, not application functionality.

| Endpoint | Response |
| --- | --- |
| `/` | `{"message": "supply chain security lab"}` |
| `/healthz` | `{"status": "ok"}` |

### The Docker image

The application is packaged into a lightweight Node.js Alpine container. The Dockerfile follows these container-hardening principles:

- Uses an Alpine-based Node.js image
- Installs only production dependencies
- Copies only the required application files
- Runs the application as the non-root `node` user
- Removes unnecessary runtime tooling
- Exposes only the application port
- Starts the application directly with Node.js

Example runtime configuration:

```dockerfile
USER node

EXPOSE 3000

CMD ["node", "server.js"]
```

---

## Case Study: Container Hardening Investigation

This is the most instructive part of the project: two scanners disagreed, and resolving the disagreement improved the artifact.

```mermaid
flowchart TD
    A["npm audit<br/>0 vulnerabilities"] --> B["Trivy image scan<br/>4 HIGH"]
    B --> C["Investigate<br/>package ownership"]
    C --> D["Cause: npm, corepack, yarn<br/>bundled in Node base image<br/>not needed at runtime"]
    D --> E["Remove tooling<br/>in Dockerfile"]
    E --> F["Rebuild and rescan"]
    F --> G["0 HIGH, 0 CRITICAL<br/>smaller attack surface"]
```

### 1. Initial state

| Check | Result |
| --- | --- |
| `npm audit` (application dependencies) | 0 vulnerabilities |
| Trivy (container image) | **4 HIGH** vulnerabilities |

Both results were correct. They answer different questions.

### 2. Investigation

Instead of assuming the scanners contradicted each other, the image contents were investigated:

- The vulnerable packages were traced to tooling **bundled inside the Node.js base image**, including npm, Corepack, and Yarn-related runtime tooling.
- These were **base-image tooling, not application dependencies**, which is why `npm audit` never saw them.
- The tools were **not required to execute the application**; the runtime only needs Node.js and the installed production dependencies.

### 3. Remediation

The Dockerfile was hardened by removing the unnecessary runtime tooling:

```dockerfile
RUN rm -rf \
    /usr/local/lib/node_modules/npm \
    /usr/local/lib/node_modules/corepack \
    /opt/yarn-v1.22.22
```

### 4. Verification

The image was rebuilt and rescanned. The application continued to run successfully, and Trivy reported no HIGH/CRITICAL vulnerabilities:

```text
Alpine vulnerabilities: 0
Node package vulnerabilities: 0
HIGH: 0
CRITICAL: 0
```

### 5. Engineering lesson

`npm audit` and Trivy answer different questions:

| Tool | Question |
| --- | --- |
| npm audit | Are the application's Node.js dependencies vulnerable? |
| Trivy | Is the shipped container artifact vulnerable? |

A clean application dependency audit does not automatically mean the final container is secure.

A scanner is not just a red/green gate. The goal was not:

> "Make Trivy ignore the vulnerability."

The goal was:

> "Determine why the vulnerability exists and remove unnecessary vulnerable components from the artifact."

Every package in a production image widens the potential attack surface. If software is not needed at runtime, removing it is both a security and an operational improvement.

---

## SBOM

**Simple explanation:** an SBOM (Software Bill of Materials) is an ingredient list for the container, so its components can be identified and tracked.

**Technical explanation:** Syft analyzes the built container image and produces an SPDX JSON SBOM describing the software components inside the artifact. The local image was cataloged successfully, producing an **SPDX 2.3** document.

| Item | Detail |
| --- | --- |
| Artifact inventoried | The built container image (`supply-chain-demo:local` locally) |
| Tool | Syft |
| Format | SPDX JSON (`sbom.spdx.json`), SPDX 2.3 |
| Local generation | `make sbom` |
| In CI | Generated after the Trivy scan and uploaded as a GitHub Actions workflow artifact |
| In Git | Intentionally ignored; generated during validation and CI instead |

```bash
make sbom          # writes sbom.spdx.json
```

**Why it matters:** a container image should not be treated as an opaque binary. An SBOM supports:

- Dependency visibility
- Vulnerability correlation
- Software inventory
- Incident response
- Compliance workflows
- Supply-chain analysis

---

## Registry, Signing, and Verification

### Registry and tagging

After passing the security gates, the image is published to GitHub Container Registry (GHCR). It is tagged with the **Git commit SHA** rather than relying only on a mutable tag:

```text
ghcr.io/<repository>/supply-chain-demo:<commit-sha>
```

This ties the artifact chain together:

```mermaid
flowchart LR
    A[Source commit] --> B[Container image] --> C[SBOM] --> D[Signature]
```

### Signing and verification flow

**Simple explanation:** the pipeline cryptographically signs the container so consumers can verify that it came through the expected trusted workflow and was not modified after signing.

**Technical explanation:** the workflow uses GitHub's OIDC identity instead of a long-lived Cosign private key stored in GitHub Secrets. Cosign uses that identity for keyless signing, then verifies the signature against the expected GitHub Actions identity.

```mermaid
flowchart TD
    GA["GitHub Actions<br/>workflow run"] -->|obtains| ID["GitHub OIDC<br/>identity token"]
    ID --> CS["Cosign<br/>keyless sign"]
    CS -->|signature stored with image| REG[("GHCR")]
    REG --> VF["Cosign verify"]
    EXP["Expected identity<br/>and OIDC issuer"] --> VF
    VF --> RES{"Match?"}
    RES -->|yes| T[Trusted]
    RES -->|no| R[Rejected]
```

### The trust chain

```mermaid
flowchart LR
    A[Repository] --> B[Actions workflow] --> C[Image] --> D[Cosign signature] --> E[Verification]
```

### Trust model in plain English

1. The image is built and pushed by a specific GitHub Actions workflow in this repository.
2. That workflow is given a short-lived identity by GitHub (OIDC), so no permanent signing key sits in CI secrets to be stolen or leaked.
3. Cosign signs the image under that identity.
4. Verification checks three things: a **valid signature**, a **valid identity** (the expected repository workflow), and a **valid OIDC issuer**.
5. Only an image signed by the expected workflow identity passes.

**Why workload identity over long-lived keys:** a stored private signing key is a high-value secret that must be protected, rotated, and can be exfiltrated. A short-lived identity issued to the running workflow removes that standing secret.

> **Background (general Cosign keyless mechanism, not repo-specific):** in keyless signing, the OIDC token lets Sigstore issue a short-lived certificate that binds an ephemeral signing key to the workflow identity, and the signing event is recorded in a transparency log. Verification then checks the certificate identity and issuer rather than a pre-shared public key.

### What signing does and does not tell you

These controls are related but different. Do not read one as proof of another.

| Control | Answers | Does not answer |
| --- | --- | --- |
| Trivy scan | Does the artifact contain known vulnerabilities? | Who built it, or whether it was modified later |
| Syft SBOM | What software is inside the artifact? | Whether that software is safe or the artifact is authentic |
| Cosign signature | Was it signed by the expected identity, and is it unchanged since signing? (authenticity and integrity) | Whether the image is free of vulnerabilities or the code is secure |
| GitHub OIDC | Which workflow identity signed it, without a long-lived key? | Anything about the image contents |
| Provenance | How and where an artifact was built | Not implemented as a separate provenance attestation in this project |

Vulnerability scanning asks *"does this artifact contain known vulnerabilities?"* Signing asks a different question: *"can we establish trust in this artifact's identity and integrity?"* Both are useful.

### Verifying an image yourself

Illustrative template, not copied from the workflow: substitute your repository, tag, and the workflow identity your pipeline signs with.

```bash
cosign verify \
  ghcr.io/<owner>/<repo>/supply-chain-demo:<commit-sha> \
  --certificate-identity-regexp "https://github.com/<owner>/<repo>/.github/workflows/security.yml@refs/heads/main" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

---

## Deliberate Failure Labs

The security gates were not only configured; they were **tested by breaking them**. Each lab injects a fault, confirms the gate fails, removes the fault, and confirms a clean result.

```mermaid
flowchart LR
    A[Inject fault] --> B[Gate detects] --> C[Gate fails] --> D[Remove fault] --> E[Clean scan]
```

| Lab | Injected fault | Tool | Detection | Recovery and result |
| --- | --- | --- | --- | --- |
| 1. Secret detection | Fake AWS key `AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE` in a temporary file | Gitleaks | `RuleID: aws-access-token`, `leaks found: 1` | Temp file removed; rescan: `no leaks found` |
| 2. Vulnerable dependency | Temporary project using an intentionally vulnerable lodash version | `npm audit --omit=dev` | `1 high severity vulnerability` | Dependency removed from the temporary environment; real project unchanged at `found 0 vulnerabilities` |
| 3. Insecure source | Temporary file containing `eval(req.query.code);` | Semgrep | Blocking finding: user-controlled data flowing into `eval()`, i.e. arbitrary code execution risk | Insecure code removed; rescan: `0 findings`, `0 blocking` |
| 4. Container vulnerabilities | Real finding, not injected: clean `npm audit` but Trivy reports 4 HIGH | Trivy | 4 HIGH, traced to base-image npm/Corepack/Yarn tooling | Tooling removed, image rebuilt; Trivy clean (see [case study](#case-study-container-hardening-investigation)) |

**Lessons from the labs:**

- **Secrets:** secret scanning should happen before an artifact is built or published.
- **Dependencies:** dependency scanning is an early gate before vulnerable third-party packages become part of the build artifact.
- **Source:** SAST can detect dangerous coding patterns before they reach the container build stage.
- **Overall:** security work involves investigation and remediation, not simply running scanners.

---

## Design Decisions and Lessons Learned

### Security design decisions

| Decision | Rationale |
| --- | --- |
| Non-root container | The app runs as the built-in Node.js `node` user, reducing the impact of a potential application compromise compared with running as root. |
| Production dependencies only | The Docker image installs with `npm install --omit=dev`; development dependencies are not needed at runtime. |
| Remove unnecessary runtime tooling | npm, Corepack, and Yarn are not needed after the image is built. Removing them reduces attack surface and eliminated the vulnerabilities found in the initial Trivy investigation. |
| Immutable image identification | The pipeline identifies the published image by Git commit SHA, tying source revision directly to artifact. |
| Keyless signing | Cosign uses GitHub OIDC rather than a long-lived private signing key stored in repository secrets. |
| Security gates before publication | The container is not published until the relevant security stages succeed. |

### Lessons learned

1. **Different scanners answer different questions.** A clean dependency audit does not mean the container is secure. Application dependencies and the final artifact must be evaluated separately.
2. **Security findings require investigation.** A vulnerability report is only the beginning; the key question is *why does this vulnerability exist?* Here, the answer was unnecessary runtime tooling inherited from the base image.
3. **Minimize runtime contents.** Every package in a production container increases potential attack surface.
4. **Security gates should fail loudly.** The pipeline uses non-zero exit codes for security failures, making findings actionable instead of informational.
5. **SBOMs provide artifact visibility.** A container image should not be treated as an opaque binary.
6. **Artifact integrity matters.** Scanning and signing answer different questions, and both are useful.
7. **Deliberate failure beats theoretical configuration.** The gates were tested with fake credentials, vulnerable dependencies, insecure source code, and vulnerable container contents, demonstrating they behave as expected.

---

## Quick Start

### Prerequisites

| Tool | Needed for |
| --- | --- |
| Git | Cloning the repository |
| Node.js and npm | `make validate` syntax check, dependency audit |
| Docker | Building and running the image; running Semgrep via its official image |
| Trivy | Container scanning |
| Syft | SBOM generation |
| Gitleaks | Secret scanning |
| Cosign | Local signing and verification exercises |
| PyYAML (optional) | Workflow YAML parse in `make validate`; skipped if not installed |

### 1. Clone

```bash
git clone https://github.com/hammadqureshi31/supply-chain-security.git
cd supply-chain-security
```

### 2. Validate

```bash
make validate
```

Performs Node.js syntax validation plus a GitHub Actions workflow YAML parse. Expected result:

```text
node --check app/server.js
workflow yaml ok
```

### 3. Run the application locally

```bash
make up
```

The container listens on port `3000` internally and is published on **host port 8081**.

```bash
curl http://localhost:8081/
# {"message":"supply chain security lab"}

curl http://localhost:8081/healthz
# {"status":"ok"}
```

```bash
make logs    # follow container logs
make down    # stop and remove the container
```

### 4. Run the security tooling locally

```bash
make scan    # build image, run Trivy, generate SBOM (via scripts/local-scan.sh)
make sbom    # SPDX SBOM only, written to sbom.spdx.json
```

Individual scanners:

```bash
gitleaks detect --source . --verbose
npm --prefix app audit --omit=dev
docker run --rm -v "$PWD:/src" semgrep/semgrep semgrep --config=p/ci /src/app
trivy image --severity HIGH,CRITICAL supply-chain-demo:local
```

### 5. Trigger and inspect the CI pipeline

- **Pull request:** runs the security analysis only; no image is published or signed.
- **Push to `main`:** runs the full flow: scans, SBOM, push to GHCR, sign, verify.
- **Inspect results:** open the Actions tab for the run, download the SBOM workflow artifact, and check GHCR for the commit-SHA-tagged image.
- **Verify a signed image:** see [Verifying an image yourself](#verifying-an-image-yourself).

### Makefile targets

| Target | Description |
| --- | --- |
| `make help` | List available targets |
| `make validate` | Node syntax check and optional workflow YAML parse |
| `make up` | Build and run the local demo container (host `8081` to container `3000`) |
| `make logs` | Follow demo container logs |
| `make down` | Stop and remove the demo container |
| `make scan` | Build the image, run Trivy, and generate an SBOM (`scripts/local-scan.sh`) |
| `make sbom` | Generate an SPDX SBOM from the local image |

`IMAGE` (default `supply-chain-demo:local`) and `CONTAINER` (default `supply-chain-demo`) can be overridden, e.g. `make up IMAGE=my-image:dev`.

`scripts/local-scan.sh` is a lightweight local equivalent of the container-security portion of the CI pipeline:

```mermaid
flowchart LR
    A[Docker build] --> B[Trivy scan] --> C[Syft SBOM]
```

---

## Repository Structure

```text
supply-chain-security/
├── .github/
│   └── workflows/
│       └── security.yml      # CI security, build, publish, sign, verify pipeline
├── app/
│   ├── Dockerfile            # Hardened Alpine-based Node.js image
│   ├── package.json
│   ├── package-lock.json
│   └── server.js             # Express app: / and /healthz
├── scripts/
│   └── local-scan.sh         # Local build + Trivy + Syft
├── Makefile                  # validate, up, logs, down, scan, sbom
├── project.yaml              # Project metadata
├── .gitignore
└── README.md
```

| Path | Contents |
| --- | --- |
| `.github/workflows/security.yml` | The GitHub Actions pipeline |
| `app/` | Application code, dependency manifests, and Docker configuration |
| `scripts/local-scan.sh` | Local container scan and SBOM automation |
| `Makefile` | Developer entry points |
| `project.yaml` | Project metadata (see [Project Metadata](#project-metadata)) |

Generated SBOM files are intentionally ignored from Git and generated during validation and CI instead.

---

## Scope and Extensions

This project intentionally focuses on **CI/CD supply-chain security**. It does not attempt every DevSecOps capability.

**Not included:**

- Kubernetes deployment
- Infrastructure as Code
- Production hosting
- DAST against a deployed environment
- TLS scanning
- Production database migrations
- Multi-environment promotion
- Runtime SIEM integration

**Possible extensions for a larger platform:** DAST, infrastructure scanning, policy enforcement, environment promotion, and admission-time signature verification. For example, Kubernetes admission policies could require valid signatures before images are allowed to run, extending the trust model beyond CI (dashed = not implemented here):

```mermaid
flowchart LR
    A[CI security] --> B[Artifact signing] --> C[Registry] --> D[Admission verification] --> E[Runtime]
    classDef planned stroke-dasharray: 4 4
    class D,E planned
```

---

## Design Q&A

<details>
<summary><strong>Why did you use multiple security scanners?</strong></summary>

Each tool addresses a different layer. Gitleaks detects secrets, npm audit analyzes application dependencies, Semgrep analyzes source code, and Trivy analyzes the final container artifact.
</details>

<details>
<summary><strong>Why wasn't npm audit enough?</strong></summary>

`npm audit` only evaluates the Node.js dependency tree. The final container can still contain vulnerabilities from the operating system or other packages and tools in the image. That happened during this project.
</details>

<details>
<summary><strong>What did you do when Trivy reported vulnerabilities?</strong></summary>

I investigated where the vulnerable packages came from instead of suppressing the findings. They were associated with unnecessary npm/Corepack/Yarn tooling inherited from the Node base image. Since those tools were not required at runtime, I removed them, rebuilt the image, and rescanned it.
</details>

<details>
<summary><strong>Why generate an SBOM?</strong></summary>

It provides visibility into the components in an artifact, making dependency inventory and vulnerability correlation easier, and supporting incident response and compliance processes.
</details>

<details>
<summary><strong>Why sign the image?</strong></summary>

Scanning tells us whether known vulnerabilities exist. Signing provides a mechanism to establish trust in the artifact's origin and integrity.
</details>

<details>
<summary><strong>Why use Cosign keyless signing?</strong></summary>

It avoids maintaining a long-lived private signing key in CI secrets. GitHub Actions can obtain an OIDC identity, and Cosign can use that identity as part of signing and verification.
</details>

<details>
<summary><strong>How did you test the security gates?</strong></summary>

I deliberately introduced failures: a fake AWS credential for Gitleaks, a vulnerable lodash dependency for npm audit, and an `eval()` vulnerability for Semgrep. I also investigated real container vulnerabilities reported by Trivy. After remediation, the corresponding scans returned clean results.
</details>

---

## Project Metadata

The main objective was not to collect security tools. It was to build a workflow where **insecure source or dependencies can be detected early, the final artifact is scanned, its contents are inventoried, and the resulting artifact can be cryptographically signed and verified.**

| Field | Value |
| --- | --- |
| Status | Complete |
| Classification | DevSecOps / supply-chain security lab (`project.yaml`: `learning`, status `ci_lab`) |
| Deployability | `ci_cd_ready` (per `project.yaml`) |
| Cloud infrastructure | None required |
| Primary platform | GitHub Actions |
| Container registry | GitHub Container Registry |
| Signing | Cosign + GitHub OIDC |
| SBOM format | SPDX JSON |
| Cost risk | Low (`project.yaml`) |
| Validation command | `make validate` |
| Description (`project.yaml`) | Local and CI-based container supply-chain security lab for students |
