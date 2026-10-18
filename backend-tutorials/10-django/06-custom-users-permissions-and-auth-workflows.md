# Django custom users, permissions and authentication workflows

Prerequisites: [course map](../README.md), [models](01-models-orm-and-querysets.md), [email](05-static-media-email-and-localization.md), and Django authentication concepts. Use a **fresh practice database and a fresh copy of the capstone project before its first migrate command**. Changing AUTH_USER_MODEL in an already migrated application requires a planned data/schema migration; this lab does not perform that conversion.

## Objectives

Choose a replaceable user model, configure session and password-reset views, implement single-use email verification, and authorize task access through current tenant membership.

## Choose the user model early

AbstractUser retains Django's familiar username, password and permission structure. AbstractBaseUser supplies lower-level authentication foundations and requires more decisions, including an identifier, manager, admin/forms and permission integration. Prefer the level that matches the domain without collecting unrelated profile data in the authentication table.

The selected custom user must be created in its app's first migration. Use settings.AUTH_USER_MODEL for model relations and get_user_model() when accessing the active class at runtime. Cross-app migrations referencing users use migrations.swappable_dependency(settings.AUTH_USER_MODEL). [Django custom-user guidance](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/)

## Lab 1: create the account foundation

Before migrating the fresh capstone, run `python manage.py startapp accounts`. Add accounts to INSTALLED_APPS and set `AUTH_USER_MODEL = "accounts.User"`. Create `accounts/models.py`:

~~~python
from uuid import uuid4
from django.contrib.auth.models import AbstractUser
from django.db import models
from django.db.models.functions import Lower

class User(AbstractUser):
    email = models.EmailField(unique=True)
    email_verified = models.BooleanField(default=False)
    verification_nonce = models.UUIDField(default=uuid4, editable=False)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                Lower("email"), name="accounts_user_email_casefold_uq"
            )
        ]
~~~

This lab chooses case-insensitive email uniqueness using PostgreSQL's lower() behavior; define and test your application's address normalization policy separately. Username remains the login identifier. Email verification is an explicit application state and does not automatically block Django's default login.

In `accounts/admin.py` register User with django.contrib.auth.admin.UserAdmin. Then run:

~~~powershell
python manage.py makemigrations accounts tasks
python manage.py migrate
python manage.py createsuperuser
~~~

For a regular learning account, run this in `python manage.py shell`:

~~~python
from getpass import getpass
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password

User = get_user_model()
user = User(username="learner", email="learner@example.invalid")
password = getpass("Learning account password: ")
validate_password(password, user)
user.set_password(password)
user.save()
~~~

set_password hashes the credential; assigning raw text to user.password does not. create_user handles hashing, but programmatic creation still needs the intended password-validation policy. Never log passwords or reset tokens.

## Lab 2: session login and password reset

Keep SessionMiddleware, AuthenticationMiddleware and CsrfViewMiddleware enabled in the generated project. Add settings:

~~~python
LOGIN_URL = "/accounts/login/"
LOGIN_REDIRECT_URL = "/accounts/me/"
LOGOUT_REDIRECT_URL = "/accounts/login/"
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
DEFAULT_FROM_EMAIL = "no-reply@example.invalid"
PASSWORD_RESET_TIMEOUT = 3600
~~~

Create `accounts/views.py`:

~~~python
from django.contrib.auth.decorators import login_required
from django.http import JsonResponse

@login_required
def me(request):
    return JsonResponse({
        "id": request.user.pk,
        "username": request.user.get_username(),
        "email_verified": request.user.email_verified,
    })
~~~

Append routes to the existing config/urls.py urlpatterns:

~~~python
from django.urls import include, path
from accounts.views import me

urlpatterns += [
    path("accounts/me/", me, name="account-me"),
    path("accounts/", include("django.contrib.auth.urls")),
]
~~~

Django supplies authentication views but expects your application templates. The following shell snippet writes a minimal complete template set into the new accounts app:

~~~python
from pathlib import Path

folder = Path("accounts/templates/registration")
folder.mkdir(parents=True, exist_ok=True)
form = """<!doctype html><title>Account</title>
<form method="post">{% csrf_token %}{{ form.as_p }}
<input type="hidden" name="next" value="{{ next|default:'' }}">
<button type="submit">Continue</button></form>"""
for name in ("login", "password_reset_form", "password_change_form"):
    (folder / f"{name}.html").write_text(form, encoding="utf-8")
for name in ("password_reset_done", "password_reset_complete", "password_change_done"):
    (folder / f"{name}.html").write_text(
        "<p>Request complete. If applicable, check your email.</p>"
        "<a href='/accounts/login/'>Sign in</a>", encoding="utf-8"
    )
(folder / "password_reset_confirm.html").write_text(
    "{% if validlink %}" + form +
    "{% else %}<p>This reset link is invalid or expired.</p>{% endif %}",
    encoding="utf-8",
)
(folder / "password_reset_subject.txt").write_text(
    "Reset your tutorial password", encoding="utf-8"
)
(folder / "password_reset_email.html").write_text(
    "{{ protocol }}://{{ domain }}"
    "{% url 'password_reset_confirm' uidb64=uid token=token %}",
    encoding="utf-8",
)
~~~

Start the local server and visit /accounts/login/, then /accounts/me/. Visit /accounts/password_reset/ and submit the learning email; follow the reset URL printed by the console backend. Repeat with an unknown email and verify the outward completion response does not reveal account existence. Logout uses a CSRF-protected POST to /accounts/logout/.

The built-in views handle standard form/session/reset behavior. Add deployment rate limits and abuse monitoring for login, reset and verification endpoints; the default authentication backend does not supply a complete brute-force defense. Use HTTPS and secure session/CSRF cookie settings in production. [Django authentication views](https://docs.djangoproject.com/en/5.2/topics/auth/default/)

The explicit reset timeout is measured in seconds. Expiration is only one invalidation condition; verify successful reset and token reuse behavior through the built-in flow. [PASSWORD_RESET_TIMEOUT](https://docs.djangoproject.com/en/5.2/ref/settings/#password-reset-timeout)

## Lab 3: single-use email confirmation

Create `accounts/verification.py`. The timestamped signature protects integrity, while the database nonce supplies single-use consumption:

~~~python
from uuid import uuid4
from django.contrib.auth import get_user_model
from django.core import signing
from django.core.exceptions import ValidationError

SALT = "accounts.email-verification.v1"

def issue_email_verification(user):
    return signing.dumps({
        "user_id": user.pk,
        "email": user.email,
        "nonce": str(user.verification_nonce),
    }, salt=SALT)

def consume_email_verification(token):
    try:
        payload = signing.loads(token, salt=SALT, max_age=3600)
        changed = get_user_model().objects.filter(
            pk=payload["user_id"],
            email=payload["email"],
            verification_nonce=payload["nonce"],
            email_verified=False,
            is_active=True,
        ).update(email_verified=True, verification_nonce=uuid4())
        return changed == 1
    except (signing.BadSignature, ValidationError, KeyError, TypeError, ValueError):
        return False
~~~

In the shell, load the learner, issue a token, and call consume_email_verification twice: expect True then False. Django signing authenticates a payload; it does not encrypt it. [Django signing documentation](https://docs.djangoproject.com/en/5.2/topics/signing/)

Connect issuance to enrollment with a durable email intent. A confirmation link should show a page on GET and perform the state change on a CSRF-protected POST. On email change, atomically clear email_verified and rotate verification_nonce before sending a new confirmation. Protect resend with rate limits. Require verified status explicitly for workflows that depend on proven mailbox ownership.

## Tenant membership and object access

Reuse the canonical tasks.Membership model from [forms and admin](02-forms-templates-and-admin.md) and [DRF](03-django-rest-framework.md). If it is absent, add this model to tasks/models.py after the initial user migration, then run `python manage.py makemigrations tasks` and `python manage.py migrate`. Do not create a second membership table in accounts.

~~~python
from django.conf import settings

class Membership(models.Model):
    tenant = models.ForeignKey("tasks.Tenant", on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    role = models.CharField(max_length=10, choices=[
        ("viewer", "Viewer"), ("editor", "Editor"), ("admin", "Admin")
    ])
    is_active = models.BooleanField(default=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["tenant", "user"], name="membership_tenant_user_uq"
            )
        ]
~~~

Create a tenant and a Membership for the learner through the shell. An application service can scope editable tasks with the current membership:

~~~python
from django.core.exceptions import PermissionDenied
from django.shortcuts import get_object_or_404
from tasks.models import Membership, Task

def editable_task_for(user, task_id):
    if not user.is_authenticated or not user.is_active or not user.email_verified:
        raise PermissionDenied
    allowed_tenants = Membership.objects.filter(
        user=user, is_active=True, role__in=["editor", "admin"]
    ).values("tenant_id")
    return get_object_or_404(
        Task, pk=task_id, tenant_id__in=allowed_tenants
    )
~~~

This is a selection boundary; mutation still needs validation, optimistic concurrency and an appropriate transaction. The capstone's original demonstration tenant shortcut must be replaced when adopting real authentication.

Django groups/model permissions represent capabilities, while membership represents tenant-specific scope. is_staff controls admin access, not business ownership. Core ModelBackend does not implement arbitrary object-level permission storage merely because has_perm accepts obj. Use explicit policies or a suitable object-permission backend.

## MFA and external identity integration

For MFA, integrate a maintained implementation with a documented enrollment/recovery flow, such as [django-allauth MFA](https://docs.allauth.org/en/latest/mfa/introduction.html). Require recent authentication before enrolling or removing factors, confirm a new factor before activating it, protect recovery codes and audit recovery. Resetting an account password must not silently remove its MFA requirement.

For OIDC, use a provider-specific supported client integration and the authorization-code flow appropriate to that client. Validate issuer, audience, signature, expiry and nonce, and bind the callback to the initiating browser state. Store the stable issuer-plus-subject identity mapping; an email address alone is not a safe automatic account-linking key. [OpenID Connect validation requirements](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)

Identity-provider authentication does not grant local tenant roles automatically. Apply explicit invitation, membership, deprovisioning and reauthentication policies. Keep provider secrets and refresh tokens out of browser-readable sessions and logs.

## Failure tests and completion checklist

Test expired, modified and replayed verification/reset links; changed email addresses; revoked memberships; viewer write attempts; anonymous access; and identical task IDs requested outside the user's tenant scope. Add throttling tests and confirm reset responses remain generic.

Common mistakes include changing AUTH_USER_MODEL after migration without a data plan, assigning raw passwords, treating email_verified as automatically enforced, trusting a request's role value, and equating is_staff with tenant administration.

Exercise: add an invitation acceptance service that binds an invitation to a verified address, tenant and allowed role. Consume the invitation atomically and test concurrent acceptance, expiry, revocation and attempts to change the invited role.

- [ ] The custom user is selected before initial migration.
- [ ] Password hashing, validation and reset use the intended framework flow.
- [ ] Email confirmation binds to the current address and is single-use.
- [ ] Every resource operation enforces current membership and capability.
- [ ] MFA recovery and external identity linking have explicit policies.
- [ ] Session, CSRF, token and cross-tenant failures are tested.
