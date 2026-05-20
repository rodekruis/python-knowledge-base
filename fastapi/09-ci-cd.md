# 09 — CI/CD

> **Shared foundation:** See [../shared/ci-cd.md](../shared/ci-cd.md) for the universal lint-and-test workflow, action versions, OIDC authentication, environment secrets, and branch protection rules.

This addendum covers **FastAPI-specific** deployment details only.

---

## Workflow File Structure

```
.github/workflows/
├── ci.yml                 # Lint + test on every push/PR (shared pattern)
├── build-deploy-dev.yml   # Deploy to dev on push to dev branch
└── build-deploy-prod.yml  # Deploy to production on push to main
```

---

## Deploy Strategy: Docker → Azure App Service

FastAPI apps are deployed as Docker containers:

```yaml
name: build and deploy

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ vars.REGISTRY_URL }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ vars.REGISTRY_URL }}/my-app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: production
      url: ${{ steps.deploy.outputs.webapp-url }}
    steps:
      - uses: azure/webapps-deploy@v3
        id: deploy
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          images: ${{ vars.REGISTRY_URL }}/my-app:${{ github.sha }}
```

### Docker Build Caching

Use GitHub Actions cache (`type=gha`) to dramatically speed up builds:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

---

## Release Promotion Flow

For mature apps with dev → staging → production:

1. **Push to `dev`** → builds image, deploys to dev environment
2. **Push to `main`** → builds image, deploys to staging
3. **Publish GitHub Release** → promotes the same image digest to production

This ensures the **exact same image artifact** moves through environments — no rebuild variation.

