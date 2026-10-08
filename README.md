# E-Commerce GitOps Repository

This repository contains the Kubernetes and GitOps configuration for the E-Commerce DevOps project.

The application source code is maintained separately in the `ecommerce-app` repository.

This repository represents the **desired state of the Kubernetes environment**.

The main idea is:

```text
Application code changes
        ↓
GitHub Actions
        ↓
Docker image built
        ↓
Image pushed to GHCR
        ↓
Image version updated in this repository
        ↓
Argo CD detects the Git change
        ↓
Kubernetes deployment updated
```

Instead of manually deploying application changes with `kubectl apply`, application deployments are controlled through Git.

---

# 1. Purpose of This Repository

The `ecommerce-gitops` repository is responsible for:

```text
Kubernetes Deployments
Kubernetes Services
Kustomize configuration
Development environment configuration
Ingress
Horizontal Pod Autoscaling
Monitoring
Logging
Alerts
Sealed Secrets
GitOps deployments
Rollback configuration
```

The application repository answers:

```text
"What should the application do?"
```

This GitOps repository answers:

```text
"How should the application run inside Kubernetes?"
```

---

# 2. High-Level Architecture

The complete deployment flow is:

```text
Developer
   |
   | git push
   v
ecommerce-app
   |
   v
GitHub Actions
   |
   +--> Test
   |
   +--> Build Docker image
   |
   +--> Push image to GHCR
   |
   v
Update image tag in ecommerce-gitops
   |
   v
Git commit
   |
   v
Argo CD
   |
   | compares Git with cluster
   v
Kubernetes
   |
   +--> product-service
   |
   +--> cart-service
   |
   +--> order-service
   |
   +--> frontend
   |
   +--> redis
```

This is the main GitOps flow used throughout the project.

---

# 3. Why GitOps?

Without GitOps, a deployment may look like:

```text
Developer
   ↓
kubectl apply
   ↓
Kubernetes
```

The problem is that someone can manually change the cluster and Git may no longer show the real desired configuration.

With GitOps:

```text
Git
 ↓
Argo CD
 ↓
Kubernetes
```

Git becomes the main source of truth.

This means changes are:

```text
version controlled
reviewable
repeatable
traceable
revertible
```

---

# 4. Repository Structure

The repository is organized roughly like this:

```text
ecommerce-gitops/
|
+-- apps/
|   |
|   +-- product-service/
|   |
|   +-- cart-service/
|   |
|   +-- order-service/
|   |
|   +-- frontend/
|   |
|   +-- redis/
|   |
|   +-- ingress/
|   |
|   +-- monitoring/
|
+-- envs/
|   |
|   +-- dev/
|       |
|       +-- kustomization.yaml
|       +-- order-secrets-sealed.yaml
|
+-- monitoring/
|   |
|   +-- kube-prometheus-stack-values.yaml
|   +-- loki-values.yaml
|   +-- alloy-values.yaml
|
+-- README.md
```

The exact configuration is separated into application resources, environment configuration and monitoring configuration.

---

# 5. The `apps` Directory

The `apps/` directory contains Kubernetes manifests for the application components.

For example:

```text
apps/product-service/
apps/cart-service/
apps/order-service/
apps/frontend/
apps/redis/
```

A service normally contains Kubernetes resources such as:

```text
Deployment
Service
ConfigMap
HPA
```

depending on what that service requires.

---

# 6. Kubernetes Deployment

A Deployment tells Kubernetes how an application should run.

For example:

```text
Deployment: product-service
```

defines things such as:

```text
container image
replica count
container port
resource requests
resource limits
environment variables
health probes
```

The Deployment does not directly create and maintain Pods itself.

The flow is:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

The ReplicaSet tries to keep the required number of Pods running.

---

# 7. Desired State

Kubernetes works using desired state.

For example:

```text
Desired replicas = 3
Actual Pods      = 2
```

Kubernetes sees that one Pod is missing and creates another one.

The goal is:

```text
Actual state
     =
Desired state
```

This behavior was also demonstrated during the self-healing test.

---

# 8. Kubernetes Services

Pods are temporary.

A Pod can:

```text
restart
be replaced
get a new IP address
```

Therefore applications should not normally communicate directly using Pod IP addresses.

A Kubernetes Service provides a stable network endpoint.

The flow becomes:

```text
Client / another service
        ↓
Kubernetes Service
        ↓
matching Pods
```

For example:

```text
order-service
      ↓
product-service Kubernetes Service
      ↓
product-service Pod
```

---

# 9. ConfigMaps

ConfigMaps store normal, non-sensitive configuration.

Examples include:

```text
service URLs
application configuration
environment settings
```

For example, `order-service` can receive normal configuration using:

```yaml
envFrom:
  - configMapRef:
      name: order-service-config
```

This loads ConfigMap keys into the container as environment variables.

---

# 10. Secrets

Sensitive values should not normally be stored in ConfigMaps.

Examples include:

```text
passwords
API keys
tokens
credentials
```

For these values, Kubernetes Secrets are used.

However, a normal Kubernetes Secret manifest is not safe to commit directly to Git because the values in its `data` field are Base64 encoded rather than protected encryption.

For this reason, this project uses **Sealed Secrets**.

---

# 11. Kustomize

Kustomize is used to organize and customize Kubernetes manifests.

The development environment is under:

```text
envs/dev/
```

Its main file is:

```text
envs/dev/kustomization.yaml
```

This file combines application resources and environment-specific configuration.

The general idea is:

```text
Base application manifests
         ↓
Kustomize
         ↓
Development configuration
         ↓
Complete Kubernetes manifests
```

---

# 12. Validate Kustomize

Before committing Kubernetes configuration, the environment can be validated with:

```bash
kubectl kustomize envs/dev
```

If only validation is required:

```bash
kubectl kustomize envs/dev > /dev/null && echo "Kustomize OK"
```

This means:

```text
Build the manifests
        ↓
Discard the generated output
        ↓
If successful
        ↓
print "Kustomize OK"
```

This is useful for finding YAML or Kustomize configuration problems before deployment.

---

# 13. Image Management with Kustomize

Application Docker images are also controlled through the development Kustomization.

For example:

```yaml
images:
  - name: product-service
    newName: ghcr.io/abdullahmudassar/product-service
    newTag: <git-commit-sha>
```

The `newTag` identifies the exact application image that Kubernetes should run.

Using a specific Git SHA instead of only:

```text
latest
```

makes deployments easier to trace and roll back.

---

# 14. CI to GitOps Flow

When application code changes:

```text
ecommerce-app
     ↓
GitHub Actions
     ↓
Docker image
     ↓
GitHub Container Registry
     ↓
GitHub Actions updates newTag
     ↓
ecommerce-gitops
```

For example:

```text
product-service source commit
        ↓
Docker image
        ↓
ghcr.io/abdullahmudassar/product-service:<SHA>
        ↓
newTag updated in envs/dev/kustomization.yaml
```

Argo CD then detects that Git changed.

---

# 15. Argo CD

Argo CD continuously compares:

```text
Desired state in Git
        ↓
         vs
        ↓
Actual state in Kubernetes
```

If they are different, Argo CD can synchronize the cluster with Git.

The application is:

```text
ecommerce-dev
```

Useful command:

```bash
kubectl get application ecommerce-dev -n argocd
```

Example:

```text
NAME            SYNC STATUS   HEALTH STATUS
ecommerce-dev   Synced        Healthy
```

---

# 16. Argo CD Sync Status vs Health Status

These are different concepts.

## Synced

```text
Synced
```

means:

> The Kubernetes resources match the desired configuration stored in Git.

## OutOfSync

```text
OutOfSync
```

means:

> Git and the current cluster state are different.

## Healthy

```text
Healthy
```

means:

> The deployed resources are operating correctly.

## Progressing

```text
Progressing
```

means:

> Kubernetes is still trying to complete the rollout or resource transition.

This means an application can sometimes be:

```text
Synced + Progressing
```

Git has been applied successfully, but the application has not reached a fully healthy state yet.

We observed this during the rollback drill.

---

# 17. Ingress and Traefik

Traefik is used as the Kubernetes ingress controller.

External requests follow approximately this path:

```text
Browser / curl / k6
        ↓
Host port 80
        ↓
kind
        ↓
Traefik
        ↓
Ingress rules
        ↓
Kubernetes Service
        ↓
Application Pod
```

The shop can be accessed using:

```text
http://shop.localtest.me
```

Example:

```text
http://shop.localtest.me/api/products
```

---

# 18. Health and Readiness

Application health is checked through `/health`.

For example:

```text
Kubernetes
    ↓
GET /health
    ↓
Application
```

If the endpoint returns:

```text
HTTP 200
```

the probe succeeds.

If it returns:

```text
HTTP 500
```

the probe fails.

Readiness is important because a Pod that is not Ready should not receive normal application traffic through the Service.

---

# 19. Horizontal Pod Autoscaler

The project uses Kubernetes HPA to automatically scale workloads.

Check HPA using:

```bash
kubectl get hpa -n ecommerce-dev
```

Conceptually:

```text
Traffic increases
      ↓
CPU increases
      ↓
Metrics Server
      ↓
HPA compares current usage with target
      ↓
HPA increases replica count
```

During load testing, Product Service scaled approximately:

```text
1 replica
   ↓
3 replicas
   ↓
4 replicas
```

when CPU usage exceeded the configured target.

---

# 20. HPA and Metrics Server

HPA requires resource metrics.

The flow is approximately:

```text
Pod CPU usage
      ↓
Metrics Server
      ↓
Kubernetes metrics API
      ↓
HPA
      ↓
scaling decision
```

Without metrics, the HPA cannot make normal CPU-based scaling decisions.

---

# 21. Monitoring Architecture

The monitoring stack uses:

```text
Prometheus
Grafana
```

The main flow is:

```text
Application /metrics endpoints
          ↓
ServiceMonitor
          ↓
Prometheus
          ↓
Grafana
```

The monitoring stack was installed using the `kube-prometheus-stack` Helm chart.

---

# 22. ServiceMonitor

ServiceMonitor resources tell Prometheus which Kubernetes Services should be scraped.

ServiceMonitors were configured for:

```text
product-service
cart-service
order-service
```

The basic relationship is:

```text
Application Pod
      ↓
Service
      ↓
ServiceMonitor
      ↓
Prometheus
```

The Services use labels and named ports so the ServiceMonitors can correctly discover them.

---

# 23. Prometheus

Prometheus collects metrics from the application.

Examples include:

```text
HTTP request count
HTTP response duration
orders created
Kubernetes resource metrics
```

Prometheus stores these metrics as time-series data.

They can then be queried using PromQL.

---

# 24. Grafana

Grafana is used to visualize metrics collected by Prometheus.

A custom dashboard called:

```text
Shop Overview
```

was created.

It contains panels such as:

```text
Requests per second
Error percentage
p95 response time
Orders created per minute
Application error logs
```

Grafana is also used to query Loki logs.

---

# 25. Access Grafana

Grafana can be exposed locally using port forwarding:

```bash
kubectl port-forward \
  -n monitoring \
  svc/monitoring-grafana \
  3000:80
```

Then open:

```text
http://localhost:3000
```

---

# 26. Requests Per Second

Prometheus can calculate request rate using the application HTTP request counter.

Conceptually:

```text
HTTP requests over time
        ↓
rate()
        ↓
requests per second
```

This panel was useful during load testing because it clearly showed traffic increasing when k6 started.

---

# 27. Error Percentage

The dashboard monitors HTTP 5xx responses.

Conceptually:

```text
5xx request rate
       ÷
all request rate
       ×
100
       ↓
error percentage
```

During the self-healing test, a temporary error spike was visible while Product Service Pods were being replaced.

---

# 28. p95 Response Time

The dashboard also shows the 95th percentile response time.

For example:

```text
p95 = 150 ms
```

means:

> Around 95% of requests completed within 150 ms.

It is useful because an average can hide a smaller number of slow requests.

---

# 29. Centralized Logging

The logging stack uses:

```text
Alloy
Loki
Grafana
```

The flow is:

```text
Application Pods
      ↓
Container logs
      ↓
Alloy
      ↓
Loki
      ↓
Grafana
```

---

# 30. Alloy

Grafana Alloy discovers Kubernetes Pods and collects their logs.

It adds useful Kubernetes information such as:

```text
namespace
pod
application labels
```

and forwards the log streams to Loki.

This allows logs from many Pods to be searched centrally.

---

# 31. Loki

Loki stores application logs.

Instead of manually running:

```bash
kubectl logs
```

against every individual Pod, logs can be searched centrally from Grafana.

An example LogQL query is:

```text
{namespace="ecommerce-dev", app="order-service"}
```

To find order-created logs:

```text
{namespace="ecommerce-dev", app="order-service"} |= "order created"
```

---

# 32. Monitoring Failure Example

Redis was intentionally stopped during an earlier monitoring exercise.

This caused Cart Service problems.

The failure could be investigated using:

```text
Grafana metrics
        +
Loki logs
```

This demonstrated an important observability concept:

```text
Metrics
→ show that something is wrong

Logs
→ help explain why it is wrong
```

---

# 33. Alerting

Prometheus alert rules are defined using:

```text
PrometheusRule
```

Custom alerts include:

```text
ServiceHasNoPods
HighErrorRate
```

---

# 34. ServiceHasNoPods

This alert detects when an important Deployment has no available Pods.

Conceptually:

```text
Deployment
     ↓
available replicas = 0
     ↓
condition remains for configured duration
     ↓
alert fires
```

This helps detect complete service outages.

---

# 35. HighErrorRate

This alert watches the percentage of HTTP 5xx responses.

Conceptually:

```text
5xx requests
      ÷
all HTTP requests
      ↓
percentage
      ↓
greater than threshold
      ↓
condition remains for configured duration
      ↓
alert fires
```

In this project the configured threshold was greater than 5%.

---

# 36. Prometheus and Alertmanager

Their roles are different.

```text
Prometheus
→ collects metrics
→ evaluates alert rules
→ decides when an alert is firing
```

Then:

```text
Alertmanager
→ receives firing alerts
→ groups/routes/manages notifications
```

The flow is:

```text
Metrics
   ↓
Prometheus
   ↓
PrometheusRule
   ↓
Alert fires
   ↓
Alertmanager
```

---

# 37. Sealed Secrets

The project uses Sealed Secrets so that sensitive values can be safely stored in Git in encrypted form.

A fake payment API key was created for `order-service`.

The Secret is named:

```text
order-secrets
```

and contains the key:

```text
PAYMENT_API_KEY
```

---

# 38. Sealed Secrets Flow

The complete flow is:

```text
Plain PAYMENT_API_KEY
        ↓
kubectl create secret --dry-run
        ↓
temporary Kubernetes Secret object
        ↓
kubeseal
        ↓
encrypted SealedSecret
        ↓
envs/dev/order-secrets-sealed.yaml
        ↓
Git
        ↓
Argo CD
        ↓
Kubernetes
        ↓
Sealed Secrets Controller
        ↓
normal Secret: order-secrets
        ↓
order-service
```

Only the encrypted SealedSecret is stored in Git.

---

# 39. Why a Normal Secret Is Not Committed

A standard Kubernetes Secret manifest stores `data` values using Base64 encoding.

Base64 can easily be reversed.

Therefore:

```text
Base64
≠
encryption
```

The SealedSecret contains encrypted data instead.

---

# 40. Public Certificate and Private Key

The Sealed Secrets controller uses cryptographic keys.

Easy way to understand it:

```text
Public certificate
→ used to encrypt

Private key
→ used to decrypt
```

The public certificate can be given to developers.

The private key remains protected by the controller.

This allows developers to create encrypted SealedSecrets without having access to the decryption key.

---

# 41. Using the Secret in Order Service

The Order Service Deployment contains:

```yaml
envFrom:
  - secretRef:
      name: order-secrets
```

This means:

> Load all keys from `order-secrets` as environment variables inside the container.

Therefore:

```text
PAYMENT_API_KEY
```

becomes available to the Node.js application.

---

# 42. Verifying the Secret

The application provides:

```text
GET /api/orders/status
```

A successful result is:

```json
{
  "paymentConfigured": true
}
```

This proves that the complete path worked:

```text
SealedSecret
    ↓
Secret
    ↓
Deployment
    ↓
Pod
    ↓
environment variable
    ↓
application
```

---

# 43. Load Testing

The application was load tested using k6 from the `ecommerce-app` repository.

The test used:

```text
20 virtual users
for 3 minutes
```

The test repeatedly:

```text
GET product list
      +
POST new order
```

During the test, Grafana and the HPA were observed.

---

# 44. Load Test Results

During load:

```text
Requests/sec increased
Orders/min increased
CPU increased
p95 latency changed
HPA scaled Product Service
```

Product Service scaled from approximately:

```text
1 → 3 → 4 Pods
```

This demonstrated autoscaling under application load.

---

# 45. Kubernetes Self-Healing

While k6 was generating traffic, Product Service Pods were deliberately deleted.

Example:

```bash
kubectl delete pod \
  -n ecommerce-dev \
  -l app=product-service
```

The Deployment/ReplicaSet automatically created replacement Pods.

Observed flow:

```text
Pods deleted
    ↓
ReplicaSet detects missing replicas
    ↓
new Pods created
    ↓
ContainerCreating
    ↓
Running
    ↓
Ready
```

The shop recovered without manually recreating the Pods.

---

# 46. Rollback Drill

A controlled deployment failure was created by changing:

```text
/health
```

in Product Service so it returned:

```text
HTTP 500
```

The bad source-code change was pushed.

The CI/CD pipeline then produced a new image and updated this GitOps repository.

---

# 47. Failed Rollout

After Argo deployed the bad image, the new Pods appeared with a different ReplicaSet hash.

The new Pods remained:

```text
0/1 Running
```

because the health/readiness checks were failing.

The Deployment rollout did not complete.

Argo showed:

```text
Synced
Progressing
```

meaning the Git desired state had been applied, but the application was not becoming healthy.

---

# 48. Why the Old Pod Stayed Running

During the failed rollout, Kubernetes kept the previous healthy replica available.

Conceptually:

```text
Old version
/health → 200
Ready ✅
       |
       | continues serving
       |
New version
/health → 500
Ready ❌
```

This prevented Kubernetes from immediately replacing the working version with an unhealthy one.

---

# 49. Git-Based Rollback

The bad image-update commit in this repository was identified.

The previous good Product Service image was:

```text
93f32d51e999da4ae57fcff72045f75f52f7dfb7
```

The deliberately bad image was:

```text
016dfcf1d9bc8b654b747a684850ea35c5732939
```

The bad GitOps commit was reverted using:

```bash
git revert <bad-gitops-commit>
```

Then the revert was pushed.

---

# 50. Why `git revert`?

`git revert` does not remove Git history.

Instead:

```text
Good state
    ↓
Bad commit
    ↓
Revert commit
    ↓
Good state restored
```

This is useful for shared branches because the complete history remains visible.

The rollback therefore remained fully traceable.

---

# 51. Rollback Recovery Flow

The recovery worked like this:

```text
Bad image
   ↓
Pods fail readiness
   ↓
Problem detected
   ↓
Bad GitOps commit identified
   ↓
git revert
   ↓
push revert
   ↓
Argo detects Git change
   ↓
previous good image restored
   ↓
Pods Ready
   ↓
application recovered
```

No manual image change was required in Kubernetes.

This is one of the key GitOps lessons from the project.

---

# 52. Useful Commands

Check the Kubernetes node:

```bash
kubectl get nodes
```

Check application Pods:

```bash
kubectl get pods -n ecommerce-dev
```

Check Product Service Pods:

```bash
kubectl get pods \
  -n ecommerce-dev \
  -l app=product-service
```

Check Deployments:

```bash
kubectl get deployments -n ecommerce-dev
```

Check Services:

```bash
kubectl get svc -n ecommerce-dev
```

Check HPA:

```bash
kubectl get hpa -n ecommerce-dev
```

Check Argo CD:

```bash
kubectl get application ecommerce-dev -n argocd
```

Check rollout:

```bash
kubectl rollout status \
  deployment/product-service \
  -n ecommerce-dev
```

Validate Kustomize:

```bash
kubectl kustomize envs/dev > /dev/null \
  && echo "Kustomize OK"
```

---

# 53. Troubleshooting a Failed Pod

Start with:

```bash
kubectl get pods -n ecommerce-dev
```

Then inspect the Pod:

```bash
kubectl describe pod <pod-name> -n ecommerce-dev
```

Then check logs:

```bash
kubectl logs <pod-name> -n ecommerce-dev
```

The general troubleshooting flow is:

```text
Is Pod running?
      ↓
Is Pod Ready?
      ↓
Describe Pod
      ↓
Check Events
      ↓
Check logs
      ↓
Check probes
      ↓
Check dependencies
      ↓
Check configuration/secrets
```

---

# 54. Troubleshooting Argo CD

Check:

```bash
kubectl get application ecommerce-dev -n argocd
```

If:

```text
OutOfSync
```

check whether Git contains a newer desired state.

If:

```text
Progressing
```

check whether Deployments or Pods are still trying to become healthy.

If Pods are failing, inspect:

```bash
kubectl get pods
kubectl describe pod
kubectl logs
```

Argo tells us deployment status, while Kubernetes commands help explain the underlying resource problem.

---

# 55. Troubleshooting Monitoring

Check monitoring Pods:

```bash
kubectl get pods -n monitoring
```

Check ServiceMonitors:

```bash
kubectl get servicemonitor -n ecommerce-dev
```

Check PrometheusRules:

```bash
kubectl get prometheusrule -n ecommerce-dev
```

If a dashboard has no data, verify:

```text
application exposes /metrics
Service has correct labels
Service port is named correctly
ServiceMonitor matches labels
Prometheus target is UP
```

---

# 56. Important GitOps Rule

Avoid manually fixing application versions in Kubernetes with commands such as:

```text
kubectl set image ...
```

when Git is supposed to be the source of truth.

Otherwise:

```text
Cluster state
    ≠
Git state
```

and Argo may later overwrite the manual change.

The preferred flow is:

```text
Change Git
    ↓
Commit
    ↓
Push
    ↓
Argo
    ↓
Kubernetes
```

---

# 57. Application Repository vs GitOps Repository

## `ecommerce-app`

Responsible for:

```text
application source code
Dockerfiles
tests
GitHub Actions
k6 load test
application documentation
```

## `ecommerce-gitops`

Responsible for:

```text
Kubernetes configuration
deployment state
Kustomize
Argo CD
HPA
monitoring
logging
alerts
Sealed Secrets
Git-based rollback
```

Together they form the complete CI/CD and GitOps system.

---

# 58. Complete Project Flow

```text
                    Developer
                        |
                        | git push
                        v
                  ecommerce-app
                        |
                        v
                  GitHub Actions
                        |
            +-----------+-----------+
            |                       |
            v                       v
          Tests                 Docker Build
                                    |
                                    v
                         GitHub Container Registry
                                    |
                                    v
                          ecommerce-gitops
                                    |
                                    v
                                Argo CD
                                    |
                                    v
                              Kubernetes
                                    |
       +----------------------------+------------------------+
       |                            |                        |
       v                            v                        v
Application Services              HPA                Sealed Secrets
       |
       |
       +-----------------------------+
       |
       v
   Prometheus
       |
       +-------------+
       |             |
       v             v
    Grafana      Alertmanager


Application Pods
       |
       | logs
       v
     Alloy
       |
       v
      Loki
       |
       v
    Grafana
```

---

# 59. What This Repository Demonstrates

This repository demonstrates how application infrastructure can be managed through Git.

The major concepts are:

```text
Declarative Kubernetes configuration
Kustomize
GitOps
Argo CD
Automated image updates
Autoscaling
Monitoring
Centralized logging
Alerting
Secure secret storage
Self-healing
Git-based rollback
```

The main principle is:

```text
Git defines what should run.
Argo CD makes Kubernetes match Git.
Kubernetes keeps the workloads running.
Monitoring tells us how the system is behaving.
```

---

# 60. Final Summary

The `ecommerce-gitops` repository is the deployment and operations side of the E-Commerce project.

The complete operational flow is:

```text
Code change
    ↓
CI
    ↓
Container image
    ↓
GitOps update
    ↓
Argo CD
    ↓
Kubernetes
    ↓
Application
    ↓
Prometheus + Loki
    ↓
Grafana + Alertmanager
```

Security is handled using Sealed Secrets.

Scaling is handled using HPA.

Kubernetes provides self-healing.

Failures are detected using monitoring and health checks.

Deployment recovery can be performed by reverting Git instead of manually changing the cluster.

This makes the project a practical example of a complete local DevOps and GitOps workflow.
