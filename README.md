# Secret Scanning with TruffleHog

This document describes how secret scanning is implemented using **TruffleHog** within GitHub Actions workflows.  
The purpose of this implementation is to detect exposed credentials such as API keys, tokens, passwords, and private keys before code is merged into protected branches.

The implementation uses **custom internal modules** designed for consistent secret scanning across repositories.

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

Example workflow location:

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
          echo "Starting secret scan"

          trufflehog filesystem . \
            --json \
            --no-update \
            > scan-results.json
```

---

## Custom TruffleHog Module

To maintain consistency across repositories, a **custom internal module** is used for secret scanning.

This module runs the TruffleHog scan, processes the results, and determines whether the pipeline should pass or fail.

Example implementation:

```yaml
name: Internal Secret Scan Module

on:
  workflow_call:

jobs:
  secret-scan:
    runs-on: self-hosted

    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Prepare Scan Variables
        run: |
          echo "Preparing secret scanning environment"
          SCAN_TARGET="."
          RESULT_FILE="scan-output.json"

      - name: Execute TruffleHog Scan
        run: |
          echo "Running TruffleHog scan"

          trufflehog filesystem "$SCAN_TARGET" \
            --json \
            --no-update \
            > "$RESULT_FILE" 2>&1 || true

      - name: Evaluate Scan Output
        run: |
          echo "Evaluating scan results"

          if [ -s scan-output.json ]; then
            SECRET_TOTAL=$(grep -c '"DetectorName"' scan-output.json || echo "0")
          else
            SECRET_TOTAL=0
          fi

          echo "Secrets detected: $SECRET_TOTAL"

          if [ "$SECRET_TOTAL" -gt 0 ]; then
            echo "Secret scan failed"
            exit 1
          else
            echo "Secret scan passed"
          fi
```

Advantages of this module:

- Centralized implementation
- Consistent scanning logic
- Easier maintenance
- Reduced duplication across repositories

---

## Advanced TruffleHog Module Implementation

An advanced module can enhance the scanning process by:

- Excluding unnecessary paths
- Performing filesystem and history scans
- Parsing scan results
- Automatically failing the pipeline when secrets are detected

Example implementation:

```yaml
name: Advanced Secret Scan

on:
  workflow_call:

jobs:
  trufflehog-advanced-scan:
    runs-on: self-hosted

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Create Exclusion File
        run: |
          echo ".git/" > .scan-exclude

      - name: Filesystem Secret Scan
        run: |
          trufflehog filesystem . \
            --json \
            --no-update \
            --exclude-paths=.scan-exclude \
            > filesystem-results.json 2>&1 || true

      - name: Git History Secret Scan
        run: |
          trufflehog git file://. \
            --json \
            --no-update \
            > history-results.json 2>&1 || true

      - name: Validate Scan Results
        run: |
          SECRET_COUNT=$(grep -c '"DetectorName"' *.json || echo "0")

          echo "Total secrets detected: $SECRET_COUNT"

          if [ "$SECRET_COUNT" -gt 0 ]; then
            echo "Secrets detected in repository"
            exit 1
          else
            echo "No secrets detected"
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

- Detects secrets committed in historical commits  
- Identifies credentials removed in later commits  
- Provides deeper visibility into repository security  

---

## Pull Request Comment Integration

To improve developer visibility, the workflow can automatically post scan results on the Pull Request.

Example implementation:

```yaml
- name: Comment Scan Results
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');

      let message = "### TruffleHog Secret Scan Result\n\n";

      if (fs.existsSync("filesystem-results.json") && fs.statSync("filesystem-results.json").size > 0) {
        message += "⚠️ Potential secrets detected. Please review the workflow logs.";
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

To validate the secret scanning workflow:

1. Create a test branch  
2. Add a file containing a sample credential  

Example:

```javascript
const SECRET_KEY = "TEST_SECRET_1234";
```

3. Create a Pull Request  
4. Verify the workflow execution  
5. Confirm that the secret scan detects the credential  

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
