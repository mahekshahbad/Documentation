# Secret Scanning with TruffleHog

## Overview
Secret scanning helps detect exposed credentials such as API keys, tokens, passwords, and private keys in source code repositories. Identifying secrets early helps prevent unauthorized access, data leaks, and security incidents.

To evaluate automated secret detection in our repositories, we explored using **TruffleHog** for secret scanning during Pull Request and push workflows.

---

## Tool Used

**TruffleHog**

TruffleHog is an open-source secret scanning tool that detects sensitive information in repositories and Git history.

Key features:
- Detects API keys, tokens, passwords, and private keys
- Scans repository files and commit history
- Integrates with CI/CD pipelines
- Can automatically run during Pull Requests

---

## Implementation

### Repository Used
Testing was performed in the **test-access** repository within the Medica Dev Platform GitHub organization.

### Workflow File Location

```
.github/workflows/trufflehog.yml
```

### GitHub Action Configuration

```yaml
name: TruffleHog Secret Scan

on:
  pull_request:
  push:

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: .
```

---

## Workflow Behavior

The workflow triggers automatically when:

- A Pull Request is created
- Code is pushed to the repository

Workflow process:

```
Developer creates Pull Request
        ↓
GitHub Actions workflow triggers
        ↓
TruffleHog scans repository
        ↓
Secrets detected → PR check fails
No secrets detected → PR check passes
```

---

## Testing Performed

A test Pull Request was created with a sample file containing a simulated secret.

Example test code:

```javascript
const SECRET_KEY = "TEST_SECRET_1234";
```

This test was used to verify whether the automated scan would detect potential secrets during the Pull Request workflow.

---

## Issue Encountered

During testing, the GitHub Action failed during the **repository checkout step**.

Error message observed:

```
The repository owner has an IP allow list enabled, and the GitHub Actions runner IP is not permitted to access this repository.
Error: 403
```

---
