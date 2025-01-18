# [Backend](https://github.com/dungpham91/devops.demo.backend)

![backend-result.jpg](../images/backend/backend-result.jpg)

I use Node.js and Express library to create Backend application, which provides API to Frontend application.

> Please note that I am not a frontend, backend or fullstack expert. I just google and do it.

What does this application do?

- Every 5 minutes, it will call this public API to get information https://api.blockcypher.com/v1/btc/main and then save it to the PostgreSQL database

- It will provide an API `/api/btc-block` with the data retrieved from the database, the frontend application will call this API.

- And it also provides a health check path `/health` to respond to the status of the application is OK or not OK.

### Table of contents

- [1. Use in local environment](#1-use-in-local-environment)
- [2. Automatically build and push to ECR with GitHub Actions](#2-automatically-build-and-push-to-ecr-with-github-actions)
  - [2.1 Update secrets for GitHub Actions](#21-update-secrets-for-github-actions)
  - [2.2 GHA workflow explained](#22-gha-workflow-explained)

## 1. Use in local environment

To use this application, you will need to install Docker (or Docker Desktop if you are using Windows + WSL) and Nodejs, NPM.

You can easily google how to install the above tools.

Now run the following command to run the app:

```sh
docker-compose up -d
```

Your terminal will look like this.

![backend-local-01.jpg](../images/backend/backend-local-01.jpg)

Then, you access the address http://localhost:4000/api/btc-block on your browser and the app will appear as below.

![backend-local-02.jpg](../images/backend/backend-local-02.jpg)

Once you're done working locally, you can turn compose down.

```sh
docker-compose down
```

## 2. Automatically build and push to ECR with GitHub Actions

The app is set up to automatically build and push to private ECR with GitHub Actions. You can see the [workflow file here](https://github.com/dungpham91/devops.demo.backend/blob/main/.github/workflows/main.yml).

> An important note is that you should only create tags to push images to ECR, after [Terraform](https://github.com/dungpham91/devops.demo.terraform) has finished creating the infrastructure. Because if you push images without ECR, it will fail.

### 2.1 Update secrets for GitHub Actions

You will need to update the values ​​for the GitHub Actions secrets as shown in the image below.

![secret-gha.jpg](../images/backend/secret-gha.jpg)

There are 2 values ​​to note:

- **`AWS_REGION`**: Region where you are creating the system, for example `ap-southeast-1`.

- **`ECR_REPOSITORY`**: The URI of the ECR backend repository, which is in the form `xxxxxxxxxxxx.dkr.ecr.ap-southeast-1.amazonaws.com/devopslite-be` where `xxxxxxxxxxxx` is the 12-digit ID of your AWS account.

### 2.2 GHA workflow explained

Workflow is divided into 2 main streams:

- PRs or commits directly to the main branch will perform job scans and then send notifications to Slack.

![build-backend-01.jpg](../images/backend/build-backend-01.jpg)

- Tags created in the format `vXX.XX.XX` (e.g. `v0.0.1` or `v.1.10.1`) will perform job scans, then build Docker images, push to private ECR repositories, and finally send notifications to Slack.

![build-backend-02.jpg](../images/backend/build-backend-02.jpg)

After GHA finishes running, you can go to ECR to check the uploaded image.

![build-backend-03.jpg](../images/backend/build-backend-03.jpg)

![build-backend-04.jpg](../images/backend/build-backend-04.jpg)

The backend app will perform the following scans to ensure security.

- **`Unit Tests`**: I wrote a short [Unit Tests file](https://github.com/dungpham91/devops.demo.backend/blob/main/index.test.js) to run the test, demo for the actual flow.

- **`SCA scan with Snyk`**: Scanning libraries as well as static code, Snyk can be used for many purposes, even SAST.

- **`SAST scan with CodeQL`**: I use CodeQL in addition to Snyk for SAST scanning, ensuring results when running through different tools, detecting security vulnerabilities better.

- **`Container image scan with Trivy`**: I use Trivy to scan Docker images for security vulnerabilities before pushing them to ECR.

Once the image is pushed to ECR, it will be placed on the [Helm chart](https://github.com/dungpham91/devops.demo.argocd/tree/main/apps/backend) and automatically deployed to the [EKS cluster using ArgoCD](https://github.com/dungpham91/devops.demo.argocd/blob/main/env/dev/templates/backend.yaml).
