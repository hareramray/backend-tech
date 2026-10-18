# Django forms, templates, and the admin

## Purpose and prerequisites

Django forms parse and validate submitted data, templates render representations, and the admin provides an internal management interface. Each is useful, but none automatically establishes tenant authorization.

Prerequisites: the [Django 5.2 capstone](../09-capstone/01-task-service.md), [authentication](../03-security/02-authentication-passwords-and-sessions.md), and [tenant authorization](../03-security/04-authorization-and-multitenancy.md). Work in a disposable capstone copy. The browser views here are separate from its local JSON demo routes: use real Django login sessions and keep CSRF middleware enabled. Configure a login view and templates before using `login_required` in a browser.

## Bound forms and ModelForms

An unbound form has no submitted data and normally renders an initial page. A bound form contains submitted data and can report validation errors. Passing an empty dictionary still creates a bound form. Use `request.POST if request.method == "POST" else None`; `request.POST or None` incorrectly turns an empty POST into an unbound form. Access `cleaned_data` only after validation, and use its typed values rather than the original strings. [Django forms guide](https://docs.djangoproject.com/en/5.2/topics/forms/).

A plain `Form` describes input independently of a model, useful for search or multi-model workflows. A `ModelForm` derives fields and participates in model validation. Always specify an explicit field allowlist; `fields = "__all__"` can expose a newly added privileged field accidentally. Fields excluded from the form are not automatically validated as if supplied, and database races still require constraints. [Django ModelForms](https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/).

Create `tasks/forms.py`:

```python
from django import forms
from .models import Task

class TaskCreateForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ["title"]

    def clean_title(self):
        title = self.cleaned_data["title"].strip()
        if not title:
            raise forms.ValidationError("Enter a task title.")
        try:
            title.encode("utf-8")
        except UnicodeEncodeError as exc:
            raise forms.ValidationError("Enter valid Unicode text.") from exc
        return title
```

`clean_<field>()` validates one field after field conversion. Override `clean()` for relationships between fields, call `super().clean()`, and handle fields absent because earlier validation failed. In a ModelForm, forgetting the parent call can also interfere with validation behavior.

`save(commit=False)` returns an unsaved model so the server can assign trusted fields. It is not an authorization hook by itself. If a form contains many-to-many fields, save the instance and then call `save_m2m()` as appropriate. Do not assume `model.save()` calls `full_clean()` automatically.

## Explicit membership extension

Add this optional model to `tasks/models.py`, alongside the existing `Tenant` and `Task` models:

```python
from django.conf import settings

class Membership(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    role = models.CharField(
        max_length=10,
        choices=[("viewer", "Viewer"), ("editor", "Editor"), ("admin", "Admin")],
    )
    is_active = models.BooleanField(default=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["tenant", "user"], name="membership_tenant_user_uq"
            )
        ]
```

Run `python manage.py makemigrations tasks` and `python manage.py migrate`. Create two test users and two tenant memberships through a trusted shell or an explicitly secured enrollment workflow. Do not let a public form assign its own membership role.

For a brand-new project, decide whether to use a custom user model before initial migrations. If choosing one, define it early and configure `AUTH_USER_MODEL`; switching a populated project later is a data and migration project. Reusable models should reference `settings.AUTH_USER_MODEL`, and runtime code can use `get_user_model()`. [Django custom authentication guide](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/).

## A protected creation view

Create `tasks/browser_views.py`:

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.db import transaction
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_http_methods
from .forms import TaskCreateForm
from .models import Membership

@login_required
@require_http_methods(["GET", "POST"])
def task_create(request, tenant_id):
    allowed = Membership.objects.filter(
        tenant_id=tenant_id, user=request.user,
        is_active=True, role__in=["editor", "admin"],
    )
    get_object_or_404(allowed)
    form = TaskCreateForm(request.POST if request.method == "POST" else None)
    if request.method == "POST" and form.is_valid():
        with transaction.atomic():
            membership = get_object_or_404(allowed.select_for_update())
            task = form.save(commit=False)
            task.tenant_id = membership.tenant_id
            task.status = "todo"
            task.save()
        messages.success(request, "Task created.")
        return redirect("task-create", tenant_id=tenant_id)
    return render(request, "tasks/task_form.html", {"form": form})
```

Register `path("tenants/<uuid:tenant_id>/tasks/new/", task_create, name="task-create")` in your URL configuration, importing the view. The tenant in the URL is only a requested scope; membership authorizes it. Locking and rechecking the membership serializes this write against concurrent updates or deletion of that membership row. More complex permission stores need corresponding coordination.

The redirect after success prevents accidental resubmission on ordinary refresh. It does not guarantee idempotency after ambiguous network failures. If a database constraint can still reject valid-looking input, catch the expected `IntegrityError` outside the `atomic()` block, translate the known case into a form error, and let unexpected errors be investigated.

## Templates: presentation without hidden authority

Create `tasks/templates/tasks/base.html`:

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>{% block title %}Tasks{% endblock %}</title></head>
  <body>
    <main>
      {% for message in messages %}<p>{{ message }}</p>{% endfor %}
      {% block content %}{% endblock %}
    </main>
  </body>
</html>
```

Create `tasks/templates/tasks/task_form.html`:

```html
{% extends "tasks/base.html" %}
{% block title %}Create task{% endblock %}
{% block content %}
  <h1>Create task</h1>
  <form method="post">
    {% csrf_token %}
    {{ form.non_field_errors }}
    {{ form.title.errors }}
    {{ form.title.label_tag }} {{ form.title }}
    <button type="submit">Create task</button>
  </form>
{% endblock %}
```

Template inheritance centralizes layout. Default HTML escaping helps when displaying untrusted text, but avoid `safe` or `mark_safe()` on task titles. JavaScript, URLs, and other contexts need appropriate handling; HTML escaping is not a universal encoder. Use `json_script` for safely embedding structured data rather than interpolating JSON into executable script.

Keep database selection and authorization in views or selectors. Templates can trigger lazy queryset evaluation and related-object lookups, so pass bounded, appropriately prefetched data. Hiding a button from a viewer is a presentation choice; the mutation endpoint must still enforce permission.

## A deliberately read-only tenant admin

For internal staff inspection, register a constrained admin in `tasks/admin.py`:

```python
from django.contrib import admin
from .models import Membership, Task

@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = ["id", "tenant_id", "title", "status", "version"]
    search_fields = ["title"]
    list_filter = ["status"]
    actions = None

    def tenant_ids(self, request):
        return Membership.objects.filter(
            user=request.user, is_active=True, role="admin"
        ).values_list("tenant_id", flat=True)

    def get_queryset(self, request):
        return super().get_queryset(request).filter(
            tenant_id__in=self.tenant_ids(request)
        )

    def has_view_permission(self, request, obj=None):
        allowed = super().has_view_permission(request, obj)
        if obj is None:
            return allowed and self.tenant_ids(request).exists()
        return allowed and self.tenant_ids(request).filter(
            tenant_id=obj.tenant_id
        ).exists()

    def has_add_permission(self, request):
        return False

    def has_change_permission(self, request, obj=None):
        return False

    def has_delete_permission(self, request, obj=None):
        return False
```

Staff status allows entry to the admin, while model permission and active tenant-admin membership control these records. This example intentionally grants no automatic cross-tenant superuser bypass. Define any platform emergency-access workflow separately. Avoid global tenant filter dropdowns or relation choices that reveal other tenant names.

For editable admin extensions, enforce tenant scope in object permissions, form field querysets, save hooks, inline relations, autocomplete, and actions. A filtered changelist alone is insufficient. Bulk `QuerySet.update()` bypasses per-instance `save()` and related hooks; operations requiring domain rules, versions, events, or audit records should call an explicit service with bounded transaction and permission semantics. [Django admin reference](https://docs.djangoproject.com/en/5.2/ref/contrib/admin/).

## Lab, exercises, and completion checklist

1. Submit a valid title, whitespace-only title, missing title, and overlong title. Check both form errors and database changes.
2. Add a submitted `tenant` or `status` field. Confirm the allowlisted form and server assignment prevent changing either.
3. Submit to another tenant's URL. Expect no creation even if the title is valid.
4. Remove the CSRF token using a test client with `enforce_csrf_checks=True`. Expect rejection.
5. Display a title containing HTML characters and verify it remains text.
6. Give a staff user view permission and tenant A membership. Verify tenant B is absent from lists, direct object URLs, filters, and search.
7. Exercise membership revocation while a form is open, then submit it.

- [ ] Form fields are explicit and validated server-side.
- [ ] Trusted ownership fields are assigned after authorization.
- [ ] Browser mutations retain CSRF protection.
- [ ] Templates preserve escaping and avoid hidden query work.
- [ ] Admin lists, object views, relations, and actions respect tenant policy.
- [ ] Editing and bulk workflows preserve concurrency and audit rules.

Related: [web security](../03-security/05-web-security-and-input-attacks.md) and [modular architecture](../08-architecture/01-modular-monoliths-and-domain-design.md).
