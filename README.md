# Secret Scanning with TruffleHog

This document describes how to implement **secret scanning using TruffleHog** within GitHub Actions workflows.  
The objective is to detect exposed credentials such as API keys, tokens, passwords, and private keys before code is merged into protected branches.

The implementation uses **custom internal modules** to ensure a standardized approach to secret scanning across repositories.

---

## Table of Contents

- [Overview](#overview)
- [Key Capabilities](#key-capabilities)
- [Prerequisites](#prerequisites)
- [GitHub Actions Pipeline Configuration](#github-actions-pipeline-configuration)
- [Custom TruffleHog Module](#custom-trufflehog-module)
- [Advanced TruffleHog Module Implementation](#advanced-trufflehog-module-implementation)
- [Full Git History Scanning](#full-git-history-scanning)
- [Pull Request Comment Integration](#pull-request-comment-integration)
- [Workflow Execution Flow](#workflow-execution-flow)
- [Testing the Implementation](#testing-the-implementation)
- [Future Enhancements](#future-enhancements)
- [References](#references)

---

## Overview

Secret scanning helps detect sensitive credentials accidentally committed to source code repositories.

Examples include:

- API keys  
- Access tokens  
- Database credentials  
- Private keys  
- Cloud provider credentials  

Integrating secret scanning into CI/CD pipelines ensures potential credential leaks are detected early during development workflows such as Pull Requests or push events.

---

## Key Capabilities

The TruffleHog secret scanning implementation provides the following capabilities:

- Detects exposed secrets using entropy and pattern detection  
- Scans repository files during CI/CD pipeline execution  
- Generates structured output for automated analysis  
- Automatically fails pipelines when secrets are detected  
- Integrates with GitHub Actions workflows  
- Supports centralized reusable modules across repositories  

---

## Prerequisites

Before implementing secret scanning, ensure the following:

- GitHub repository with Actions enabled  
- Self-hosted GitHub Actions runner configured  
- TruffleHog CLI available on the runner environment  
- Repository permissions allowing workflow execution  

---

## GitHub Actions Pipeline Configuration

Secret scanning is integrated into GitHub workflows using a CI job triggered during Pull Requests or code pushes.

Example workflow file location:

```
.github/workflows/security-scan.yml
```

Example configuration:

```yaml
name: Security Scan

on:
  pull_request:
  push:

jobs:
  trufflehog-scan:
    runs-on: self-hosted

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Run TruffleHog Secret Scan
        run: |
          echo "Running TruffleHog secret scan"

          trufflehog filesystem . \
            --json \
            --no-update \
            > trufflehog-results.json
```

---

## Custom TruffleHog Module

To ensure consistency across repositories, the secret scanning logic can be implemented as a **custom reusable module**.

Example reusable module:

```yaml
name: Reusable TruffleHog Secret Scan

on:
  workflow_call:
    inputs:
      working-directory:
        required: false
        type: string
        default: "."

jobs:
  secret-scan:
    runs-on: self-hosted

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Run TruffleHog Scan
        working-directory: ${{ inputs.working-directory }}
        run: |
          trufflehog filesystem . \
            --json \
            --no-update \
            > trufflehog-results.json
```

Benefits:

- Centralized security configuration  
- Consistent scanning across repositories  
- Simplified maintenance  
- Reduced workflow duplication  

---

## Advanced TruffleHog Module Implementation

For improved flexibility and better result handling, an advanced module can:

- Exclude unnecessary paths  
- Generate JSON output  
- Parse scan results  
- Count detected secrets  
- Fail the pipeline automatically  

Example:

```yaml
name: Advanced TruffleHog Scan

on:
  workflow_call:

jobs:
  trufflehog-scan:
    runs-on: self-hosted

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Run Filesystem Scan
        run: |
          echo ".git/" > .trufflehog-exclude

          trufflehog filesystem . \
            --json \
            --no-update \
            --exclude-paths=.trufflehog-exclude \
            > filesystem-scan.json 2>&1 || true

          rm -f .trufflehog-exclude

      - name: Run Git History Scan
        run: |
          trufflehog git file://. \
            --json \
            --no-update \
            > history-scan.json 2>&1 || true

      - name: Count Detected Secrets
        run: |
          SECRET_COUNT=$(grep -c '"DetectorName"' *.json || echo "0")

          echo "Secrets detected: $SECRET_COUNT"

          if [ "$SECRET_COUNT" -gt 0 ]; then
            echo "Secret scan failed"
            exit 1
          else
            echo "Secret scan passed"
          fi
```

---

## Full Git History Scanning

TruffleHog can scan the **entire Git commit history** to detect secrets that may have existed in earlier commits.

Example command:

```
trufflehog git file://. --json --no-update
```

Benefits:

- Detects secrets committed in previous commits  
- Identifies credentials removed later  
- Provides deeper repository security visibility  

---

## Pull Request Comment Integration

To improve developer visibility, the workflow can automatically comment scan results on the Pull Request.

Example step:

```yaml
- name: Comment Scan Results on PR
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');

      let message = "### TruffleHog Secret Scan Result\n\n";

      if (fs.existsSync("filesystem-scan.json") && fs.statSync("filesystem-scan.json").size > 0) {
        message += "⚠️ Potential secrets detected.\n\nCheck workflow logs.";
      } else {
        message += "✅ No secrets detected.";
      }

      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: message
      });
```

---

## Workflow Execution Flow

```
Developer creates Pull Request
        ↓
GitHub Actions Workflow Triggered
        ↓
Self-hosted Runner Executes Job
        ↓
TruffleHog Scans Repository Files
        ↓
TruffleHog Scans Git Commit History
        ↓
Secrets Detected → Pipeline fails
No Secrets Detected → Pipeline passes
```

---

## Testing the Implementation

1. Create a test branch  
2. Add a sample secret  

Example:

```javascript
const SECRET_KEY = "TEST_SECRET_1234";
```

3. Create a Pull Request  
4. Verify workflow triggers  
5. Confirm the secret scan detects the credential  

---

## Future Enhancements

Possible improvements include:

- Generate SARIF security reports  
- Integrate results with security dashboards  
- Schedule periodic repository scans  
- Send alerts when secrets are detected  

---

## References

- TruffleHog Documentation  
- GitHub Actions Documentation
