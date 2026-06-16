# Implementation Plan: k8s-cicd-pipeline

## Overview

This plan implements a GitHub Actions CI/CD pipeline for deploying Termix to Kubernetes. Tasks are ordered so each step builds on the previous: first the K8s manifests are created (the deployment target), then the workflow file is built job-by-job following the sequential pipeline stages (validate → build → push → deploy → verify → rollback).

## Tasks

- [x] 1. Create Kubernetes manifest files
  - [x] 1.1 Create namespace and ConfigMap manifests
    - Create `k8s/namespace.yml` defining the `termix` namespace
    - Create `k8s/configmap.yml` defining a ConfigMap named `termix-config` in namespace `termix` with keys: PORT, NODE_ENV, GUACD_HOST, ENABLE_SSL, BASE_PATH
    - _Requirements: 6.1, 7.1_

  - [x] 1.2 Create PersistentVolumeClaim manifest
    - Create `k8s/pvc.yml` with name `termix-data`, namespace `termix`, accessMode `ReadWriteOnce`, storage request `1Gi`
    - _Requirements: 7.3_

  - [x] 1.3 Create Deployment manifest
    - Create `k8s/deployment.yml` in namespace `termix` with:
      - 2 replicas
      - Container name `termix`, image `docker.io/DOCKER_HUB_USERNAME/termix:PIPELINE_IMAGE_TAG`
      - Container ports: 8080 (http), 30001 (health)
      - Resource requests: 128Mi memory, 100m CPU
      - Resource limits: 512Mi memory, 500m CPU
      - Liveness probe: HTTP GET `/health` port 30001, initialDelaySeconds 60, periodSeconds 30, timeoutSeconds 10, failureThreshold 3
      - Readiness probe: same config as liveness
      - `envFrom` referencing ConfigMap `termix-config` and Secret `termix-secrets` (optional: false)
      - Volume mount: `termix-data` PVC at `/app/data`
    - _Requirements: 6.1, 6.3, 6.4, 6.5, 6.6, 7.2, 7.3, 7.4_

  - [x] 1.4 Create Service manifest
    - Create `k8s/service.yml` defining a ClusterIP Service named `termix` in namespace `termix`
    - Port 8080 → targetPort 8080, selector matching the Deployment pod labels
    - _Requirements: 6.2_

- [x] 2. Checkpoint - Validate Kubernetes manifests
  - Ensure all manifests are syntactically valid YAML. Run `kubectl apply --dry-run=client -f k8s/` if possible. Ask the user if questions arise.

- [x] 3. Create GitHub Actions workflow — trigger and validate job
  - [x] 3.1 Create workflow file with triggers and validate job
    - Create `.github/workflows/k8s-deploy.yml` with:
      - `on.push.branches: [main]` trigger
      - `on.workflow_dispatch` with inputs: `version` (string, required), `build_type` (choice: Development/Production, required), `rollback` (boolean, default false)
    - Add `validate` job (runs-on: ubuntu-latest) that:
      - Checks that the workflow is running on the `main` branch, fails with message "releases must be run from main branch" otherwise
      - If `rollback` input is true, skips build/push/deploy and jumps to rollback job
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [x] 4. Add build and push jobs to workflow
  - [x] 4.1 Add build job
    - Add `build` job (needs: validate) that:
      - Checks out the repository
      - Builds Docker image from `docker/Dockerfile` with repo root as context
      - Tags with first 7 chars of `${{ github.sha }}` and `latest` (when on main)
      - Uploads image as artifact or uses Docker layer caching for the push job
      - Fails workflow with build error on failure
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [x] 4.2 Add push job
    - Add `push` job (needs: build) that:
      - Condition: only runs when `build_type == 'Production'`
      - Logs into Docker Hub using `DOCKER_HUB_USERNAME` and `DOCKER_HUB_PASSWORD` secrets
      - Pushes all tagged images to Docker Hub
      - Skips entirely for Development builds
      - Fails with authentication error message on login failure
      - Fails with push error message on push failure
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [x] 5. Add deploy and verify jobs to workflow
  - [x] 5.1 Add deploy job
    - Add `deploy` job (needs: push) that:
      - Decodes `KUBE_CONFIG` secret (base64) and writes kubeconfig
      - Verifies kubectl connectivity to the cluster
      - Applies all manifests: `kubectl apply -f k8s/`
      - Updates container image: `kubectl set image deployment/termix termix=docker.io/${{ secrets.DOCKER_HUB_USERNAME }}/termix:<sha7> -n termix`
      - Fails with deployment error on manifest apply or set image failure
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [x] 5.2 Add verify job
    - Add `verify` job (needs: deploy) that:
      - Runs `kubectl rollout status deployment/termix -n termix --timeout=300s`
      - Gets pods with label selector, checks Ready count equals replica count (2)
      - Verifies all pods are running the expected image tag
      - Fails on timeout, pod count mismatch, or wrong image
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

- [x] 6. Add rollback job to workflow
  - [x] 6.1 Add rollback job
    - Add `rollback` job that triggers:
      - Automatically when verify job fails (`if: failure()`)
      - Manually when `rollback` input is true (skipping build/push/deploy)
    - Implementation:
      - Decodes `KUBE_CONFIG` and configures kubectl
      - Checks that a previous revision exists, fails with "no previous revision" if not
      - Runs `kubectl rollout undo deployment/termix -n termix`
      - Waits up to 120 seconds for rollback completion
      - Reports revision restored and success/failure status
      - Fails workflow if rollback times out or command fails
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

- [x] 7. Final checkpoint - Validate complete workflow
  - Ensure the workflow file is valid YAML and all job dependencies are correct. Ensure all tests pass, ask the user if questions arise.

## Notes

- No property-based tests apply — this feature is entirely declarative IaC and CI/CD configuration
- Each task references specific requirement acceptance criteria for traceability
- Checkpoints ensure manifests and workflow are validated incrementally
- The `k8s/` manifests use placeholder values (DOCKER_HUB_USERNAME, PIPELINE_IMAGE_TAG) that are substituted at deploy time by the pipeline
- Repository secrets (DOCKER_HUB_USERNAME, DOCKER_HUB_PASSWORD, KUBE_CONFIG) must be configured in GitHub before the pipeline can run successfully

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.4"] },
    { "id": 1, "tasks": ["1.3"] },
    { "id": 2, "tasks": ["3.1"] },
    { "id": 3, "tasks": ["4.1"] },
    { "id": 4, "tasks": ["4.2"] },
    { "id": 5, "tasks": ["5.1"] },
    { "id": 6, "tasks": ["5.2"] },
    { "id": 7, "tasks": ["6.1"] }
  ]
}
```
