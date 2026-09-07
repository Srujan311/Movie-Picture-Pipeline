# CI/CD Workflows

This directory contains four GitHub Actions workflows for the Movie Picture
Pipeline project:

| File                | Workflow name                        | Trigger                                             |
| ------------------- | ------------------------------------- | ---------------------------------------------------- |
| `frontend-ci.yaml`   | Frontend Continuous Integration       | PR to `main` touching `starter/frontend/**`, manual  |
| `backend-ci.yaml`    | Backend Continuous Integration        | PR to `main` touching `starter/backend/**`, manual   |
| `frontend-cd.yaml`   | Frontend Continuous Deployment        | Push to `main` touching `starter/frontend/**`, manual|
| `backend-cd.yaml`    | Backend Continuous Deployment         | Push to `main` touching `starter/backend/**`, manual |

## Required GitHub Secrets

Configure these under **Settings → Secrets and variables → Actions → Secrets**
before running the CD workflows. No AWS credentials are ever hard-coded in the
workflow files themselves — they are only referenced via `secrets.*`.

| Secret name             | Description                                                        |
| ------------------------ | ------------------------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`      | Access key for the `github-action-user` IAM user (see project setup)|
| `AWS_SECRET_ACCESS_KEY`  | Secret key for the same IAM user                                    |

## Optional repository variable

| Variable name                | Description                                                                                     |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| `REACT_APP_MOVIE_API_URL`     | Backend URL baked into the frontend build at deploy time. Defaults to `http://localhost:5000` if unset — set this to the backend's real (LoadBalancer) URL for a working deployed frontend. |

## What each workflow does

- **CI workflows** run `lint` and `test` jobs in parallel, then a `build` job
  (gated with `needs`) that builds the Docker image to confirm it compiles.
  Nothing is pushed anywhere — these are verification-only. A final
  `comment-status` job posts a ✅/❌ summary comment back on the pull request
  so a reviewer doesn't have to open the Actions tab.
- **CD workflows** run the same `lint`/`test` jobs in parallel, then a
  `build-and-push` job (gated with `needs`) that authenticates to ECR via
  `aws-actions/amazon-ecr-login`, builds the image tagged with the commit SHA
  (`github.sha`), and pushes it to ECR. A final `deploy` job updates the
  kubeconfig for the EKS cluster, uses `kustomize edit set image` to point the
  manifests at the freshly pushed image tag, applies them with `kubectl`, and
  waits for the rollout to complete.

## Shared composite actions

`.github/actions/frontend-deps` and `.github/actions/backend-deps` wrap the
setup/cache/install steps that would otherwise be copy-pasted into every
lint/test/build job across all four workflows. Each workflow just does
`actions/checkout` followed by `uses: ./.github/actions/<name>-deps`.
