# Requirements Document

## Introduction

This feature adds a GitHub Actions CI/CD pipeline that automatically builds a Docker image of the Termix web application, pushes it to Docker Hub, and deploys it to a Kubernetes cluster. The pipeline integrates with the existing Docker workflow and leverages repository secrets for authentication with Docker Hub and the Kubernetes cluster.

## Glossary

- **Pipeline**: The GitHub Actions workflow that orchestrates the build, push, and deploy stages
- **Docker_Image**: The container image produced from the Termix Dockerfile located at `docker/Dockerfile`
- **Docker_Hub_Registry**: The Docker Hub container registry where built images are pushed, authenticated via DOCKER_HUB_USERNAME and DOCKER_HUB_PASSWORD secrets
- **Kubernetes_Cluster**: The target Kubernetes cluster where the application is deployed, accessed via KUBE_CONFIG (base64-encoded kubeconfig) secret
- **Deployment_Manifest**: The Kubernetes YAML manifest(s) defining the Deployment, Service, and related resources for Termix
- **Health_Check**: A verification step that confirms the deployed application is running and responsive in the cluster
- **Image_Tag**: The tag applied to the Docker image, derived from the Git SHA or semantic version

## Requirements

### Requirement 1: Pipeline Trigger

**User Story:** As a developer, I want the deployment pipeline to run automatically on pushes to the main branch, so that production deployments are consistent and hands-free.

#### Acceptance Criteria

1. WHEN a push event occurs on the main branch, THE Pipeline SHALL trigger the build, push, and deploy stages in sequential order where each stage begins only after the previous stage completes successfully
2. WHEN a workflow_dispatch event is triggered manually, THE Pipeline SHALL require a version input (non-empty string, e.g., "1.8.0") and a build_type input (one of "Development" or "Production") before execution begins
3. THE Pipeline SHALL NOT trigger on pull request events or push events to branches other than main
4. IF a workflow_dispatch event is triggered from a branch other than main, THEN THE Pipeline SHALL fail and report that releases must be run from the main branch

### Requirement 2: Docker Image Build

**User Story:** As a developer, I want the pipeline to build a Docker image from the existing Dockerfile, so that the same image used locally is deployed to production.

#### Acceptance Criteria

1. WHEN the build stage executes, THE Pipeline SHALL build a Docker image using the Dockerfile at `docker/Dockerfile` with the repository root as the build context
2. WHEN the build stage executes, THE Pipeline SHALL tag the Docker_Image with the first 7 characters of the Git commit SHA
3. IF the build executes on the main branch, THEN THE Pipeline SHALL tag the Docker_Image with `latest` in addition to the commit SHA tag
4. IF the Docker image build fails, THEN THE Pipeline SHALL fail the workflow and report the build error in the job logs
5. WHEN the build stage executes, THE Pipeline SHALL push the Docker_Image to the configured container registry

### Requirement 3: Docker Image Push

**User Story:** As a developer, I want the pipeline to push the built image to Docker Hub, so that it is available for deployment.

#### Acceptance Criteria

1. WHEN the build stage completes successfully AND the build_type is "Production", THE Pipeline SHALL authenticate with Docker_Hub_Registry using the DOCKER_HUB_USERNAME and DOCKER_HUB_PASSWORD repository secrets
2. WHEN authenticated with Docker_Hub_Registry, THE Pipeline SHALL push the Docker_Image with all tags assigned during the tagging stage to Docker_Hub_Registry within 15 minutes
3. IF authentication with Docker_Hub_Registry fails, THEN THE Pipeline SHALL fail the workflow and report an error message indicating the authentication failure
4. IF the push to Docker_Hub_Registry fails after successful authentication, THEN THE Pipeline SHALL fail the workflow and report an error message indicating the push failure
5. WHEN the build_type is "Development", THE Pipeline SHALL NOT push the Docker_Image to Docker_Hub_Registry

### Requirement 4: Kubernetes Deployment

**User Story:** As a developer, I want the pipeline to deploy the new image to the Kubernetes cluster, so that changes reach production automatically.

#### Acceptance Criteria

1. WHEN the push stage completes successfully, THE Pipeline SHALL configure kubectl using the KUBE_CONFIG secret (base64-decoded) and verify connectivity to the Kubernetes_Cluster
2. WHEN kubectl is configured, THE Pipeline SHALL apply the Deployment_Manifest to the Kubernetes_Cluster in the target namespace
3. WHEN the manifest is applied, THE Pipeline SHALL update the container image reference in the Deployment to the newly pushed Docker_Image tag using kubectl set image
4. IF applying the Deployment_Manifest fails, THEN THE Pipeline SHALL fail the workflow and report the deployment error in the job logs

### Requirement 5: Deployment Verification

**User Story:** As a developer, I want the pipeline to verify the deployment succeeded, so that I am confident the new version is running.

#### Acceptance Criteria

1. WHEN the deployment is applied, THE Pipeline SHALL wait for the Kubernetes Deployment rollout to complete using a timeout of 300 seconds
2. WHEN the rollout completes, THE Pipeline SHALL verify that the number of pods in a Ready state equals the replica count specified in the Deployment_Manifest
3. WHEN the rollout completes, THE Pipeline SHALL verify that all Ready pods are running the newly pushed Docker_Image tag
4. IF the rollout does not complete within 300 seconds, THEN THE Pipeline SHALL fail the workflow and report a timeout error in the job logs
5. IF the number of Ready pods does not equal the Deployment_Manifest replica count after the rollout completes, THEN THE Pipeline SHALL fail the workflow and report the mismatch in the job logs

### Requirement 6: Kubernetes Manifests

**User Story:** As a developer, I want Kubernetes manifests stored in the repository, so that the cluster state is version-controlled and reproducible.

#### Acceptance Criteria

1. THE Deployment_Manifest SHALL define a Kubernetes Deployment resource with a replica count defaulting to 2 and configurable via the manifest
2. THE Deployment_Manifest SHALL define a Kubernetes Service resource of type ClusterIP that routes traffic on port 8080 to the container's port 8080
3. THE Deployment_Manifest SHALL configure resource requests of 128Mi memory and 100m CPU, and resource limits of 512Mi memory and 500m CPU on the container
4. THE Deployment_Manifest SHALL include a liveness probe and a readiness probe that issue HTTP GET requests to port 30001 on path /health, with an initial delay of 60 seconds, a period of 30 seconds, a timeout of 10 seconds, and a failure threshold of 3
5. THE Deployment_Manifest SHALL reference the Docker_Image from Docker_Hub_Registry using the tag value "PIPELINE_IMAGE_TAG" as a placeholder that the Pipeline replaces with the actual Image_Tag at deploy time
6. THE Deployment_Manifest SHALL declare a container port of 8080 for application traffic and a container port of 30001 for health check traffic

### Requirement 7: Environment Configuration

**User Story:** As a developer, I want the deployment to support environment-specific configuration, so that secrets and settings are managed outside the image.

#### Acceptance Criteria

1. THE Deployment_Manifest SHALL reference a Kubernetes ConfigMap via `envFrom` to inject non-sensitive environment variables (PORT, NODE_ENV, GUACD_HOST, ENABLE_SSL, BASE_PATH) into the container
2. THE Deployment_Manifest SHALL reference a Kubernetes Secret via `envFrom` to inject sensitive environment variables (database credentials, API keys) into the container
3. THE Deployment_Manifest SHALL define a PersistentVolumeClaim with ReadWriteOnce access mode and a minimum storage capacity of 1Gi, mounted at `/app/data`, to preserve application state across pod restarts
4. IF a required Secret referenced by the Deployment_Manifest is not present in the namespace, THEN THE Deployment_Manifest SHALL prevent the pod from starting by setting all Secret-sourced environment variables as required (optional: false)

### Requirement 8: Rollback Capability

**User Story:** As a developer, I want the ability to roll back a failed deployment, so that production can be restored quickly.

#### Acceptance Criteria

1. IF the deployment verification fails, THEN THE Pipeline SHALL execute a rollback to the previous Kubernetes Deployment revision and wait no longer than 120 seconds for the rollback to complete
2. WHEN a rollback is executed, THE Pipeline SHALL report the rollback action, the revision restored, and whether the rollback succeeded or failed in the job logs
3. WHEN a manual workflow_dispatch is triggered with the rollback input set to true, THE Pipeline SHALL roll back to the previous revision without building or pushing a new image and wait no longer than 120 seconds for the rollback to complete
4. IF the rollback does not complete within 120 seconds or the rollback command fails, THEN THE Pipeline SHALL fail the workflow and report the rollback failure in the job logs
5. IF a rollback is requested and no previous Deployment revision exists, THEN THE Pipeline SHALL fail the workflow and report that no previous revision is available
