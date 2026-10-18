# Deployment, cloud infrastructure, and orchestration

[Tutorial index](../README.md) · Related: [release strategies](03-ci-cd-and-release-strategies.md)

## Outcome and prerequisites

Choose a deployment shape based on workload and operational needs, then describe how the Task Service runs safely in it. You need containers, HTTP networking, and database basics. Kubernetes examples require an existing authorized lab cluster and compatible `kubectl`; they are templates for review, not instructions to create cloud resources or incur charges.

A virtual machine provides broad control but leaves patching and process supervision to you. A managed application platform reduces infrastructure work while constraining runtime and networking choices. Managed containers retain a familiar image deployment model. Kubernetes coordinates workloads across nodes but introduces a substantial control plane and operational surface. Serverless functions suit some event-driven and bursty workloads but impose execution, lifecycle, and connection-management constraints.

Choose the least operationally demanding platform that meets requirements. “Microservices” and “Kubernetes” are not prerequisites for a dependable backend. A small service with a managed database and a disciplined release pipeline may be easier to operate than an unnecessarily distributed platform.

## Map the production request path

Write the path from DNS to edge/load balancer, TLS termination, routing, application instance, connection pool, and database. Identify ownership and timeouts at each step. Keep the database on a private network and grant access only from intended workloads. Separate administrative access from application access. Record where client identity is established and which proxy headers can be trusted.

Stateless application replicas can be replaced when durable state lives in a database or object store. Local memory caches and sessions are per-instance unless deliberately shared. Uploads stored only on a container filesystem disappear with replacement. Background workers require their own lifecycle, scaling signal, and shutdown behavior even when they share the application codebase.

Account for total database connections: `replicas × per-instance pool maximum`, plus workers, migrations, administration, and failover headroom. Autoscaling an API from five replicas to fifty can exhaust the database before CPU improves. In function platforms, rapidly expanding concurrency can create connection storms; AWS documents RDS Proxy as a connection-pooling option for Lambda workloads. [Lambda and RDS](https://docs.aws.amazon.com/lambda/latest/dg/services-rds.html)

## Walkthrough: review a Kubernetes workload template

1. Build and publish a tested Django Task Service image through the release process, using Gunicorn for WSGI or an explicitly configured ASGI server. Resolve its immutable digest. Create configuration and secrets through the cluster's approved mechanism. The template deliberately contains a placeholder image and assumes the `task-service-config` Secret supplies `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`, and `DJANGO_SECRET_KEY`. Configure production allowed hosts and disable debug/demo mode in the selected settings before deploying.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: task-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: task-service
    spec:
      terminationGracePeriodSeconds: 35
      volumes:
        - name: temporary
          emptyDir:
            sizeLimit: 64Mi
      containers:
        - name: api
          image: REPLACE_WITH_REGISTRY_IMAGE_AT_SHA256_DIGEST
          ports:
            - name: http
              containerPort: 3000
          envFrom:
            - secretRef:
                name: task-service-config
          volumeMounts:
            - name: temporary
              mountPath: /tmp
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /health/live/
              port: http
            periodSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /health/ready/
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
          livenessProbe:
            httpGet:
              path: /health/live/
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 10001
            runAsGroup: 10001
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

These resource values are lab starting points, not sizing recommendations. A compatible non-root image using UID/GID 10001 and a stateless application are prerequisites. The small `/tmp` mount supports production-server temporary files while the image filesystem remains read-only; configure and measure its size. A Service, ingress/gateway, certificates, namespace policy, and network policy are separate required deployment decisions; this manifest alone does not expose an application endpoint.

2. Confirm that `/health/live/` measures whether the process can continue, while `/health/ready/` measures whether this instance should receive requests. Do not make liveness depend on a shared database outage: restarting every instance can worsen that outage. A startup probe allows initialization before the other probes run. [Kubernetes probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

3. Replace the image placeholder, choose the lab namespace, and validate the manifest with the cluster's normal review workflow. Read-only inspection after an authorized deployment includes `kubectl get pods`, `kubectl describe deployment task-service`, and `kubectl logs deployment/task-service --tail=50`, with the correct namespace flag. Expected evidence includes ready replicas, the intended image digest, no restart loop, and successful critical journeys through the actual service route.

## Capacity, placement, and autoscaling

Requests inform scheduling; limits constrain resource use. Memory limits can result in termination when exceeded, while CPU limits can produce throttling. Measure the application under realistic load before tuning. [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

Spread replicas across failure domains when availability requires it. Multiple replicas on one node do not survive that node's loss. Pod disruption budgets help govern some voluntary disruptions, but they are not protection from every failure or a substitute for correct rollout settings. Ensure spare capacity exists for surge replicas and maintenance.

Autoscale on a signal connected to useful work: CPU for CPU-bound traffic, queue age for workers, or carefully designed request concurrency metrics. Use stabilization and bounded scaling to avoid oscillation. Define minimum replicas, maximum capacity, database limits, and the behavior when scaling cannot continue. Backpressure remains necessary at the maximum.

## Infrastructure as code and drift

Describe networks, roles, databases, and deployment settings in version-controlled infrastructure code. Review plans before applying them, protect state files because they may contain secrets, and avoid concurrent modifications without state locking. Keep environments separate enough that a development change cannot accidentally target production.

Drift occurs when actual resources differ from declared configuration. Detect it regularly and decide whether to adopt or revert the change. Emergency console changes need a follow-up reconciliation so the next automated apply does not unexpectedly undo the fix. Manage secrets through a dedicated system rather than hardcoding their values into infrastructure source.

## Verification and exercises

In a lab, remove one replica through the platform's controlled mechanism and verify continued service and replacement. Make the database unavailable and check that readiness changes without a restart storm. Restore it and observe recovery. Test a shutdown during active requests and confirm bounded draining.

Common mistakes include exposing databases publicly, using unlimited pools, omitting resource measurements, trusting every forwarded header, and storing durable data on ephemeral disks. Exercise: compare a VM, managed container platform, and Kubernetes for the same expected traffic and staffing. Include deployment effort, backup ownership, incident response, scaling limits, and monthly cost inputs without inventing provider prices.
