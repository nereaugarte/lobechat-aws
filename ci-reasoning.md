# CI Reasoning

## Evidence

[placeholder — I will embed the screenshot here after the Actions run]

Actions run URL: [TO BE FILLED]
Commit SHA: [TO BE FILLED]

---

## Part A — Why what I built matters (repository-specific)

### 1. hadolint — `dockerfiles/mcphub.Dockerfile` line 1: unpinned `FROM`

```
FROM samanhappy/mcphub:latest   # line 1
```

`hadolint` flags this `FROM samanhappy/mcphub:latest` as DL3007 (avoid `:latest`). Because `docker-compose.production.yml` (lines 76–91) builds mcphub **locally** from this Dockerfile rather than pulling a pre-built image, the unpinned risk lives entirely in the **Dockerfile `FROM`**, not in the compose `image:` field. On every `docker compose build`, Docker resolves `:latest` at that moment — so two builds a day apart may produce different images with no indication in git. If the upstream `samanhappy/mcphub` image is compromised or silently updated, the next local build silently ships the change. Pinning to a digest (e.g. `samanhappy/mcphub@sha256:...`) makes the build reproducible and auditable.

A second issue in the same file: **line 5 is `USER root`** and is never followed by a `USER` switch back to a non-root account. The `RUN` on line 7 installs `docker.io` and `gcc` into this runtime image — the Docker socket is also bind-mounted at runtime (`docker-compose.production.yml` line 91: `/var/run/docker.sock:/var/run/docker.sock`). A process running as root inside the container with access to the Docker socket has effective host-root privileges. `trivy config` and `hadolint` both flag the missing non-root user drop.

### 2. hadolint / trivy config — `dockerfiles/sandbox.Dockerfile`: unpinned tool downloads and NOPASSWD sudo

Three separate supply-chain risks in the same file:

- **Line 44** — kubectl downloaded from `https://dl.k8s.io/release/${KVER}/bin/linux/${ARCH}/kubectl` where `$KVER` resolves to the live "latest stable" string at build time. No checksum is verified. A DNS-hijack or CDN compromise could swap the binary.
- **Lines 49–50** — eksctl fetched from `"https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_${ARCH}.tar.gz"`. Again `latest`, no pin, no `sha256sum` check.
- **Lines 62** — zellij downloaded the same way: `zellij-$Z.tar.gz` from `releases/latest`.
- **Lines 21–22** — the container user `oriol` is granted `NOPASSWD:ALL` sudo (`echo 'oriol ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/oriol`). If an MCP tool call executing inside this sandbox is tricked into running an arbitrary command, full root access is one `sudo` away.

Together these mean the sandbox image is not reproducible and its runtime privilege boundary is minimal.

### 3. trivy config / docker compose config — `docker-compose.production.yml` line 16: `sslmode=disable`

```yaml
- dataSourceName=user=postgres password=${POSTGRES_PASSWORD:-postgres} host=postgres port=5432 sslmode=disable dbname=casdoor
```

Casdoor's Postgres connection string explicitly disables TLS (`sslmode=disable`). Traffic between the Casdoor container and the `shared-postgres` container flows unencrypted. While both containers sit on the same Docker bridge network (lower immediate risk), this is a misconfiguration that trivy's `config` mode flags: any network-layer inspection (e.g. a compromised container on the same host) can read authentication tokens, session data, and user records in plaintext. Additionally, `lobe-chat`'s `DATABASE_URL` at line 34 (`postgresql://postgres:...@postgres:5432/lobechat`) specifies no TLS parameters at all, inheriting the driver's default — also unencrypted.

### 4. gitleaks — honest assessment: no committed secrets found on a clean tree

The `.gitignore` already excludes the files most likely to carry real secrets:
- line 5: `.env`
- line 10: `aws_credentials.yaml`
- line 11: `*.pem`
- line 6: `db/flyway/.env.flyway`

On a clean working tree with no accidentally committed secrets, `gitleaks` will surface no findings — and that is the correct outcome. The value of running it in CI is not that it will find something today, but that it acts as a continuous tripwire: if a future commit accidentally includes a real API key, a Postgres password, or an AWS credential, the gate catches it before the commit reaches the remote. It is set to `continue-on-error: true` so an unexpected finding surfaces visibly without immediately blocking the team — the finding should then be triaged and the secret rotated.

### 5. yamllint + actionlint — `ci.yml` validates itself

`yamllint` parses `.github/workflows/ci.yml` as YAML: it catches indentation errors, duplicate keys, and tab/space mixing that would cause GitHub's parser to silently misinterpret the workflow. `actionlint` goes further and understands GitHub Actions semantics: it checks that `uses:` references exist, that `with:` keys match the action's inputs, and that expression syntax like `${{ }}` is well-formed. A workflow file with a YAML parse error or an invalid `uses:` reference simply does not run on GitHub — it shows a red "invalid workflow" banner with no job output. Running both linters in CI means a broken workflow is caught in the PR that introduces it, not discovered after the fact when the pipeline silently goes dark.

---

### Why the pipeline is build-free

The `docker-compose.production.yml` stack (8 services) cannot run on a standard GitHub-hosted runner. The exam spec references a `vllm` service (present in some configurations of this stack) that requires an NVIDIA GPU and a `start_period: 300s` health-check grace period — standard runners have no GPU and a 6-hour job timeout is shared across all steps. Even without vllm, standing up postgres, minio, casdoor, mcphub, qdrant, hayhooks, hayhooks-mcp, and lobe-chat requires substantial memory, inter-service health checks, and several minutes of startup before any test could run. A build-free pipeline of static gates completes in under two minutes on any runner and catches the most impactful classes of defect (supply-chain, secrets, misconfigurations, schema errors) without any of that complexity.

### Why `tests/` is excluded

The exam brief (and the `pyproject.toml` dev toolchain) references integration tests that import `openai` and `httpx` and hit live running endpoints such as a vLLM `/health` route. These are live-stack integration tests, not unit tests: they require the full service graph to be running and healthy before they can execute. Without a running stack they fail immediately at connection. They belong in a later pipeline stage — after a successful deploy to an ephemeral environment — not in a static-analysis CI job.

### Why the interpolation fix is safe

`docker compose -f docker-compose.production.yml config -q` interpolates every `${VAR}` in the compose file. Without values for variables like `NEXT_AUTH_SECRET`, `AUTH_CASDOOR_ID`, `AUTH_CASDOOR_SECRET`, `KEY_VAULTS_SECRET`, and `OPENROUTER_API_KEY`, Compose aborts with "variable is not set". The fix is `cp .env.example .env` inside the CI job before running `config`. This is safe because:

1. `.env.example` contains only documented placeholder strings (e.g. `KEY_VAULTS_SECRET=Y2hhbmdlLW1lLWtleS12YXVsdHMtc2VjcmV0LTMyYg==` — the base64 of `"change-me-key-vaults-secret-32b"`), not real credentials.
2. `.gitignore` line 5 excludes `.env`, so the copied file is never staged or committed.
3. The `config -q` command only validates schema and interpolation — it starts nothing, connects to nothing, and the values themselves are never used at runtime.

---

## Part B — What's missing for a real production CI/CD pipeline

This workflow implements **Continuous Integration**: a set of static quality and security gates that run on every push and pull request. It stops short of **Continuous Delivery/Deployment** — it never builds an artifact, pushes to a registry, or touches the running EC2 instance. The following items are required to close that gap for this specific system.

### 1. Build, push, sign, and SBOM the locally-built images to ECR

`dockerfiles/mcphub.Dockerfile` and `dockerfiles/sandbox.Dockerfile` are built locally with `docker compose build` — there is no registry push, no image digest recorded, no SBOM generated. `docker-compose.production.yml` line 80 shows `image: lobechat-aws-mcphub:latest` as the result of that local build, but `:latest` is not an immutable reference. A real pipeline would build both Dockerfiles in CI, push them to ECR with a content-addressed digest tag (e.g. `sha256:...`), sign them with Cosign, generate an SBOM, and update the compose `image:` field to the pinned digest before deploying — so every deploy is bit-for-bit reproducible.

### 2. GitHub OIDC federation instead of static AWS credentials

The previous `.github/workflows/ci.yml` used `aws-actions/configure-aws-credentials@v4` with `${{ secrets.AWS_ACCESS_KEY_ID }}` and `${{ secrets.AWS_SECRET_ACCESS_KEY }}` — long-lived static keys stored as GitHub secrets. `.env.example` lines 61–63 show the same pattern as commented deployment-time placeholders (`# AWS_ACCESS_KEY_ID=your-access-key`, `# AWS_SECRET_ACCESS_KEY=your-secret-key`, `# AWS_SESSION_TOKEN=your-session-token`). A real pipeline replaces both with GitHub OIDC: the runner assumes an IAM role via a short-lived token scoped to the specific repository and branch, with no standing credentials anywhere — not in GitHub secrets, not in `.env` on the host. This removes the entire class of "leaked CI secret" incidents; a compromised runner token expires in minutes rather than persisting until manually rotated.

### 3. Secrets injection from SSM Parameter Store / Secrets Manager at deploy time

`docker-compose.production.yml` passes roughly 15 secrets as plaintext environment variables (lines 13–57): `NEXT_AUTH_SECRET`, `AUTH_CASDOOR_ID`, `AUTH_CASDOOR_SECRET`, `KEY_VAULTS_SECRET`, `OPENROUTER_API_KEY`, `MINIO_ROOT_PASSWORD`, `POSTGRES_PASSWORD`, and others. On the EC2 host these come from a `.env` file that must be provisioned manually. A real pipeline fetches each secret from AWS SSM Parameter Store or Secrets Manager at deploy time (via `aws ssm get-parameter --with-decryption`), writes them into the runtime environment only for the duration of the compose run, and never stores them in git, in the runner environment, or in a static file on disk. This eliminates the `.env` file as a secret-at-rest on the host.

### 4. Database migration stage with guarded `clean` command

`db/flyway/provision.sh` manages three databases (lobechat, casdoor, litellm) via versioned Flyway migrations. The script accepts a `clean` subcommand (line 8 of the script header comment) that drops all rows and resets migration history — irreversible data destruction. A real pipeline adds a migration job between the build/push stage and the deploy stage, running `provision.sh migrate` (not `clean`) against the target database, and puts the `clean` command behind a manual approval environment in GitHub Actions so it can never be triggered by an automated push.

### 5. Environment promotion dev → stage → prod with manual approval

There is currently no promotion flow: every push to the main branch would, in theory, go directly to the single EC2 instance. A real pipeline defines at least two GitHub Actions environments (`staging`, `production`) with protection rules: required reviewers for production, deployment branch restrictions, and environment-scoped secrets. A deploy to production only proceeds after a staging deploy has passed its smoke tests and a human has clicked "Approve". This matches the final-project Q2 promotion requirement and prevents a broken push from reaching production users.

### 6. Deploy mechanism to EC2 that keeps port 47000 behind the reverse proxy

There is no deploy step in CI at all. The current workflow of `docker compose pull && docker compose up -d` on the EC2 instance is done manually over SSH. A real pipeline automates this via AWS SSM Run Command (no open SSH port required) or a self-hosted runner on the EC2 instance, running `docker compose -f docker-compose.production.yml pull && docker compose -f docker-compose.production.yml up -d --remove-orphans`. Critically, the Caddy reverse proxy (`Caddyfile` in the repo root) terminates HTTPS and proxies to `localhost:47000` — port 47000 must never be opened in the EC2 security group, and the deploy step must validate this invariant.

### 7. Post-deploy smoke tests and running integration tests against an ephemeral environment

The integration tests referenced by the exam brief hit live endpoints (`/health`, the OpenRouter proxy, the Casdoor auth flow). These cannot run without the stack running. A real pipeline spins up an ephemeral environment (e.g. a second EC2 instance or a Docker-in-Docker compose run with a GPU-less model stub replacing vllm), deploys the stack, waits for all healthchecks to pass, runs the integration suite, and tears the environment down. Only if those tests pass does the pipeline promote to production. This is the "right" home for tests that import `openai` and `httpx`.

### 8. Automated rollback — patch the monkeypatch out of the deploy unit

`docker-compose.production.yml` line 29 bind-mounts `./patches/route.js` (a 91,521-line compiled Next.js bundle) into the running lobe-chat container:

```yaml
- ./patches/route.js:/app/.next/server/app/(backend)/trpc/tools/[trpc]/route.js:ro
```

This means the deploy unit is an unpinned image **plus** a committed 3 MB binary blob that overwrites a file inside the container at runtime. If the upstream `lobehub/lobe-chat-database` image changes its internal path structure, the bind-mount silently stops applying, with no error. A real pipeline forks the image, bakes the patch in with a `COPY` instruction in a thin wrapper Dockerfile, pushes that image to ECR with a pinned digest, and removes the bind-mount entirely — so rollback is simply re-pinning the compose `image:` field to the previous digest.

---

### Prioritisation: the single highest-value next step

**GitHub OIDC federation + ECR image push** is the single highest-value next step for this system.

It resolves two critical risks in one pipeline stage: the standing-credentials risk (the old `ci.yml` used long-lived `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` secrets, which if leaked grant persistent AWS access) and the `:latest` non-reproducibility risk (both locally-built images resolve to whatever was last built on the EC2 host, with no audit trail). Replacing static keys with OIDC and pushing signed, digest-tagged images to ECR means every deploy is traceable to a specific CI run, a specific git commit, and a specific image digest — the three pillars of a reproducible and auditable delivery pipeline. Everything else (migration stages, promotion gates, smoke tests) depends on having an immutable, signed artifact to promote in the first place.
