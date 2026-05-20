# 09 — CI/CD

> **Shared foundation:** See [../shared/ci-cd.md](../shared/ci-cd.md) for the universal lint-and-test workflow, action versions, OIDC authentication, environment secrets, and branch protection rules.

This addendum covers **Flask-specific** deployment details only.

---

## Deploy Strategy: Zip Deploy to Azure App Service

Flask apps are typically deployed as zip artifacts (no Docker required for simple deployments):

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: lint-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: Production
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        run: uv python install 3.12

      - name: Install production dependencies
        run: uv sync --no-dev --frozen

      - name: Zip artifact
        run: zip release.zip ./* -r -x '.git/*' 'tests/*' '.github/*' '*.md' '.venv/*' '.env'

      - uses: azure/webapps-deploy@v3
        id: deploy-to-webapp
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE }}
          package: release.zip
```

### Excluded from zip

```bash
.git/*  tests/*  .github/*  *.md  .env  .flaskenv  venv/*  __pycache__/*
```

### Alternative: Docker deploy

For more complex Flask apps, use Docker instead (see [08-docker.md](08-docker.md) and the shared [Docker deploy job](../shared/ci-cd.md#option-b-docker-deploy-container-based)).

