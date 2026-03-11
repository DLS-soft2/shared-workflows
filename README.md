# Shared Workflows

Reusable GitHub Actions workflows for all microservices in the DLS-2 system. Each service calls these via `workflow_call` — no duplication across repos.

---

## Workflows

### Python Services

#### `python-tollgate.yaml` — Quality Gate (PRs)
Runs on every pull request to `main`. Posts a summary comment on the PR and fails if quality thresholds aren't met.

**Steps:**
1. **Pylint** — lints all `.py` files, uploads report as artifact. Fails the workflow if score is below 6/10.
2. **Tests** — if a `tests/` or `app/tests/` directory exists, installs dependencies and runs `pytest`. Fails if any tests fail.
3. **Version check** — compares the version in `pyproject.toml` against the latest git tag. Fails if the version hasn't been bumped.

#### `python-build.yaml` — Build & Push (main)
Runs on every push to `main`. Builds a Docker image and publishes it to GHCR.

**Steps:**
1. Extracts the version from `pyproject.toml` and uses `service-specific-input` as the image name.
2. Builds and pushes the image tagged as both `:<version>` and `:latest` to `ghcr.io/dls-soft2/<service-name>`.
3. Creates a git tag and GitHub Release for the version if one doesn't already exist.

### Java Services

#### `java-tollgate.yaml` — Quality Gate (PRs)
Same purpose as the Python tollgate but using Java tooling.

**Steps:**
1. **Checkstyle** — runs `mvn checkstyle:check` and reports violations.
2. **Tests** — runs `mvn test` and parses Surefire reports. Fails if any tests fail.
3. **Version check** — compares the version in `pom.xml` against the latest git tag. Fails if the version hasn't been bumped.

#### `java-build.yaml` — Build & Push (main)
Same purpose as the Python build but compiles with Maven first.

**Steps:**
1. Builds the project with `mvn package`.
2. Extracts the version from `pom.xml` and builds/pushes the Docker image to GHCR.
3. Creates a git tag and GitHub Release for the version if one doesn't already exist.

---

## Usage

### Python services

In your service repo, create `.github/workflows/tollgate.yaml`:

```yaml
name: Tollgate

on:
  pull_request:
    branches:
      - main

jobs:
  tollgate:
    permissions:
      pull-requests: write
      contents: read
    uses: DLS-soft2/shared-workflows/.github/workflows/python-tollgate.yaml@main
    with:
      service-specific-input: "your-service-name"
    secrets: inherit
```

And `.github/workflows/build.yaml`:

```yaml
name: Build

on:
  push:
    branches:
      - main

jobs:
  integrate:
    permissions:
      pull-requests: write
      contents: write
      packages: write
    uses: DLS-soft2/shared-workflows/.github/workflows/python-build.yaml@main
    with:
      service-specific-input: "your-service-name"
    secrets: inherit
```

### Java services

Same structure but pointing to the Java workflows:

`.github/workflows/tollgate.yaml`:

```yaml
name: Tollgate

on:
  pull_request:
    branches:
      - main

jobs:
  tollgate:
    permissions:
      pull-requests: write
      contents: read
    uses: DLS-soft2/shared-workflows/.github/workflows/java-tollgate.yaml@main
    with:
      service-specific-input: "your-service-name"
    secrets: inherit
```

`.github/workflows/build.yaml`:

```yaml
name: Build

on:
  push:
    branches:
      - main

jobs:
  integrate:
    permissions:
      pull-requests: write
      contents: write
      packages: write
    uses: DLS-soft2/shared-workflows/.github/workflows/java-build.yaml@main
    with:
      service-specific-input: "your-service-name"
    secrets: inherit
```

---

## Requirements

### Python services
- `pyproject.toml` with a `version` field following [SemVer](https://semver.org/)
- `Dockerfile` at the repo root

### Java services
- `pom.xml` with a `<version>` field following [SemVer](https://semver.org/)
- `Dockerfile` at the repo root
- Checkstyle plugin configured in `pom.xml` (optional but recommended)
