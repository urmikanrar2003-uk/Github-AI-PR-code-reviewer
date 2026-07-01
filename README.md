# AI Pull Request Code Reviewer

An automated, event-driven AI Code Reviewer built on a robust microservices architecture. This GitHub App automatically listens to Pull Request events, analyzes code changes using OpenAI, and posts intelligent review comments directly on GitHub.

🎥 **[View the Project Demo here](https://drive.google.com/drive/folders/1KPLy--tNL6VF1PLOtBdEMs5PbvGfSKeE?usp=sharing)**

## 🏗️ Architecture

*(Please see **`final-diagram.mmd`** in this repository for the complete visual workflow diagram.)*

The system is built using a highly decoupled, event-driven microservices architecture running on Kubernetes (AWS EKS):

- **Webhook Service:** Receives webhook payloads directly from GitHub when a PR is opened or synchronized, and pushes the event to RabbitMQ.
- **Gateway Service:** An API router that manages traffic and routing.
- **Orchestrator:** Consumes events from the message broker, coordinates the review process, and delegates tasks.
- **Reviewer:** Uses the OpenAI API to analyze code diffs and generate intelligent, context-aware comments.
- **Learner & Learner-Worker:** Analyzes past PRs to continuously improve the review context and accuracy.
- **Evaluate CI Job:** A GitHub Action workflow that uses `ragas` to automatically evaluate the LLM's faithfulness and answer relevancy against historical findings.

## 💻 Tech Stack

- **AI & LLM Frameworks:** LangGraph (Core orchestration framework), LangChain, OpenAI API, Ragas (for RAG pipeline evaluation)
- **LLM Observability:** Langfuse (LLM tracing and metrics)
- **Cloud & Infrastructure:** AWS EKS (Elastic Kubernetes Service), AWS RDS (PostgreSQL), AWS ElastiCache (Redis), AWS ECR, Terraform
- **Containerization & Orchestration:** Docker, Kubernetes, Kubectl, Helm
- **Backend & Async Processing:** Python, Celery (Distributed task queue), RabbitMQ (Message broker)
- **Database & ORM:** PostgreSQL, SQLAlchemy (ORM), Alembic (Database migrations)
- **Security & Authentication:** PyJWT (JSON Web Tokens), Cryptography (HMAC SHA-256 for GitHub webhook payload verification)
- **CI/CD:** GitHub Actions

## 🚧 AWS Free Tier Limitations & Workarounds

This project was intentionally engineered to run entirely within the **AWS Free Tier**, which presented extreme resource constraints that required creative engineering solutions:

- **Strict vCPU Quotas:** AWS limits new Free Tier accounts to a hard cap of 16 vCPUs for On-Demand instances. This capped the cluster at exactly 9 `t3.micro` nodes (which have 2 vCPUs each, with one node failing to launch due to hitting the quota). 
- **Pod Slot Exhaustion:** A `t3.micro` instance is restricted to a maximum of 4 pods by the default AWS VPC CNI ENI limits. Across 9 nodes, the cluster had exactly 36 pod slots. With 22 Kubernetes system pods running, this left exactly 14 slots for applications. 
- **Graceful Degradation:** To maintain High Availability (2 replicas) for core microservices while leaving room for CI/CD Rolling Updates and the `evaluate` Job to run, the `gateway` microservice and the `aws-load-balancer-controller` were intentionally scaled down to 1 replica to free up exact pod slots.
- **CSI Driver Removal:** The `aws-ebs-csi-driver` was uninstalled because it aggressively consumes 1 pod slot per node. By removing it, 9 pod slots were immediately freed up across the cluster.
- **No Prometheus/Grafana:** While the code supports it, monitoring stacks were omitted because they require persistent volumes and significant pod slots that far exceed Free Tier capacity.

## 🚀 Future Enhancements

If this project were moved out of the Free Tier constraints or deployed to production, the following enhancements would be made:

1. **VPC CNI Prefix Delegation:** Enable `ENABLE_PREFIX_DELEGATION` on the EKS AWS VPC CNI. This assigns a `/28` IP prefix to each ENI instead of a single IP, drastically increasing the max-pod limit on `t3.micro` instances from 4 to 16+, completely eliminating the pod slot bottleneck without upgrading instance sizes.
2. **Public SaaS Model:** The GitHub App is currently marked as **Private**. To launch this as a public SaaS product:
   - Change the App visibility to **Public** in the GitHub Developer settings.
   - Build a frontend dashboard where users can authenticate and input their own OpenAI API Keys (to prevent paying the OpenAI bill for external users).
   - The webhook service is already built as a multi-tenant application and can easily handle routing for thousands of different repositories simultaneously.
3. **Observability Stack:** Deploy the Prometheus & Grafana Helm charts to monitor microservice metrics and Horizontal Pod Autoscaler (HPA) scaling events in real-time.
