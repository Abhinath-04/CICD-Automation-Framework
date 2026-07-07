# CICD-Automation-Framework

Reference implementation of a self-hosted, security-integrated CI/CD pipeline, developed at the Cyber Security Section, CDAC Thiruvananthapuram, under the IoT Network Security project. This repository accompanies the paper:


## Overview

The pipeline is built on a self-hosted GitLab instance and covers the full CI/CD lifecycle: static and dynamic code analysis, SBOM generation and dependency vulnerability tracking, containerization, and Kubernetes-based deployment.

**Toolchain:**
- **Source control / CI orchestration:** GitLab CI/CD (self-hosted)
- **Static analysis (SAST):** SonarQube
- **Software Bill of Materials (SBOM):** CycloneDX
- **Dependency vulnerability tracking (SCA):** Dependency-Track
- **Container vulnerability scanning:** Trivy
- **Dynamic analysis (DAST):** OWASP ZAP
- **Build:** Maven
- **Containerization:** Docker
- **Orchestration:** Kubernetes, Helm
- **Artifact management:** Nexus

## Repository Contents

### Paper Evidence

| File | Description |
|---|---|
| [`supplementary-data.md`](./supplementary-data.md) | Consolidated supplementary data for the paper — SonarQube code quality trend, Dependency-Track SCA results, runner-level resource utilization (Tables VI–VII), and pipeline gate validation summary (Table VIII) |
| [`pipeline-gate-test-cases.md`](./pipeline-gate-test-cases.md) | Detailed evidence for pipeline gate validation — success and failure test cases with job logs and screenshots |
| [`screenshots/`](./screenshots) | Evidence screenshots referenced in `pipeline-gate-test-cases.md` |

### Setup Guides

| File | Description |
|---|---|
| [`Installation_Guide-Gitlab_community.md`](./Installation_Guide-Gitlab_community.md) | Self-hosted GitLab Community Edition installation |
| [`Installation_Guide-Gitlab_Runner.md`](./Installation_Guide-Gitlab_Runner.md) | GitLab Runner installation and configuration |
| [`Installation_Guide_-_Nexus.md`](./Installation_Guide_-_Nexus.md) | Sonatype Nexus Repository Manager installation |
| [`SonarQube_Community_Edition_Setup.md`](./SonarQube_Community_Edition_Setup.md) | SonarQube Community Edition installation and Eclipse/SonarLint integration |
| [`Guide_-_Containerization_and_Nexus_Docker_Registry_Integration.md`](./Guide_-_Containerization_and_Nexus_Docker_Registry_Integration.md) | Docker registry setup on Nexus, image build patterns, and GitLab CI/CD publishing |

## Pipeline Stages

```
validate → sbom-upload → build → docker-publish → scan → deploy → dast-scan
```

| Stage | Jobs | Purpose |
|---|---|---|
| validate | code-quality-and-test, header_quality_check, generate_bom_csv, sonarqube_analysis | Unit tests, code style checks, SBOM generation, static analysis |
| sbom-upload | dependency-track-upload | Upload SBOM to Dependency-Track for continuous SCA |
| build | maven-build | Application build |
| docker-publish | docker-publish | Container image build and push |
| scan | trivy-container-scan | Container image vulnerability scan |
| deploy | deploy-trigger / deploy-dev / deploy-test / deploy-release | Kubernetes deployment via Helm |
| dast-scan | owasp-zap-full-scan | Dynamic application security testing against the deployed build |

## Citation

If referencing this repository in academic work, please cite the associated IC3 2026 paper. Supplementary data files in this repository are referenced directly from the manuscript.

## Contact

Cyber Security Section, CDAC Thiruvananthapuram.
