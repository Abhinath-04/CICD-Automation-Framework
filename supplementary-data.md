# Supplementary Data

**Paper:** An Open DevOps Automation System for Continuous Secure Software Delivery (IC3 2026)


---

## 1. Code Quality — SonarQube (SAST)

**Source:** TSP-Core project, SonarQube Community Build v26.4.0.121862, Activity graph (7 Jul 2026, 17:20 IST).

| Metric | Value |
|---|---|
| Baseline issue count (pre-remediation, ~Jan–Mar 2026) | ~2,000 |
| Issue count post-remediation (version 0.1.0, 3 Jul 2026) | ~0 |
| Version tagged | `0.1.0` — 3 Jul 2026, 10:00 AM |
| Snapshot version | `0.1.0-SNAPSHOT` — 3 Jul 2026, 6:40 PM |
| Reduction | ~100% of tracked issues resolved between baseline and 0.1.0 release |

---

## 2. Security — Dependency-Track (SCA / SBOM)

**Source:** Dependency-Track dashboard, `tsp-core` / `main` branch (7 Jul 2026, 17:21 IST).

| Metric | Value |
|---|---|
| Components tracked (from SBOM) | 176 |
| Critical vulnerabilities | 0 |
| High vulnerabilities | 1 |
| Medium vulnerabilities | 1 |
| Low vulnerabilities | 0 |
| Unassigned | 0 |
| Aggregate risk score | 8 |
| Last BOM import | 3 Jul 2026, 18:40:19 |
| Last vulnerability analysis | 6 Jul 2026, 17:36:32 |
| Last measurement | 7 Jul 2026, 16:36:38 |
| Audit vulnerabilities tracked | 2 |
| Policy violations | 0 |

---

## 3. Runner-Level Resource Utilization

**Method:** `docker stats` sampled at 2-second intervals across the full execution window; job-to-stage correlation performed via exact match of `started_at`/`finished_at` timestamps from the GitLab Jobs API. Runner configured with `concurrent=16`. CPU is reported both per-core-normalized (Docker convention, 100% = 1 core) and host-normalized (÷96 cores).

### Table VI — TSP-Web Pipeline (#9559, main branch, N=9 jobs)

| Stage | Job | Duration (s) | CPU % mean/peak (per-core) | CPU % mean/peak (host-norm, 96 cores) | Mem mean/peak |
|---|---|---|---|---|---|
| validate | code-quality-and-test | 133.9 | 274.5 / 1711.2 | 2.86 / 17.83 | 498.5 MiB / 1.61 GiB |
| validate | header_quality_check | 9.9 | 99.1 / 99.1 | 1.03 / 1.03 | 7.1 / 7.1 MiB |
| validate | generate_bom_csv | 43.0 | 51.5 / 127.4 | 0.54 / 1.33 | 38.5 / 103.2 MiB |
| sbom-upload | dependency-track-upload | 16.2 | 113.5 / 127.9 | 1.18 / 1.33 | 18.2 / 27.9 MiB |
| validate | sonarqube_analysis | 135.4 | 149.9 / 619.5 | 1.56 / 6.45 | 1.41 GiB / 4.53 GiB |
| build | maven-build | 97.1 | 136.1 / 484.4 | 1.42 / 5.05 | 431.4 MiB / 1.35 GiB |
| docker-publish | docker-publish | 63.5 | 21.5 / 198.0 | 0.22 / 2.06 | 105.6 / 146.9 MiB |
| scan | trivy-container-scan | 154.4 | 107.4 / 609.8 | 1.12 / 6.35 | 129.8 / 286.7 MiB |
| dast-scan | owasp-zap-full-scan | 59.1 | 255.3 / 1407.8 | 2.66 / 14.66 | 2.06 GiB / 5.33 GiB |
| deploy | deploy-trigger (bridge, server-side) | 5.6 | N/A | N/A | N/A |

Peak resource consumers were `sonarqube_analysis` (4.53 GiB) and `owasp-zap-full-scan` (5.33 GiB, 1407.8% CPU), both JVM-based tools. Total pipeline duration: 17:38:08–17:54:02 (~15.9 minutes).

### Table VII — TSP-Deps Pipeline (#9562, dev branch, downstream trigger)

| Stage | Job | Duration (s) | CPU % mean/peak | Mem mean/peak | Attribution confidence |
|---|---|---|---|---|---|
| package | package-helm | 9.1 | — | — | Not captured — logger initialization gap |
| deploy | deploy-dev | 90.1 | 1.3 / 23.6 | 124.2 / 129.9 MiB | High — uniquely longest-duration match |
| deploy | deploy-test + deploy-release (combined) | 63.2–63.3 | 1.4–36.5 mean / 14.4–296.6 peak (range across both) | 74–83 MiB | Cannot be individually distinguished — near-identical durations, no timing signal separates them |

**Limitations:**
1. `package-helm` resource profile was not captured due to a logger initialization gap.
2. `deploy-test` and `deploy-release` executed concurrently with near-identical durations (63.3s and 63.2s); container-level data alone does not permit unambiguous attribution between them, and results are reported as a combined range rather than per-job figures.
3. Short-duration jobs (`header_quality_check`, `dependency-track-upload`) are represented by only 1–2 samples at the 2-second polling interval; true peak values may exceed those reported here.

**Column definitions:**
- *Duration (s)*: wall-clock time (`finished_at − started_at`, GitLab Jobs API); elapsed real time, not CPU time.
- *CPU % mean/peak (per-core)*: Docker's native convention, 100% = one saturated core; values above 100% indicate multi-core usage (e.g., 1711.2% ≈ 17 cores at that instant).
- *CPU % mean/peak (host-normalized)*: the same values divided by 96 host cores, expressed as a fraction of total host capacity.
- *Mem mean/peak*: Docker-reported resident memory, averaged and peaked across all samples during the job's lifetime.

---

## 4. Pipeline Gate Validation — Success and Failure Test Cases

Full evidence (screenshots, job logs) is provided in [`pipeline-gate-test-cases.md`](./pipeline-gate-test-cases.md).

Project: `tsp-web` (main branch) / `tsp-deps` (dev branch), self-hosted GitLab (`git-css.tvm.cdac.in`), GitLab Runner 18.6.3.

### Table VIII — Pipeline Gate Test Cases

| # | Test Case | Input / Trigger | Gate | Observed Behavior | Result |
|---|---|---|---|---|---|
| 1 | Success — clean build | Commit `321d8297` to `main`, no quality/vuln issues | All stages | validate → sbom-upload → build → docker-publish → scan → deploy → dast-scan all passed; downstream `deploy-trigger: TSP-Deps #9651` fired | Pass |
| 2 | Failure — SAST violation | Commit to `main` breaching SonarQube quality gate | `sonarqube_analysis` | Job failed within validate stage; `code-quality-and-test`, `generate_bom_csv`, `header_quality_check` passed independently | Blocked as expected |
| 3 | Vulnerable dependency detection | SBOM uploaded to Dependency-Track on every build | `dependency-track-upload` → Dependency-Track analysis | Upload job passed (HTTP 200); Dependency-Track audit subsequently identified 29 vulnerabilities across `tsp-web` components (2 Critical, 12 High, 13 Medium, 2 Low), including `netty-handler` CVE-2026-45416 (High) and `spring-security-web` CVE-2026-47838 (High) | Detection confirmed |
| 4 | DAST finding | `owasp-zap-full-scan` run against deployed build (commit `321d8297`) | `owasp-zap-full-scan` | Job passed (scan completed, 1m18s); generated ZAP report flagged 1 High-risk alert (CORS Misconfiguration, 4 instances), 3 Low, 2 Informational | Detection confirmed |
| 5 | Failure — deployment misconfiguration | Commit `8aba95b6` to `dev`, upstream trigger from TSP-Web #9637 | `package-helm` | `package-helm` job failed in package stage; downstream `deploy-dev`, `deploy-release`, `deploy-test` jobs did not execute | Blocked as expected |

Test Cases 3 and 4 validate detection rather than hard pipeline blocking: findings are surfaced in Dependency-Track and the ZAP report for review and triage rather than auto-failing the build, consistent with the project's current gate configuration.

---

*Data collected 3–7 July 2026 from live CDAC-TVM self-hosted infrastructure (SonarQube, Dependency-Track, OWASP ZAP, GitLab Runner).*
