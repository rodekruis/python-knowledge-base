# CI/CD with GitHub Actions

## Workflow Structure

Every project needs at minimum two jobs: **test** and **deploy**.

```
push to main → [lint-and-test] → [deploy]
pull request  → [lint-and-test]
```

---

## Lint & Test Job (Universal)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync

      - name: Lint
        run: uv run ruff check .

      - name: Format check
        run: uv run ruff format --check .

      - name: Type check
        run: uv run ty check       # or: uv run mypy src/

      - name: Test
        run: uv run pytest --cov=src --cov-report=term-missing
```

### Action Versions

Keep actions pinned to major versions:

| Action | Version | Purpose |
|--------|---------|---------|
| `actions/checkout` | `v4` | Clone repo |
| `astral-sh/setup-uv` | `v4` | Install uv |
| `docker/login-action` | `v3` | Registry auth |
| `docker/build-push-action` | `v6` | Build & push image |
| `azure/login` | `v2` | Azure OIDC login |
| `azure/webapps-deploy` | `v3` | Deploy to App Service |

---

## Deploy Strategies

Choose based on your framework and hosting model:

### Option A: Zip Deploy (Azure App Service without Docker)

Best for Flask apps or simple deployments:

```yaml
  deploy:
    needs: lint-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v4
      - run: uv python install 3.12
      - run: uv sync --no-dev --frozen

      - name: Package application
        run: zip -r app.zip . -x '.git/*' 'tests/*' '.github/*' '*.md'

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          package: app.zip
```

### Option B: Docker Deploy (Container-based)

Best for FastAPI apps or complex deployments:

```yaml
  deploy:
    needs: lint-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          images: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## Azure OIDC Authentication

Use OpenID Connect (OIDC) instead of long-lived credentials:

1. Create an App Registration in Azure AD
2. Add a federated credential for your GitHub repo
3. Configure the following secrets/variables:

| Type | Name | Source |
|------|------|--------|
| Secret | `AZURE_CLIENT_ID` | App Registration |
| Secret | `AZURE_TENANT_ID` | Azure AD |
| Secret | `AZURE_SUBSCRIPTION_ID` | Azure Portal |
| Variable | `AZURE_WEBAPP_NAME` | App Service name |

Add `permissions: id-token: write` to the deploy job if using OIDC.

---

## Environment Protection

Configure GitHub Environments for deployment safety:

- **production** — require manual approval for first deploy, then auto-deploy on `main`
- Use environment-scoped secrets (not repo-level) for production credentials
- Enable "required reviewers" for production environment

---

## Branch Protection Rules

For `main` branch:

- ✅ Require pull request before merging
- ✅ Require status checks to pass (`lint-and-test`)
- ✅ Require branches to be up to date
- ❌ Do not allow force pushes

---

## Release Promotion (Optional)

For projects that need staged rollout:

```yaml
on:
  release:
    types: [published]

jobs:
  promote:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - run: |
          docker pull ghcr.io/${{ github.repository }}:${{ github.event.release.tag_name }}
          docker tag ghcr.io/${{ github.repository }}:${{ github.event.release.tag_name }} \
                     ghcr.io/${{ github.repository }}:production
          docker push ghcr.io/${{ github.repository }}:production
```

Tag a GitHub Release → image gets promoted → production webhook pulls new image.
