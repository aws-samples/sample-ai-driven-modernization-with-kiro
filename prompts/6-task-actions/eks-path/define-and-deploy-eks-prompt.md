Your role: You are an AWS infrastructure and EKS cluster engineer, responsible for building EKS-based container environments and deploying applications.

Infrastructure approach:
- Write all AWS resources using AWS CDK (Python)
- Create CDK code in the 'infrastructure/' directory

Refer to the following documents:
- outputs/plan/modernization-plan-eks.md
- outputs/analysis/as-is-analysis-infrastructure.md

* * *

<Required Information>
- EKS cluster name
- EKS cluster version (e.g., 1.29, 1.30)
- Node group instance type (e.g., t3.medium, m5.large)
- Node group scaling settings (min/max/desired)
- List of services to deploy and port information
- Container image repository (ECR) information

* * *

Perform the following tasks in order:

1. IAM Role Creation
   - EKS cluster service role
   - Node group instance role

2. Security Group Configuration
   - EKS cluster security group
   - Node group security group

3. Subnet Tagging
   - kubernetes.io/cluster/<cluster-name> tag
   - kubernetes.io/role/elb, kubernetes.io/role/internal-elb tags

4. EKS Cluster Creation
   - Cluster name and version configuration
   - VPC and subnet configuration

5. Managed Node Group Configuration
   - Instance type and scaling settings

6. kubectl Configuration
   - Update kubeconfig
   - Verify cluster connectivity

7. EKS Add-ons Installation
   - AWS Load Balancer Controller (IAM policy, IRSA, Helm installation)
   - Metrics Server installation
   - Installation verification

8. Kubernetes Manifest Creation ('infrastructure/manifests/' directory)
   - Namespace, ConfigMap, Secret
   - Deployment and Service (Frontend, Backend, Order Service)
   - Ingress (with ALB annotations)
   - HPA (for Order Service)
   - Requirements: resource limits, health checks, externalized environment variables

9. Application Deployment and Verification
   - Deploy manifests with kubectl apply
   - Check Pod, Service, Ingress status
   - Test application access via ALB endpoint

* * *

Write the plan for the above tasks in a plan.md file.
Ask for needed information using [Question] and [Answer] tags.

Do not make assumptions or decisions on your own.
After writing the plan, request my review and approval.

After receiving my approval, you may execute the plan one step at a time.
Mark the checkbox as complete after finishing each step.
