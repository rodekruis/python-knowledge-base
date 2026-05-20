# 03 — Routes and Templates

## Blueprints

Group related routes into **[blueprints](README.md#glossary)**. Even a small app benefits from at least one blueprint:

```python
# src/my_app/main/routes.py
from flask import Blueprint, render_template, redirect, url_for, flash, session

main_bp = Blueprint("main", __name__, template_folder="../templates/main")

@main_bp.route("/", methods=["GET", "POST"])
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

@main_bp.route("/success")
def success():
    batch_names = session.get("batch_names", [])
    return render_template("main/success.html", batch_names=batch_names)
```

Register in the factory:
```python
def create_app(config_class=Config):
    app = Flask(__name__)
    from .main.routes import main_bp
    app.register_blueprint(main_bp)
    return app
```

## Route Handler Rules

### Keep handlers thin

A route handler should do **three things** and nothing more:

1. **Parse and validate** input (form data, query params)
2. **Call a service** function with plain Python arguments
3. **Return a response** (render template, redirect, error page)

❌ Bad — 40-line route function with API calls, data filtering, looping:
```python
@app.route("/", methods=["POST"])
def index():
    cookie = login121()
    if not cookie:
        return render_template("error.html", error="Invalid credentials")
    project = request.form.get("121programid")
    regs = getAllRegistrations(cookie, project)
    filtered = getFilteredRegistrations(regs, gnlabel, dsdiv_name, district_name)
    lastBatch = getLastBatchNumber(regs, district_name)
    batch_names = createBatchesRandomSampling(filtered, ...)
    session["batch_names"] = batch_names
    return redirect(url_for("success"))
```

✅ Good — handler as wiring:
```python
@main_bp.route("/", methods=["POST"])
def index():
    form = BatchForm()
    if not form.validate_on_submit():
        return render_template("main/index.html", form=form)

    try:
        result = batch_service.create_and_sample(
            username=form.username.data,
            password=form.password.data,
            program_id=form.program_id.data,
            gn_label=form.gn_label.data,
            dsdiv=form.dsdiv.data,
            district=form.district.data,
            batch_size=form.batch_size.data,
            allow_smaller_last_batch=form.min_batchsize.data,
        )
        session["batch_names"] = result.batch_names
        session["program_id"] = form.program_id.data
        return redirect(url_for("main.success"))
    except AuthenticationError:
        flash("Invalid 121 credentials.", "error")
    except AuthorizationError as e:
        flash(str(e), "error")
    except ExternalServiceError as e:
        flash(f"121 API error: {e}", "error")

    return render_template("main/index.html", form=form)
```

## WTForms for Input Validation

Never validate form data manually with `request.form.get()`. Use **[Flask-WTF](README.md#glossary)**:

```python
# src/my_app/main/forms.py
from flask_wtf import FlaskForm
from wtforms import StringField, IntegerField, PasswordField, RadioField
from wtforms.validators import DataRequired, NumberRange, Email

class BatchForm(FlaskForm):
    username = StringField("121 Username", validators=[DataRequired(), Email()])
    password = PasswordField("121 Password", validators=[DataRequired()])
    program_id = IntegerField("Program ID", validators=[DataRequired(), NumberRange(min=1)])
    gn_label = StringField("GN Label", validators=[DataRequired()])
    dsdiv = StringField("DS Division", validators=[DataRequired()])
    district = StringField("District", validators=[DataRequired()])
    batch_size = IntegerField("Batch Size", validators=[DataRequired(), NumberRange(min=1)])
    min_batchsize = RadioField(
        "Allow smaller last batch?",
        choices=[("yes", "Yes"), ("no", "No")],
        validators=[DataRequired()],
    )
```

### Benefits over raw `request.form`

- **Type coercion** — `IntegerField` gives you an `int`, not a string
- **Validation** — `DataRequired()`, `NumberRange()`, `Email()` run server-side automatically
- **CSRF** — `FlaskForm` includes CSRF token handling by default
- **Error messages** — each field carries error messages to display in templates
- **Re-rendering** — failed validation re-renders the form with user input preserved

## Jinja2 Templates

**[Jinja2](README.md#glossary)** is Flask's default templating engine with auto-escaping, inheritance, and macros.

### Use template inheritance

Every template should extend a base layout:

```html
{# templates/base.html #}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>{% block title %}App{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    <link rel="shortcut icon" href="{{ url_for('static', filename='img/favicon.ico') }}">
</head>
<body>
    {% include "partials/_navbar.html" %}

    <main class="container">
        {# Flash messages #}
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }}">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}

        {% block content %}{% endblock %}
    </main>

    {% block scripts %}{% endblock %}
</body>
</html>
```

```html
{# templates/main/index.html #}
{% extends "base.html" %}

{% block title %}Batch Creation{% endblock %}

{% block content %}
<form method="POST">
    {{ form.hidden_tag() }}
    {# form.hidden_tag() renders CSRF token + any hidden fields #}

    <h2>121 Sri Lanka — Batch Creation</h2>
    {{ render_field(form.username) }}
    {{ render_field(form.password) }}
    {# ... #}
    <button type="submit">Create Batches</button>
</form>
{% endblock %}
```

### Macros for form fields

Avoid repeating HTML for each field. Define a macro:

```html
{# templates/macros/forms.html #}
{% macro render_field(field) %}
<div class="form-group {% if field.errors %}has-error{% endif %}">
    <label for="{{ field.id }}">{{ field.label.text }}</label>
    {{ field(class="form-control") }}
    {% for error in field.errors %}
        <span class="error-text">{{ error }}</span>
    {% endfor %}
</div>
{% endmacro %}
```

Import and use:
```html
{% from "macros/forms.html" import render_field %}
{{ render_field(form.username) }}
```

### Auto-escape is on by default

Jinja2 in Flask auto-escapes all variables in HTML templates. **Never** disable it:
```html
{# ✅ Safe — auto-escaped #}
<p>{{ user_input }}</p>

{# ❌ DANGEROUS — disables escaping, XSS risk #}
<p>{{ user_input | safe }}</p>
```

Only use `| safe` when you are rendering HTML you generated yourself (e.g., Markdown → HTML with a trusted library).

## CSRF Protection

Flask-WTF provides **[CSRF](README.md#glossary)** protection. Ensure it's active everywhere:

```python
# extensions.py
from flask_wtf.csrf import CSRFProtect
csrf = CSRFProtect()
```

In templates, **always** include the CSRF token:
```html
<form method="POST">
    {{ form.hidden_tag() }}  {# If using WTForms #}
    {# OR manually: #}
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
</form>
```

For AJAX / htmx requests, include the token in headers:
```javascript
document.body.addEventListener('htmx:configRequest', (evt) => {
    const token = document.querySelector('meta[name="csrf-token"]').content;
    evt.detail.headers['X-CSRFToken'] = token;
});
```

## PRG Pattern (Post/Redirect/Get)

Use the **[PRG pattern](README.md#glossary)** — always redirect after a successful POST to prevent duplicate submissions:

```python
@main_bp.route("/", methods=["GET", "POST"])
def index():
    form = BatchForm()
    if form.validate_on_submit():
        # ... process ...
        flash("Batches created successfully!", "success")
        return redirect(url_for("main.success"))  # ← redirect, not render
    return render_template("main/index.html", form=form)
```
