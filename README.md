# Python Knowledge Base

This repository collects all the knowledge, documentation, best practices, tips, templates and manuals developed by 510 for python applications.

## Contents

### [Shared](shared/)

Framework-agnostic tooling and standards that apply to **all** Python projects: dependency management (uv, pyproject.toml), CI/CD (GitHub Actions), code quality (ruff, type checking, pre-commit), logging & observability (Azure Monitor), and Docker best practices.

### [FastAPI](fastapi/)

Best practices for building FastAPI web APIs. Covers project structure, configuration, testing, and framework-specific deployment details.

### [Flask](flask/)

Best practices for building Flask web applications with server-rendered HTML (Jinja2 templates, forms, sessions). Covers the application factory, blueprints, WTForms, security headers, testing, and framework-specific deployment details.

### [ETL Pipelines](etl-pipelines/)

Best practices for building data pipelines that extract data from external sources, transform it, and load it into a target system (API, database, file store). Covers project structure, configuration, data types, error handling, testing, CLI design, deployment, and more.

### [Data Analysis](data-analysis/)

Best practices for building reproducible, well-structured **Jupyter notebook** workflows for data analysis, statistical calibration, and geospatial modeling. Covers notebook structure, conda environments, lazy data loading, memory management, visualization, code reuse, and more.

### AI Use Disclaimer

Parts of this knowledge base - including the consolidated `copilot-instructions.md` files - were drafted with AI assistance and then reviewed and edited by 510 engineers. AI tools here are meant to **augment** engineers, not replace their judgment: a human reviews, understands, and owns all output. For the principles we follow, see the [Responsible AI guide](https://responsibleai.guide/), of which 510 is a collaborator.

