# AWS DevOps Interview Deep Dive

Practical preparation for senior AWS, platform, infrastructure, SRE, and DevOps interviews. The material emphasizes decisions, failure isolation, tradeoffs, and safe recovery—not trivia.

> AWS, Kubernetes, Terraform, and CI/CD behavior changes. Confirm version-sensitive details in current official vendor documentation before using them in production.

## Recommended study order

### First pass: foundations

1. [AWS Foundations](01_AWS_FOUNDATIONS.md)
2. [VPC Deep Dive](02_VPC_DEEP_DIVE.md)
3. [IAM Deep Dive](03_IAM_DEEP_DIVE.md)
4. [S3 Deep Dive](04_S3_DEEP_DIVE.md)
5. [EC2 Deep Dive](05_EC2_DEEP_DIVE.md)

### Platform services

6. [EKS Deep Dive](06_EKS_DEEP_DIVE.md)
7. [ECS Deep Dive](07_ECS_DEEP_DIVE.md)
8. [RDS Deep Dive](08_RDS_DEEP_DIVE.md)
9. [Load Balancing and Auto Scaling](09_LOAD_BALANCING_AUTO_SCALING.md)
10. [Route 53 and DNS](10_ROUTE53_DNS.md)
11. [CloudFront and WAF](11_CLOUDFRONT_WAF.md)
12. [CloudWatch, Logging, and Observability](12_CLOUDWATCH_LOGGING_OBSERVABILITY.md)
13. [Secrets, KMS, and Security](13_SECRETS_KMS_SECURITY.md)

### Delivery and automation

14. [Terraform on AWS](14_TERRAFORM_AWS.md)
15. [CI/CD Pipeline Design](15_CICD_PIPELINE.md)
16. [GitHub Actions](16_GITHUB_ACTIONS.md)
17. [Jenkins](17_JENKINS.md)
18. [Docker and Kubernetes DevOps](18_DOCKER_KUBERNETES_DEVOPS.md)

### Interview practice

19. [AWS Troubleshooting Scenarios](19_AWS_TROUBLESHOOTING_SCENARIOS.md)
20. [AWS Architecture Scenarios](20_AWS_ARCHITECTURE_SCENARIOS.md)
21. [150 Interview Questions](21_AWS_DEVOPS_INTERVIEW_QUESTIONS.md)
22. [Last-Minute Cheat Sheet](22_AWS_DEVOPS_LAST_MINUTE_CHEATSHEET.md)

## How to use it

- Explain every diagram aloud in plain language.
- For each design, state requirements, assumptions, failure modes, security, observability, cost, and rollback.
- In scenarios, diagnose by layer and change one variable at a time.
- Answer the question bank before reading its answer.
- Never claim production ownership you do not have. Say: “I have implemented this in a lab; in production I would validate the following…” and then give the engineering approach.

## If the interview is tomorrow

Read chapters 2, 3, 6, 9, 14, 19, and 20; practice chapter 21 aloud; finish with chapter 22 only.
