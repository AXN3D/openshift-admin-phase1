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



