# Django REST Framework: serializers, views, permissions, and testing

[Curriculum](../README.md) · Prerequisites: [capstone](../09-capstone/01-task-service.md), ORM, and tenant authorization

## What DRF adds

Django REST Framework (DRF) adds request parsing, response rendering, serializers, authentication adapters, permission checks, generic views, routers, pagination, throttling, and API testing helpers. It builds on Django's ORM and middleware; it does not automatically define tenant ownership or safe business transitions.

This tutorial adds a separate authenticated list/create endpoint to a disposable capstone copy. It does not replace the capstone's version-aware PATCH with a generic unrestricted update. Install DRF in the virtual environment, record the resolved version, and add `rest_framework` to `INSTALLED_APPS`:

```powershell
python -m pip install djangorestframework
```

Serializer input passes through validation before becoming `validated_data`; a ModelSerializer can derive fields from a model, but you should explicitly select public fields. [DRF serialization tutorial](https://www.django-rest-framework.org/tutorial/1-serialization/).

## 1. Add explicit membership

For this lab, use the same `Membership` model as the preceding forms/admin tutorial. If it is already present, do not add it again. Otherwise add this model to `tasks/models.py`. An existing project with a different membership model must adapt the imports and fields consistently:

```python
from django.conf import settings

class Membership(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    role = models.CharField(max_length=10, choices=[
        ("viewer", "Viewer"), ("editor", "Editor"), ("admin", "Admin"),
    ])
    is_active = models.BooleanField(default=True)

    class Meta:
        constraints = [models.UniqueConstraint(
            fields=["tenant", "user"], name="membership_tenant_user_uq",
        )]
```

Run `python manage.py makemigrations tasks` and `python manage.py migrate`. In a real new product, decide your custom user model before initial auth migrations; this disposable extension uses the project's existing user model through the supported setting.

## 2. Define separate input and output contracts

Create `tasks/serializers.py`:

```python
from rest_framework import serializers
from .models import Task

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ["id", "title", "status", "version", "created_at", "updated_at"]
        read_only_fields = ["id", "version", "created_at", "updated_at"]

    def validate_title(self, value):
        value = value.strip()
        if not value:
            raise serializers.ValidationError("A title is required.")
        return value

    def to_internal_value(self, data):
        if not isinstance(data, dict):
            raise serializers.ValidationError("Expected a JSON object.")
        extra = set(data) - {"title", "status"}
        if extra:
            raise serializers.ValidationError("Unsupported input field.")
        return super().to_internal_value(data)
```

The explicit unknown-field rule rejects attempts to submit tenant, version, or audit fields. Read-only declarations alone can ignore input rather than reject it, so test the behavior you intend. Nested writes require an explicit transaction and ownership policy; do not assume a nested serializer automatically implements every relational update safely.

## 3. Build a bounded authorized view

Create `tasks/drf_views.py`:

```python
from django.db import transaction
from rest_framework.authentication import SessionAuthentication
from rest_framework.exceptions import PermissionDenied
from rest_framework.generics import ListCreateAPIView
from rest_framework.pagination import PageNumberPagination
from rest_framework.parsers import JSONParser
from rest_framework.permissions import IsAuthenticated
from .models import Task, Membership
from .serializers import TaskSerializer

class TaskPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = "page_size"
    max_page_size = 100

class TaskListCreateView(ListCreateAPIView):
    serializer_class = TaskSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticated]
    parser_classes = [JSONParser]
    pagination_class = TaskPagination

    def membership(self):
        if not hasattr(self, "_membership"):
            self._membership = Membership.objects.filter(
                tenant_id=self.kwargs["tenant_id"], user=self.request.user, is_active=True,
            ).first()
        if self._membership is None:
            raise PermissionDenied("Tenant access denied.")
        return self._membership

    def get_queryset(self):
        membership = self.membership()
        return Task.objects.filter(tenant_id=membership.tenant_id).order_by(
            "-created_at", "-id",
        )

    def perform_create(self, serializer):
        with transaction.atomic():
            membership = Membership.objects.select_for_update().filter(
                tenant_id=self.kwargs["tenant_id"], user=self.request.user,
                is_active=True, role__in=["editor", "admin"],
            ).first()
            if membership is None:
                raise PermissionDenied("Write permission required.")
            serializer.save(tenant_id=membership.tenant_id)
```

Add a route to the existing `tasks/urls.py`:

```python
from .drf_views import TaskListCreateView

urlpatterns += [path(
    "drf/tenants/<uuid:tenant_id>/tasks/",
    TaskListCreateView.as_view(), name="drf-task-list",
)]
```

List authorization is implemented through queryset filtering. Create permission is checked before save. Object-level hooks do not automatically filter every item in a list or solve creation authorization. [DRF permissions](https://www.django-rest-framework.org/api-guide/permissions/).

This lab uses page-number pagination to show DRF's mechanism. It has offset tradeoffs; preserve the capstone's cursor design for workloads that need keyset behavior. Do not claim these two endpoints have identical pagination contracts.

## 4. Test actual boundaries

Create users through `get_user_model().objects.create_user`, grant one membership with write permission and another read-only membership, and use DRF's APIClient in tests. Verify anonymous access is denied, membership is required, read-only members cannot create, and tenant rows never mix.

`force_authenticate` is useful for isolated permission tests but bypasses actual authentication. Add separate tests using a real session and `APIClient(enforce_csrf_checks=True)` for cookie-authenticated POST. Session authentication can return 403 for an unauthenticated denial depending on the authentication scheme; assert the documented behavior instead of assuming every unauthenticated request is 401.

For browser login, use Django's protected login flow and CSRF token handling. Do not reuse the capstone's `csrf_exempt` demo decorator on this endpoint. Session authentication and OAuth bearer tokens have different CSRF and revocation considerations.

## Viewsets, throttling, filtering, and schemas

A ViewSet groups related operations and a router generates URLs. Convenience increases the importance of auditing which actions exist. A full ModelViewSet can expose update/delete actions you did not intend. Restrict methods and implement mutations through the version-aware service.

Allowlist filters and sort keys, bound pagination, and restrict parser/renderers to the required formats. Throttling helps application policy but is not a complete distributed abuse-control boundary; test its storage and concurrency behavior. Generate an OpenAPI contract using an integration compatible with your chosen DRF version and annotate the business rules generators cannot infer.

## Exercises and checklist

Add a detail endpoint that scopes its queryset before lookup. Implement PATCH through the capstone's expected-version transaction rather than default `serializer.save()` behavior. Add a request-count assertion to prevent a newly added relation from introducing N+1 queries.

- [ ] Serializers expose explicit fields and reject unauthorized input.
- [ ] List, retrieve, create, update, and delete each have authorization rules.
- [ ] Session-authenticated writes enforce CSRF.
- [ ] Generic views do not silently remove concurrency controls.
- [ ] Pagination, throttles, schema, and errors are tested against real responses.
