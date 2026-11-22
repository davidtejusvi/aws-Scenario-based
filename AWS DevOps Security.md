# AWS DevOps Security – Simple Notes

## 1. Complete CI/CD Architecture

```text
Developer
   |
   v
GitHub Repository
   |
   v
GitHub Actions
   |
   | OIDC Authentication
   v
AWS IAM Role
   |
   | Permission to push image
   v
Amazon ECR
   |
   | Docker Image
   v
Amazon EKS
   |
   v
Kubernetes Pods
   |
   v
Application Running
```

## 2. GitHub Actions – AWS Security

How does GitHub Actions connect to AWS?

**Recommended method: OIDC**

OIDC allows GitHub Actions to access AWS without storing permanent AWS access keys.

```text
GitHub Actions
      |
      | OIDC Token
      v
AWS IAM OIDC Provider
      |
      v
AWS IAM Role
      |
      v
Temporary AWS Credentials
      |
      v
Access AWS Services
```

Example GitHub Actions permissions:

```yaml
permissions:
  id-token: write
  contents: read
```

**Remember:**
- OIDC = Secure authentication.
- IAM Role = AWS permissions.
- Temporary credentials = No permanent access keys required.

## 3. Docker Container Security

Before pushing Docker images to ECR, scan them for vulnerabilities.

```text
Application Code
      |
      v
Docker Build
      |
      v
Trivy Security Scan
      |
      v
Amazon ECR
      |
      v
Amazon EKS
```

Important security practices:

| Practice | Purpose |
|---|---|
| Trivy | Scan vulnerabilities |
| Non-root user | Avoid running containers as root |
| Minimal base image | Reduce security risks |
| ECR scanning | Find image vulnerabilities |
| Image tags/digests | Identify image versions |

## 4. Where Do We Store Secrets in AWS?

Never hardcode passwords or API keys inside Dockerfiles or GitHub repositories.

| Service | Purpose |
|---|---|
| AWS Secrets Manager | Database passwords and API keys |
| Parameter Store | Application configuration and secure parameters |
| IAM Role | Temporary AWS credentials |
| GitHub Secrets | CI/CD secrets when needed |

Example:

```text
Application Pod
      |
      v
AWS Secrets Manager
      |
      v
Database Password
      |
      v
Connect to Database
```

For EKS, AWS Secrets Manager can be integrated using the Secrets Store CSI Driver and AWS Secrets and Configuration Provider.

## 5. What Is Kubernetes Ingress?

Ingress is a Kubernetes resource that defines how external HTTP/HTTPS traffic reaches internal Kubernetes services.

Think of Ingress as a traffic controller.

Example:

```text
           INTERNET
               |
               v
       AWS Load Balancer
               |
               v
         INGRESS RULES
               |
       +-------+-------+
       |       |       |
       v       v       v
      /       /api   /admin
       |       |       |
       v       v       v
   Frontend  Backend  Admin
   Service   Service  Service
       |       |       |
       v       v       v
      Pods    Pods    Pods
```

### Example Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: myapp-ingress

spec:
  ingressClassName: alb

  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

**Important:**

- Ingress = Defines routing rules.
- Ingress Controller = Implements those rules.
- AWS Load Balancer Controller = Creates and manages AWS load balancers for supported Kubernetes resources.

## 6. AWS and Kubernetes Security Components

| Component | Purpose |
|---|---|
| IAM | AWS permissions |
| OIDC | GitHub authentication |
| Security Groups | Control network traffic |
| Secrets Manager | Store passwords and API keys |
| ECR | Store Docker images |
| Trivy | Scan Docker vulnerabilities |
| RBAC | Control Kubernetes API access |
| Network Policies | Control pod-to-pod traffic |
| Ingress | Manage incoming HTTP/HTTPS routing |
| ACM | Manage SSL/TLS certificates |

## 7. Important Interview Questions

**Q1. How does GitHub Actions access AWS?**

Using GitHub OIDC and an AWS IAM Role.

**Q2. What is OIDC?**

OIDC provides secure authentication without storing permanent AWS access keys.

**Q3. Where do you store database passwords?**

AWS Secrets Manager.

**Q4. How do you scan Docker images?**

Using Trivy or Amazon Inspector.

**Q5. What is ECR?**

Amazon Elastic Container Registry stores Docker images.

**Q6. What is Ingress?**

Ingress defines how external HTTP/HTTPS traffic reaches Kubernetes services.

**Q7. What is an Ingress Controller?**

It implements the routing rules defined in Ingress resources.

**Q8. What is RBAC?**

Role-Based Access Control manages who can perform actions in Kubernetes.

## 8. Final Revision

```text
GitHub Actions
      |
      | OIDC
      v
AWS IAM Role
      |
      v
Amazon ECR
      |
      v
Amazon EKS
      |
      v
Kubernetes Pods
      |
      +---- Secrets Manager
      |
      +---- Kubernetes Services
      |
      +---- Ingress
               |
               v
         AWS Load Balancer
               |
               v
            INTERNET
```

### Remember

- OIDC = GitHub to AWS authentication.
- IAM = AWS permissions.
- ECR = Docker image storage.
- EKS = Kubernetes management.
- Secrets Manager = Secure credentials.
- Trivy = Container security scanning.
- Ingress = Incoming application traffic routing.
- ALB = AWS application load balancing.

## 9. Official Documentation

- [GitHub OIDC with AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [AWS IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
- [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
- [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

**End of Notes – AWS DevOps Security**
