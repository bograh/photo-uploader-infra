# Photo Uploader — Infrastructure

AWS infrastructure for the **Photo Uploader** gallery, defined as a single
**CloudFormation** template and deployed via **CloudFormation Git sync**. Every
push to `main` that changes the template (or its deployment config) is applied
to the stack automatically — no manual `deploy` step.

Application code lives in a separate repository:
[`photo-uploader-app`](https://github.com/bograh/photo-uploader-app).

---

## Files

| File               | Purpose                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| `main.yaml`        | The full CloudFormation template (all resources)                        |
| `main-config.json` | Git sync **deployment file** — declares the template path, parameters, tags |

The deployment file uses the Git sync schema (note the mandatory
`template-file-path` key and lowercase `parameters`/`tags`):

```json
{
  "template-file-path": "main.yaml",
  "parameters": { "EnvironmentName": "photo-uploader", "GitHubOrg": "bograh", ... },
  "tags": { "Project": "PhotoUploader", "Environment": "lab" }
}
```

---

## Architecture

A highly available, single-region, containerized stack:

- **Networking** — Multi-AZ VPC with public subnets (ALB), private subnets (ECS
  tasks), and isolated data subnets (RDS). NAT gateway + **VPC endpoints** (S3,
  ECR API/DKR, CloudWatch Logs, Secrets Manager) so private tasks reach AWS
  services.
- **Compute** — **ECS on Fargate** behind a public **Application Load Balancer**,
  with CPU-based auto scaling (min 1 / desired 1 / max 4).
- **Storage** — Private **S3** bucket for images (public access fully blocked),
  served through **CloudFront** via Origin Access Control (OAC). A second S3
  bucket holds pipeline artifacts.
- **Database** — **RDS PostgreSQL** (`db.t3.micro`), credentials generated into
  **Secrets Manager**.
- **Registry & CI/CD** — **ECR** repository, **EventBridge** rule (ECR push →
  pipeline), **CodePipeline** + **CodeDeploy** blue/green onto ECS.
- **Identity** — **GitHub OIDC** federated role for CI/CD (no long-lived keys),
  plus scoped task, execution, pipeline, and deploy roles.

```
GitHub Actions ──(OIDC)──► ECR ──(EventBridge)──► CodePipeline ──► CodeDeploy (blue/green) ──► ECS Fargate
                                                                                                  │
Internet ──► ALB ──► ECS tasks ──► RDS PostgreSQL          images: S3 ──► CloudFront ──► browser
```

---

## Parameters

| Parameter          | Description                                        | Default            |
| ------------------ | -------------------------------------------------- | ------------------ |
| `EnvironmentName`  | Prefix for all resource names                      | `photo-uploader`   |
| `GitHubOrg`        | GitHub org/username (for the OIDC trust policy)    | —                  |
| `GitHubAppRepo`    | Application repository name                        | —                  |
| `InitialImageUri`  | Placeholder image for the first ECS task definition| `python:3.11-slim` |

---

## Deploying with Git sync

**One-time setup:**

1. **GitHub connection** — Developer Tools → Settings → Connections → create a
   GitHub connection and authorize the AWS Connector app for this repo.
2. **Git sync IAM role** — a role CloudFormation assumes to create resources
   (needs permissions for everything in the template).
3. **CloudFormation → Create stack → "Sync from Git"** — link this repo + `main`
   branch, set **Deployment file** = `main-config.json`, select the sync role,
   and acknowledge `CAPABILITY_NAMED_IAM`.

After that, pushing to `main` deploys changes automatically.

> **Manual deploy (fallback / testing):**
> ```bash
> aws cloudformation deploy --stack-name photo-uploader \
>   --template-file main.yaml \
>   --parameter-overrides EnvironmentName=photo-uploader GitHubOrg=bograh GitHubAppRepo=photo-uploader-app \
>   --capabilities CAPABILITY_NAMED_IAM --region eu-central-1
> ```

---

## Outputs

After the stack completes, the **Outputs** tab provides the values wired into
the app repo and used to reach the app:

| Output                | Use                                                        |
| --------------------- | ---------------------------------------------------------- |
| `ALBEndpoint`         | Public URL of the running gallery                          |
| `GitHubActionsRoleArn`| App repo secret `AWS_ROLE_ARN`                             |
| `ECRRepositoryName`   | App repo secret `ECR_REPOSITORY_NAME`                     |
| `ArtifactsBucketName` | App repo secret `ARTIFACTS_BUCKET_NAME`                   |
| `ECRRepositoryUri`    | ECR repository URI                                         |
| `CloudFrontDomain`    | Domain images are served from                              |
| `DBSecretArn`         | Secrets Manager ARN for the RDS credentials                |
| `ECSClusterName`      | ECS cluster name                                           |

---

## Notes & gotchas

- **GitHub OIDC provider is account-wide.** Only one
  `token.actions.githubusercontent.com` provider can exist per AWS account. The
  template references the existing provider by ARN rather than creating a
  duplicate — if your account has none, create one first.
- **Immutable OIDC subject claims.** GitHub may issue subjects that include
  numeric owner/repo IDs (`repo:owner@<id>/repo@<id>:...`). The CI/CD role trust
  policy accepts both the plain and immutable forms.
- **Deployment file schema.** Git sync reads the template path from
  `main-config.json`'s `template-file-path`; keys must be lowercase
  (`parameters`, `tags`). A missing/mis-cased key leaves the stack as a
  setup stub without deploying the template.
- **First deployment** runs the placeholder container until the app repo's
  pipeline performs the first real blue/green deployment.
