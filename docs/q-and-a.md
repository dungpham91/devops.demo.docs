### 1. Kubernetes Deployment

```sh
Scenario:
    Deploying a highly available web application on a Kubernetes cluster. The application consists of: a frontend service, a backend service, a PostgreSQL database

 Requirements:
 1. Write a set of Kubernetes YAML manifests for deploying the application. Include:
   - Deployments for each service
   - Services for frontend and backend
   - A ConfigMap or Secret for managing database credentials securely
 2. Ensure high availability by distributing pods across multiple nodes and specifying appropriate
 resource requests and limits.
 3. Implement a basic health check for the backend service
```
---

- App frontend: https://github.com/dungpham91/devops.demo.frontend

- App backend: https://github.com/dungpham91/devops.demo.backend

- Helm chart: https://github.com/dungpham91/devops.demo.argocd/tree/main/apps

### 2. CI/CD Pipeline

```sh
Scenario:
    Design a CI/CD pipeline for deploying the above Kubernetes application

 Requirements:
 1. Write a pipeline script (in a tool of your choice: GitHub Actions, GitLab CI, Jenkins, etc.) that:
   - Builds Docker images for the frontend and backend services.
   - Runs unit tests for the backend service.
   - Deploys the application to a Kubernetes cluster.
 2. Implement a rollback strategy for failed deployments
```
---

- Frontend CI: https://github.com/dungpham91/devops.demo.frontend/actions/runs/12833916865

- Backend CI: https://github.com/dungpham91/devops.demo.backend/actions/runs/12828379197

- Unit Tests backend:
    - https://github.com/dungpham91/devops.demo.backend/blob/main/index.test.js

    - https://github.com/dungpham91/devops.demo.backend/actions/runs/12828379197/job/35772442150

- ArgoCD:
    - https://github.com/dungpham91/devops.demo.argocd/blob/main/env/dev/templates/frontend.yaml

    - https://github.com/dungpham91/devops.demo.argocd/blob/main/env/dev/templates/backend.yaml

### 3. Infrastructure as Code

```sh
Scenario
    Setting up an AWS environment for hosting the Kubernetes cluster

Requirements:
 1. Write a Terraform script to provision:
   - An EKS cluster with at least 2 nodes.
   - A VPC with subnets in multiple availability zones
 2. Document how to extend this setup to handle multi-region deployments
```
---

- Terraform repo: https://github.com/dungpham91/devops.demo.terraform

- Terraform Scan: https://github.com/dungpham91/devops.demo.terraform/actions/runs/12828224437/job/35771812449

- Terraform CI: https://github.com/dungpham91/devops.demo.terraform/pull/5

To deploy multi-region, I will go through a few things:

- My terraform repo has been set up according to the multi-environment or multi-region mechanism. We can create corresponding folders and install modules for that region. Without having to rewrite the module many times.

- Multi region means multi VPC, we only have 2 options: VPC-Peering or Transit Gateway. Depending on each situation we consider, it can be cost, it can be transmission, it can be security... We are required to connect VPCs in regions together and consider them as a private pool to use.

- If we use EKS, we will have many clusters in many regions, we must use the Route53 + AWS Global Accelerator + ALB service set.
    - In front of each EKS cluster is an ALB, AWS Global Accelerator and Route53 will be responsible for directing services to ALBs located in these regions.
    - In case a region dies, AWS Global Accelerator will automatically redirect all traffic to the surviving region.
    - We can also easily disconnect a region from the system to perform an upgrade, without affecting the service.

- The fourth point to consider is the database issue.
    - In this demo, we require using PostgreSQL as an app to install on the cluster.
    - In practice, we will prioritize using RDS multi-region with Reader replicate instances created in the second region and beyond.
    - Besides using multi-az, multi-region RDS will also help the database be more secure. But there must always be backups.
    - However, a problem that I have encountered several times, everything in the infrastructure can support multi-region, but the user application cannot. Because it is not designed to call multiple endpoints or handle endpoint flow.

- This is also the fifth point I want to mention. The application for users is compatible with multi-region architecture, stream processing, transaction/session processing by region, running on stateless model,...

- The last point to mention is storage. Although AWS provides services such as EFS and S3, that is not enough. For example, blockchain nodes using IPFS often use EFS, but EFS is not easy to share between clusters in regions (we are not talking about opening all policies to facilitate sharing). Therefore, to deploy multi-region, we also need to consider how the application will organize storage. Which objects are accessed frequently, which objects are less, which objects need to be placed where, what is the sync mechanism, is real-time needed, etc.

### 4. Monitoring and Troubleshooting

```sh
Scenario:
    A backend service running on the Kubernetes cluster has degraded performance. CPU usage is
consistently above 85%, and response times have increased.

Requirements:
 1. Outline a step-by-step approach to diagnose and resolve the issue.
 2. Suggest a monitoring stack (e.g., Prometheus, Grafana) and demonstrate how to set up alerts for
high CPU usage
```
---

1. From my experience with systems for more than 10 years, to investigate an incident, remember the order 1 - 2 - 3 as follows:
- What we have
- What we see
- What we know

Specifically, I will go into the above situation.

- **`What we have`**:
    - We need to know what services are running on the system and more importantly, what monitoring, warning and security tools we have.

    - The first thing when an incident occurs, we should immediately check if warnings appear, it helps us identify the problem quickly.

    - Is there an alert from AWS that there is an outage, is there an alert from Prometheus-Grafana related to EKS that the node is overloaded, is there an alert from Wazuh about a DoS attack, is there an alert from PRTG about network connectivity being slow (e.g. on-premises). These will have an immediate effect.

- **`What we see`**:
    - If we don't have any alerts, what can we see now?

    - Is there access to the application to see what the specific response is? Does the Dev side have an endpoint to check the application status? Is there a problem with the monitoring system that sends false alerts?

    - The next step is to see: in this case, we need to see the status of the backend svc, we can see the backend pod log, we can see the backend resource graph, we can see the node resource graph, see the network graph... we need to see anything we can see to summarize the information. After the alert is not active.

- **`What we know`**:
    - Once we have seen the information, we have learned something. At least.

    - For example in this case:
        - We can increase resources if requests/limits are not set correctly.

        - We can increase the number of pods by adjusting HPA, reducing CPU load for existing pods.

        - We can upgrade the application if it is a version error.

        - We can expand the cluster (add nodes) if it is due to increased (valid) traffic, for example during Christmas season.

    - In short, we rely on what we see to get what we need to know. and from there we can have appropriate solutions/actions.

2. I have installed Prometheus - Grafana stack with this Helm chart: https://github.com/dungpham91/devops.demo.argocd/tree/main/infra-components/kube-prometheus-stack

- To install, we use this ArgoCD app file: https://github.com/dungpham91/devops.demo.argocd/blob/main/env/dev/templates/prometheus.yaml

- As for alert settings, Prometheus has a built-in set of rules located in this directory: https://github.com/dungpham91/devops.demo.argocd/tree/main/infra-components/kube-prometheus-stack/templates/prometheus/rules-1.14. We can create additional necessary custom rule files here, then restart the Prometheus service on ArgoCD.

- Besides, we can also create alert rules on Grafana, as I have demoed. But its disadvantage is that it does not save the rules to the git repository like Prometheus, moving rules between Grafana environments (for example from Dev to QA) will take more time. Creating rules and putting them into Prometheus's chart is still better, but takes more time.

### 5. Blockchain Infrastructure

```sh
Scenario:
    Deploy and monitor a validator node for a PoS blockchain (e.g., Ethereum, Cosmos).

Requirements:
 1. Provide a high-level architecture for setting up the validator node.
 2. List best practices for ensuring its security and uptime
```
---

1. I've installed Rocket Pool before so I'll pull that out for explanation.

    - Rocket Pool is a tool that allows us to install a validator node to stake ETH, we will receive rETH rewards.

    -   In principle, to stack and become a validator node, we need 32 ETH. But this number is still large for some individuals/small units.

    -   Rocket Pool allows us to create a pool with only 16 ETH or a mini pool with 8 ETH. This is a sharing mechanism between validator nodes that Rocket Pool develops.

    -   After we install it, select the network and start the pool. We will need to register the node to become a validator, we will need to contact Rocket Pool's support channel (Discord) so that they can provide a test token and an (agreed identifier) ​​in the network.

    -   Rocket Pool has a built-in exporter and displays an integrated dashboard using Grafana (that is, they only integrate Prometheus + Exporter + Grafana, pre-design dashboards and queries to get indexes).

    -   One of the most concerned issues when installing a validator node is the penalty for dropping out of the network (for example, the node dies, the network has problems, the hard drive is full, etc.). Rocket Pool also has a mechanism to support this, avoiding the validator being unfairly penalized.

    -   In addition, testing the validator on the testnet is often impossible because there are no peers on the network. That is also a disadvantage that I think most networks have.

    -   Another issue of concern is that high gas fees make the rETH receiving rate lower, while the cost of running a validator node (not including ETH used for stacking) is already very high. Organizations will almost always look for on-premise installation to reduce server and network costs.

Back to the point, Rocket Pool is also an ETH staking solution, so the above will all make sense for other equivalent solutions.

2. Regarding solutions to ensure security and uptime for it, I have presented a lot above and in the [General readme](../README.md).

To be safe, we need to set up multi-layer security, minimizing human impact (including internal staff) on the system.

Always have tools to support network security. Without it, we are almost blind because we can't see anything.

Regarding uptime, there are multi a-z, multi region, HA, redunancy models. If we can use any model, we use that model. This depends on the organization's budget.

> I'm currently writing [Ansible](https://github.com/dungpham91/devops.demo.ansible) to install Erigon node with CDK, but it's not finished and I don't have enough time and money. I'll just note it here but not include it in the demo.
