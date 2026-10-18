# Django static assets, media, email and localization

Prerequisites: [course map](../README.md), [models and ORM](01-models-orm-and-querysets.md), the initialized [capstone](../09-capstone/01-task-service.md), and [file pipelines](../04-async/06-files-and-media-pipelines.md). These are optional capstone extensions. Run shell examples with `python manage.py shell`; run management commands from its project directory.

## Objectives

Deploy static assets independently of user uploads, manage FileField storage without assuming distributed transactions, send templated email through a testable boundary, and localize presentation while preserving stable API data.

## Static files and media have different lifecycles

Static files are application-owned CSS, JavaScript and images shipped with a release. User media is data created after deployment. collectstatic gathers static assets into the configured staticfiles storage; it does not back up or publish user uploads. Production still needs a static server or storage/CDN integration. [Django static deployment](https://docs.djangoproject.com/en/5.2/howto/static-files/deployment/)

Add these settings, keeping the existing BASE_DIR:

~~~python
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_ROOT = BASE_DIR / "private-media"
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.ManifestStaticFilesStorage",
    },
}
~~~

Create `tasks/static/tasks/tutorial.css` containing `body { font-family: sans-serif; }`. Run:

~~~powershell
python manage.py findstatic tasks/tutorial.css
python manage.py collectstatic --noinput
~~~

Expect findstatic to identify the source and collectstatic to create collected assets plus a manifest. Reference assets through `{% load static %}` and `{% static "tasks/tutorial.css" %}` in templates so storage can select the appropriate URL. Preserve assets needed by older application releases during rolling deployments. Do not place private-media under a public web-server root.

## Lab: FileField and its storage boundary

Add this extension to `tasks/models.py`, reusing its models import. The storage key ignores the user filename:

~~~python
from uuid import uuid4

def attachment_key(instance, filename):
    return f"quarantine/{instance.task.tenant_id}/{uuid4().hex}"

class TaskAttachment(models.Model):
    task = models.ForeignKey("tasks.Task", on_delete=models.CASCADE)
    file = models.FileField(upload_to=attachment_key, max_length=300)
    approved = models.BooleanField(default=False)
~~~

Run `python manage.py makemigrations tasks` and `python manage.py migrate`. Create the shared task using the [relational seed step](../02-data/01-relational-modeling-and-sql.md), then execute:

~~~python
from django.core.files.base import ContentFile
from tasks.models import Task, TaskAttachment

task = Task.objects.get(
    tenant_id="11111111-1111-4111-8111-111111111111",
    pk="22222222-2222-4222-8222-222222222222",
)
attachment = TaskAttachment(task=task)
attachment.file.save("example.txt", ContentFile(b"Learning attachment\n"))
print(attachment.pk, attachment.file.name)
assert attachment.file.storage.exists(attachment.file.name)
~~~

The database stores a storage-relative filename; the storage backend owns the bytes. FileField APIs abstract local and remote storage, so use file.open() and storage methods rather than assuming file.path exists on every backend. [Django file management](https://docs.djangoproject.com/en/5.2/topics/files/)

In a real upload view, select the task using an authorized tenant membership, enforce size/type rules and queue validation before approving the attachment. Download only approved attachments after checking the current user's access. A private storage key or signed URL is not a substitute for that authorization check.

Database rollback does not undo an already completed filesystem or object-storage write. Deleting a model also does not automatically remove its associated file. [FileField lifecycle reference](https://docs.djangoproject.com/en/5.2/ref/models/fields/#django.db.models.fields.files.FieldFile.delete) Persist cleanup intent in an outbox and let an idempotent worker remove bytes after the database commit. on_commit can schedule best-effort cleanup but has a crash window. Retain an orphan-reconciliation job and avoid deleting a blob still referenced elsewhere.

For an isolated cleanup exercise, capture the storage and name, delete the row, verify the file remains, then delete that exact file:

~~~python
storage, name = attachment.file.storage, attachment.file.name
attachment.delete()
assert storage.exists(name)
storage.delete(name)
assert not storage.exists(name)
~~~

## Optional private object storage

Install `python -m pip install "django-storages[s3]"` in the activated environment when adding S3-compatible storage. Provision a private bucket and scoped credentials separately. Replace only the default alias:

~~~python
import os

STORAGES["default"] = {
    "BACKEND": "storages.backends.s3.S3Storage",
    "OPTIONS": {
        "bucket_name": os.environ["MEDIA_BUCKET"],
        "region_name": os.environ["AWS_REGION"],
        "default_acl": None,
        "querystring_auth": True,
        "file_overwrite": False,
    },
}
~~~

Credentials should come from the deployment's identity/credential provider. These options do not create a bucket or override a dangerously public bucket policy. Issue signed downloads only after authorization, and bind scanning to the object version you later serve. [django-storages S3 settings](https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html)

## Lab: templated email without sending externally

Use the console backend for development:

~~~python
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
DEFAULT_FROM_EMAIL = "Task tutorial <no-reply@example.invalid>"
~~~

Create `tasks/templates/emails/task_created.txt`:

~~~text
Your task "{{ title }}" has been created.
Open your task dashboard to review it.
~~~

Create `tasks/templates/emails/task_created.html`:

~~~html
<p>Your task <strong>{{ title }}</strong> has been created.</p>
<p>Open your task dashboard to review it.</p>
~~~

Run in the Django shell:

~~~python
from django.conf import settings
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string

context = {"title": "Learn Django"}
message = EmailMultiAlternatives(
    subject="Task created",
    body=render_to_string("emails/task_created.txt", context),
    from_email=settings.DEFAULT_FROM_EMAIL,
    to=["learner@example.invalid"],
)
message.attach_alternative(
    render_to_string("emails/task_created.html", context), "text/html"
)
assert message.send(fail_silently=False) == 1
~~~

Expect MIME output in the terminal; this backend makes no external delivery. Django rejects newline-containing header values to reduce header injection. Still validate recipients, keep header construction controlled, and preserve template escaping for user content. [Django email documentation](https://docs.djangoproject.com/en/5.2/topics/email/)

For production, configure SMTP or a provider backend with secrets outside source control, explicit timeouts and an authenticated sending domain. Queue durable notification intent with the business transaction; a provider timeout can occur after acceptance, so retries need provider idempotency or a reconciliation policy. Track acceptance, bounce and complaint separately from “delivered.” Use the locmem email backend and mail.outbox in tests.

## Lab: language and time-zone presentation

Keep `USE_I18N = True` and `USE_TZ = True`. Set `LANGUAGES = [("en", "English"), ("hi", "Hindi")]` and `LOCALE_PATHS = [BASE_DIR / "locale"]`. Insert LocaleMiddleware after SessionMiddleware and before CommonMiddleware in the existing middleware list.

Create `tasks/labels.py`:

~~~python
from django.utils.translation import gettext, ngettext

def completed_label():
    return gettext("Task completed")

def task_count_label(count):
    return ngettext(
        "%(count)d task", "%(count)d tasks", count
    ) % {"count": count}
~~~

With GNU gettext tools installed and available on PATH, run `python manage.py makemessages -l hi`. Edit `locale/hi/LC_MESSAGES/django.po`, filling the Task completed translation with `कार्य पूरा हुआ`, then run `python manage.py compilemessages`. Test:

~~~python
from zoneinfo import ZoneInfo
from django.utils import timezone, translation
from tasks.labels import completed_label

with translation.override("hi"):
    print(completed_label())
print(timezone.localtime(timezone.now(), ZoneInfo("Asia/Kolkata")).isoformat())
~~~

Django's standard catalog workflow uses gettext, not arbitrary ICU message syntax. Use gettext_lazy for labels evaluated later, and ngettext for grammatical plurals rather than concatenating translated fragments. [Django translation workflow](https://docs.djangoproject.com/en/5.2/topics/i18n/translation/)

Store aware instants; render them in the user's named zone. A local clock time without a date and zone may be ambiguous around daylight-saving transitions. [Django time-zone handling](https://docs.djangoproject.com/en/5.2/topics/i18n/timezones/) Localized date/number formatting belongs in presentation. Keep JSON field names, status codes, ISO timestamps and documented numeric representations stable across languages. [Django format localization](https://docs.djangoproject.com/en/5.2/topics/i18n/formatting/)

## Failure tests and completion checklist

Break a static path and verify deployment detects missing assets. Roll back a transaction after saving a test upload and observe the orphan. Request another tenant's attachment and verify denial. Test email header newlines, HTML escaping, missing translations, plural forms and multiple time zones.

Common mistakes include deploying media with collectstatic, assuming row deletion removes bytes, exposing quarantine storage publicly, treating provider acceptance as final delivery, and returning locale-dependent numbers in a JSON API.

Exercise: add a notification preference containing language and time-zone name, then render the same committed task event for two recipients without changing its machine-readable payload. Add an orphan-cleanup report that identifies candidates without deleting active uploads.

- [ ] Static assets and private media have separate storage policies.
- [ ] File cleanup survives database/storage partial failures.
- [ ] Email templates are escaped, testable and durably dispatched.
- [ ] Locale selection cannot change machine-readable API semantics.
- [ ] Translation builds and timezone boundary cases are tested.
