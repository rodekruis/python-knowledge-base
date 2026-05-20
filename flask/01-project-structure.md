# 01 — Project Structure

## Recommended Layout

For anything beyond a single-route prototype, use the **[application factory](README.md#glossary)** pattern with **[blueprints](README.md#glossary)**:

```
my-app/
├── src/my_app/
│   ├── __init__.py          # create_app() factory
│   ├── config.py            # settings classes
│   ├── extensions.py        # Flask extensions (CSRFProtect, etc.)
│   ├── models.py            # SQLAlchemy / domain models (if applicable)
│   ├── main/                # blueprint: primary routes
│   │   ├── __init__.py
│   │   ├── routes.py
│   │   └── forms.py         # WTForms form classes
│   ├── auth/                # blueprint: authentication
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── services/            # business logic (no Flask imports)
│   │   └── batch_service.py
│   ├── clients/             # external API wrappers
│   │   └── api_121.py
│   ├── templates/
│   │   ├── base.html        # shared layout
│   │   ├── main/
│   │   │   ├── index.html
│   │   │   └── success.html
│   │   └── errors/
│   │       ├── 404.html
│   │       └── 500.html
│   └── static/
│       ├── css/
│       ├── js/
│       └── img/
├── tests/
│   ├── conftest.py
│   ├── test_routes.py
│   └── test_services.py
├── Dockerfile
├── wsgi.py                  # Gunicorn entry point
├── pyproject.toml
├── uv.lock
└── README.md
```

## When a Flat Structure Is OK

For single-purpose internal tools with ≤3 routes and no complex business logic:

```
my-app/
├── app.py                   # Flask app + routes
├── utils.py                 # helper functions
├── templates/
│   ├── index.html
│   ├── success.html
│   └── error.html
├── static/
│   ├── style.css
│   └── img/
├── wsgi.py
├── requirements.txt
├── .flaskenv
├── example.env
└── README.md
```

This is acceptable for quick prototypes, but **graduate to blueprints** the moment you add a fourth route or the `app.py` exceeds ~150 lines.

## Key Principles

### Always use an application factory

Even for small apps, `create_app()` makes testing and configuration dramatically easier:

```python
# src/my_app/__init__.py
from flask import Flask
from .config import Config
from .extensions import csrf

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    # Initialize extensions
    csrf.init_app(app)

    # Register blueprints
    from .main.routes import main_bp
    app.register_blueprint(main_bp)

    # Register error handlers
    from .errors import register_error_handlers
    register_error_handlers(app)

    return app
```

❌ Bad — global `app` object with scattered config:
```python
# app.py
from flask import Flask
app = Flask(__name__)
app.secret_key = os.getenv("SECRETKEY")  # Module-level, untestable
csrf = CSRFProtect(app)
# ... 150 lines of routes mixed with config ...
```

✅ Good — factory with deferred initialization:
```python
# extensions.py
from flask_wtf.csrf import CSRFProtect
csrf = CSRFProtect()  # Not bound to app yet

# __init__.py
def create_app():
    app = Flask(__name__)
    csrf.init_app(app)  # Bind at factory time
    return app
```

### Separate business logic from route handlers

Route handlers should be thin — validate input, call a service, return a response. Never put business logic (API calls, data transformations, complex conditionals) directly in route functions.

❌ Bad — route handler doing everything:
```python
@app.route("/", methods=["POST"])
def index():
    cookie = login121()
    regs = getAllRegistrations(cookie, project)
    filtered = getFilteredRegistrations(regs, gnlabel, dsdiv_name, district_name)
    # ... 30 more lines of orchestration ...
```

✅ Good — route handler delegates to service:
```python
@main_bp.route("/", methods=["POST"])
def index():
    form = BatchForm()
    if form.validate_on_submit():
        try:
            result = batch_service.create_batches(form.data)
            session["batch_names"] = result.batch_names
            return redirect(url_for("main.success"))
        except ServiceError as e:
            flash(str(e), "error")
            return redirect(url_for("main.index"))
    return render_template("main/index.html", form=form)
```

### Keep `templates/` organized by blueprint

As the app grows, mirror the blueprint structure in `templates/`:

```
templates/
├── base.html          # {% block content %}{% endblock %}
├── main/
│   ├── index.html     # {% extends "base.html" %}
│   └── success.html
├── auth/
│   └── login.html
└── errors/
    ├── 404.html
    └── 500.html
```

### Static files in subfolders

```
static/
├── css/style.css
├── js/main.js
└── img/
    ├── favicon.ico
    ├── logo.png
    └── ...
```
