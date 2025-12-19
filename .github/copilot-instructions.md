# GCP CI/CD Demo - AI Agent Instructions

## Architecture Overview
This is a simple containerized Flask web application deployed to Google Cloud Platform (GCP) using Cloud Build for CI/CD.

- **Main Application**: `app.py` - Basic Flask app serving "Hello from GCP CI/CD Pipeline 🚀" on port 8080
- **Containerization**: `Dockerfile` uses Python 3.9-slim base image
- **Deployments**: Separate Kubernetes deployments for dev (`deployment-dev.yaml` - 1 replica) and prod (`deployment-prod.yaml` - 2 replicas)
- **CI/CD**: `cloudbuild.yaml` orchestrates build, push to Google Container Registry (GCR), and deployments to GKE clusters

## Key Workflows
- **Build Process**: Cloud Build builds Docker image tagged with `$SHORT_SHA`, pushes to `asia-south1-docker.pkg.dev/$PROJECT_ID/my-test-docker-repo/app:$SHORT_SHA`
- **Deployment Flow**: Deploy to dev cluster first, then manual approval via `gcloud builds submit` (waitFor: ['-']), then deploy to prod cluster
- **Environment Separation**: Dev uses `dev-cluster` in `asia-south1-a` zone, Prod uses `prod-cluster2` in same zone

## Project Conventions
- **Image Naming**: Use GCR in `asia-south1` region with project-specific repo `my-test-docker-repo`
- **Deployment Placeholders**: Use `IMAGE_NAME` placeholder in YAML files; Cloud Build replaces with actual image tag
- **Cluster Configuration**: Specify `CLOUDSDK_COMPUTE_ZONE` and `CLOUDSDK_CONTAINER_CLUSTER` env vars for kubectl operations
- **Manual Approvals**: Implement production deployments with `waitFor: ['-']` step for human intervention

## Integration Points
- **GCP Services**: Relies on Cloud Build, GKE, GCR; project ID `cicd-demo-project-481705`
- **No External APIs**: Pure web app with no database or external service dependencies
- **Port Exposure**: App listens on 8080; ensure containerPort matches in deployments

## Development Patterns
- **Local Testing**: Run `python app.py` directly (Flask dev server) or build Docker image for container testing
- **No Tests**: Project lacks automated tests; validate manually via HTTP requests to deployed endpoints
- **Configuration**: Hardcoded values in YAML; no environment-specific config files