Member 4 – CI/CD Pipeline, Security Automation and Secrets Management

1. Student Details
Student ID: IT24101451 
Name: Dewminie R.A.D.M

2. Selected Domain
DevSecOps Pipeline and Automated Security Quality Gates.

3. Objective & Responsibilities
To build and configure an automated GitHub Actions CI/CD pipeline embedding four mandatory security gates, enforcing security thresholds, and managing sensitive credentials securely without hardcoding.

4. Technology Stack & Security Tools
•	GitHub Actions (CI/CD Automation)
•	Semgrep (Gate 1 - SAST)
•	npm audit (Gate 2 - Dependency Scanning)
•	Gitleaks (Gate 3 - Secret Scanning)
•	Trivy (Gate 4 - Container Scanning)

5. Pipeline Architecture & Workflow Flow
The pipeline triggers automatically on code pushes and pull requests, moving sequentially through build, test, SAST, dependency analysis, secret scanning, Docker image build, and container vulnerability scanning.

6. Four Mandatory Security Gates
• Gate 1 (SAST): Utilizes Semgrep to scan application source code for security anti-patterns, vulnerabilities, and misconfigurations.

• Gate 2 (Dependency Scanning): Runs npm audit against project package manifests to identify known third-party vulnerabilities.

• Gate 3 (Secret Scanning): Employs Gitleaks with custom allowlist configurations (.gitleaks.toml) to prevent sensitive API keys, tokens, or passwords from leaking.

• Gate 4 (Container Scanning): Uses Trivy to scan the built container image for operating system and package-level vulnerabilities.

7. Pipeline Failure Demonstration
To validate security gate enforcement, the pipeline utilizes error handling—returning non-zero exit codes (such as exit code 1) when critical violations or unauthorized secrets are detected, explicitly halting the build and blocking unsafe code deployments.

8. Secrets Management
Strictly adheres to security policies by avoiding plain-text credentials or connection strings in the repository or configuration files, utilizing GitHub Actions Secrets instead.

9. Trust Boundary
The trust boundary exists between the automated GitHub Actions runner environment, external security scanning engines, and the target container image being evaluated before production release.

10. Evidence
Screenshots and logs demonstrate successful GitHub Actions workflow execution, security gate inspection steps, and the blocked build state triggered by vulnerability detection.
