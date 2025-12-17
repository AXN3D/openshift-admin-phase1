\# OpenShift Admin – Phase 1

Learning Kubernetes and OpenShift administration using OpenShift Sandbox.



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







