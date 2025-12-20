# OpenShift Admin – Phase 1: Platform Fundamentals

## Overview
This project demonstrates hands-on OpenShift administration by deploying
and operating a multi-tier application on OpenShift Sandbox.

The focus is on **platform operations and troubleshooting**, not CI/CD or DevOps tooling.


\## First OpenShift Resource



A ConfigMap was applied to the OpenShift Sandbox cluster using the `oc` CLI.

This validated local YAML → OpenShift API → running object workflow.



\## PostgreSQL CrashLoopBackOff – Root Cause Analysis



During PostgreSQL deployment, the pod entered a CrashLoopBackOff state.



\### Symptoms

\- Pod status: CrashLoopBackOff

\- Container exited immediately after start



\### Investigation

\- Checked pod logs using:

oc logs <postgres-pod>



\- Logs indicated missing required environment variables:

POSTGRESQL\_USER, POSTGRESQL\_PASSWORD, POSTGRESQL\_DATABASE



\### Root Cause

The PostgreSQL image used (`registry.redhat.io/rhel9/postgresql-15`)

expects `POSTGRESQL\_\*` environment variables.

The initial Secret incorrectly used `POSTGRES\_\*` variables.



\### Resolution

\- Updated the Secret to use correct `POSTGRESQL\_\*` keys

\- Re-applied the Secret

\- Restarted the PostgreSQL pod to pick up new values



\### Outcome

\- PostgreSQL pod started successfully

\- Persistent storage remained intact via PVC



\### Key OpenShift Learnings

\- OpenShift-certified images have strict configuration requirements

\- CrashLoopBackOff often indicates configuration issues

\- Secrets updates require pod restarts

\- Logs are the primary troubleshooting tool for failing pods

## PostgreSQL Service Without Endpoints – Root Cause

After creating the PostgreSQL Service, no endpoints were initially populated.

### Symptoms
- Service existed
- `oc get endpoints postgres` returned `<none>`

### Investigation
- Checked pod status using:
oc get pods

- No pods were running in the namespace
- Deployment was scaled to 0 replicas

### Root Cause
The PostgreSQL Deployment had been scaled down to 0 replicas,
resulting in no running pods and therefore no service endpoints.

### Resolution
- Scaled the Deployment back to 1 replica
- Verified pod readiness
- Confirmed endpoints were populated correctly

### Key Learnings
- Services only route traffic to Ready pods
- Endpoints depend on replica count and readiness
- Always verify Deployment → Pods → Endpoints in that order

---------------

## End-to-End Validation

The backend application was exposed externally using an OpenShift Route.

### Issue Encountered
- Route showed no endpoints despite Service having active endpoints

### Root Cause
OpenShift Routes bind to named Service ports.  
The backend Service used an unnamed port, which prevented the router from
resolving endpoints.

### Resolution
- Added a named port ('https') to the backend Service
- Updated the Route to reference the named port
- Verified router endpoints and browser access

### Result
End-to-end traffic flow confirmed:
Browser → OpenShift Route → Service → Pod → PostgreSQL

================================================================================================


---

## Phase 2 – OpenShift Admin Operations

Phase 2 focuses on **day-2 operational behavior**, including resource management,
security enforcement, and failure scenarios commonly encountered in production
OpenShift environments.

---

## Phase 2.1 – Resource Limits & OOMKilled

### Issue
A backend pod repeatedly entered 'CrashLoopBackOff' with the container being
terminated due to 'OOMKilled' events.

### Investigation
- Checked pod status and restart behavior
- Used `oc describe pod` and cluster events to inspect termination reason
- Observed `Reason: OOMKilled` in pod details

### Root Cause
The container exceeded its configured memory limit.
Although the pod was successfully scheduled based on its memory request,
the Linux kernel terminated the process once it exceeded the memory limit.

Increasing memory limits delayed the failure but did not prevent it because
the application continuously allocated memory with no upper bound.

### Resolution
- Adjusted memory requests and limits to appropriate values
- Restarted the pod to apply new resource settings
- Used a bounded workload to validate pod stability

### Key Learnings
- Requests affect scheduling; limits enforce runtime usage
- 'OOMKilled' is a kernel-level action, not an application crash
- Increasing limits does not fix unbounded memory consumption
- Admins must distinguish between platform misconfiguration and application behavior

---

## Repository Purpose

This repository serves as a growing knowledge base for OpenShift administration,
capturing real failures, investigations, and resolutions encountered while
operating workloads on the platform.

------------------------------------------------------------------------------------

## Phase 2.2 – SecurityContext & SCC

### Issue
A container that ran successfully in Docker/Kubernetes failed on OpenShift.

### Investigation
- Pod entered CrashLoopBackOff
- Logs showed permission denied errors when writing to system paths

### Root Cause
OpenShift enforces non-root container execution using Security Context Constraints (SCC).
The application assumed root privileges and attempted to write to restricted paths.

### Resolution
- Updated the container to run as non-root
- Modified runtime paths to writable locations
- Redeployed the application successfully

### Key Learnings
- OpenShift SCCs are stricter than vanilla Kubernetes security
- Applications must be designed to run as non-root
- Fixing apps is preferable to weakening cluster security



### Final Resolution
After repeated SCC-related failures with a generic nginx image, the deployment
was updated to use a Red Hat UBI-based httpd image designed for OpenShift.
The application ran successfully without requiring privileged access or
relaxed security constraints.

### Key Learnings
- OpenShift SCC enforcement can expose hidden assumptions in container images
- Not all upstream images are OpenShift-compatible
- Selecting OpenShift-certified images is often the correct administrative fix

----------------------------------------------------------------------------------


## Phase 2.3 – Rolling Updates & Rollbacks

### Issue
A deployment update failed after a container image with an invalid tag was applied.

### Investigation
- Rollout stalled during update
- New pods failed with ImagePullBackOff
- Existing pods continued serving traffic

### Resolution
- Used `oc rollout undo` to revert to the last known good ReplicaSet
- Verified deployment stability after rollback

### Key Learnings
- OpenShift performs rolling updates, not full restarts
- Failed rollouts do not automatically cause downtime
- Rollbacks are a critical day-2 operational skill

------------------------------------------------------------------------------------


## Phase 2.4 – Image Pull Failures

### Issue
Pods failed to start with `ImagePullBackOff` when referencing a private image.

### Investigation
- Pod events showed unauthorized image pull errors
- Verified ServiceAccount configuration

### Root Cause
The ServiceAccount had no image pull credentials for the private registry.

### Resolution
- Created a docker-registry pull secret
- Linked the secret to the ServiceAccount used by the pod
- Verified authenticated pull attempts via ServiceAccount inspection

### Key Learnings
- Image pulls are performed by ServiceAccounts
- imagePullSecrets must be explicitly linked
- Persistent ImagePullBackOff may indicate invalid credentials, not misconfiguration

---------------------------------------------------------------------------------------

## Phase 2.5 – Routes & 503 Debugging

### Scenario
An application was exposed externally using an OpenShift Route backed by a
Service and Deployment.

### Observation
The Route resolved successfully and served traffic even though the Service
port was unnamed.

### Explanation
OpenShift Routes may successfully bind to a Service when there is a single,
unambiguous port definition. In such cases, the router can resolve the target
port numerically without requiring an explicit port name.

### Operational Insight
This behavior is not guaranteed in real-world scenarios. Services with multiple
ports, TLS configurations, or metrics endpoints require explicitly named ports
for predictable Route behavior.

### Best Practice
Always define named Service ports when exposing applications through OpenShift
Routes to avoid intermittent 503 errors and router endpoint resolution issues.

### Key Learnings
- Routes bind to Services, not directly to pods
- OpenShift may auto-resolve simple single-port Services
- Explicit port naming is required for reliable routing in multi-port Services

=======================================================================================


## Phase 3.1 – Projects & RBAC (Least Privilege)

### Scenario
Project-level access control was evaluated to enforce least-privilege access
within an OpenShift Project.

### Investigation
- Reviewed existing RoleBindings in the project
- Identified that the primary user had admin privileges by default
- Observed that binding a restrictive Role to an admin user did not reduce access

### Key Observation
RBAC permissions in OpenShift are **additive**, not restrictive.
A user with admin privileges retains full access even when bound to a
more restrictive Role.

### Correct Validation Method
To accurately test RBAC behavior, a ServiceAccount was used instead of a user.

### Action
- Created a custom Role allowing read-only access to pods and services
- Created a ServiceAccount for RBAC testing
- Bound the custom Role to the ServiceAccount
- Tested permissions using `--as=system:serviceaccount`

### Validation
- Read operations (get/list pods and services) succeeded
- Write operations (creating deployments) were denied with `Forbidden` errors

### Key Learnings
- RBAC permissions are additive and cannot revoke existing access
- Admin users cannot self-restrict using RBAC
- ServiceAccounts are the correct mechanism for validating least-privilege access
- `Forbidden` errors indicate correct RBAC enforcement



