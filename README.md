# GitLab Policy Project — Layered CI (Sonar + TruffleHog + Trivy)

This repository is a **Security Policy Project** scaffold that injects modular layers into pipelines via **Pipeline Execution Policies (PEP)**:
- **SonarQube** (multi-language with Node.js and Java support)
- **TruffleHog** (secrets scan on MRs by default)
- **Trivy** (container config + fs scan when container-based repos are detected)

All logic is split across files for manageability. Secrets are kept out of YAML (use **group variables** or **File-type variables**).

## How it works
1. A group-level **Pipeline Execution Policy** uses `inject_policy` to include a single aggregator from *this* repo: `.policy/sonar/sonar-layer.yml`.
2. That aggregator includes modular CI files:
   - `detect.yml` — detects languages and container indicators and emits a **dotenv** artifact.
   - `build-java-*.yml` — compiles Java only when Maven/Gradle is present, so Sonar can use bytecode.
   - `build-node.yml` — installs deps with npm/pnpm/yarn and produces `coverage/lcov.info` when possible.
   - `scan.yml` — runs Sonar and automatically includes Dockerfile patterns and JS coverage if present.
   - `../trufflehog/scan.yml` — secrets scan (runs on merge requests by default).
   - `../trivy/scan.yml` — config + fs scan (runs only if the repo is detected as container-based).

## One-time setup
1. Push this repo to your **Security Policy Project** (`main` branch).
2. In **Group → Security & Compliance → Policies**, create a **Pipeline execution policy** and set it to inject:
   ```yaml
   type: pipeline_execution_policy
   name: Layered CI: Sonar + Security
   enabled: true
   rules:
     - type: pipeline
       branches: ["*"]
   pipeline_config_strategy: inject_policy
   content:
     include:
       - project: your-group/your-policy-project
         ref: main
         file: .policy/sonar/sonar-layer.yml
   ```
3. At the group level, add variables:
   - `SONARQUBE_TOKEN` (masked, protected)
   - `SONAR_HOST_URL` (e.g., `https://sonar.example.com`)
   - *(Optional)* `SONAR_ENV_FILE` as a **File-type** variable with:
     ```
     SONAR_HOST_URL=https://sonar.example.com
     SONARQUBE_TOKEN=REPLACE_ME
     SONAR_PROJECT_KEY=$CI_PROJECT_PATH_SLUG
     ```

## Useful toggles
- `POLICY_DISABLE_SONAR=1` (project/group) — disables the injected Sonar layer for that project.
- `TRUFFLEHOG_STRICT=1` — fail pipeline when TruffleHog finds secrets.
- `TRIVY_SEVERITY` — control which severities are reported (default: MEDIUM,HIGH,CRITICAL).

## Notes
- All injected jobs use the **`.pre`** stage to avoid clashing with custom stage lists.
- The detection job also sets `IS_CONTAINER_BASED` so Trivy only runs when appropriate.
