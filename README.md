# Automated Kubernetes Ingress with Sticky Sessions and Enhanced Security: Using Terraform to Deploy NGINX/HAProxy, AWS Application Load Balancer, and WAF

![FLTm4VP8HBSPsG8Czeti6Q](https://github.com/user-attachments/assets/c1d3f204-8a21-4f68-b8e4-8f8e7a113836)

### Introduction
In modern cloud architectures, seamless load balancing and secure handling of user sessions are critical for delivering scalable and resilient applications. Kubernetes, combined with AWS services, provides a robust environment for managing applications. By implementing an Ingress Controller like NGINX or HAProxy, along with an AWS Application Load Balancer (ALB) and AWS Web Application Firewall (WAF), we can achieve smooth load distribution, enhanced security, and consistent user experiences. Leveraging Terraform automates this deployment, ensuring repeatability and easy maintenance.

### Objective
The objective of this project is to deploy a Kubernetes Ingress Controller with sticky session support, routed through an AWS ALB and protected by AWS WAF. Using Terraform, we aim to automate this setup to provide a highly available, scalable, and secure environment that maintains user session consistency even during pod scaling events. This setup facilitates handling stateful sessions in a cloud-native way, without hardcoding infrastructure dependencies.

The key security requirements for deploying a Kubernetes Ingress Controller with AWS ALB and WAF using Terraform, ensuring a secure and resilient environment:

### 1. **Access Control**
   - **Least Privilege Principle**: Assign the minimum permissions needed for AWS IAM roles used by Terraform to provision resources (e.g., ALB, WAF, and Route 53).
   - **Role-based Access Control (RBAC)** in Kubernetes: Configure RBAC to restrict access to the Ingress Controller, allowing only authorized pods and services to use it.

### 2. **Network Security**
   - **Secure Ingress**: Use security groups to control access to the ALB, allowing only HTTPS/HTTP traffic from trusted IPs or sources.
   - **Restrict Outbound Access**: Limit outbound access from Kubernetes pods and AWS resources, especially from sensitive services.

### 3. **Data Encryption**
   - **TLS Encryption**: Enable TLS on the ALB to encrypt data in transit from clients to the Ingress Controller.
   - **SSL Termination**: Optionally enable SSL termination at the ALB or configure end-to-end TLS for traffic between ALB and Kubernetes.

### 4. **WAF Protection**
   - **Custom WAF Rules**: Define custom AWS WAF rules to block IP addresses, SQL injection, Cross-Site Scripting (XSS), and other malicious requests.
   - **Logging and Monitoring**: Enable WAF logging and CloudWatch metrics to monitor threats and access patterns.

### 5. **Session Security**
   - **Secure Cookies**: Configure cookies for sticky sessions as `Secure` and `HttpOnly` to prevent interception or manipulation by malicious actors.

### 6. **DNS Security**
   - **Route 53 Access Control**: Restrict who can modify Route 53 DNS records to prevent unauthorized changes.
   - **DNS Record TTL**: Set appropriate TTL values for DNS records to prevent outdated IPs being cached if the ALB changes.

### 7. **Auditing and Monitoring**
   - **Terraform State Management**: Securely manage Terraform state files to protect sensitive information.
   - **Enable AWS CloudTrail**: Enable CloudTrail for monitoring API calls related to the creation and modification of resources such as ALB, WAF, and Route 53.
   - **Ingress Logs**: Enable access logs for both the ALB and the Ingress Controller to analyze traffic patterns and identify any unauthorized access attempts.

### 8. **Pod Security Policies (PSP)**
   - Implement Pod Security Policies to restrict the behavior of pods within Kubernetes, preventing privilege escalation and enforcing resource limits. 

### Scope
This project encompasses:
1. Configuring Terraform to deploy NGINX/HAProxy Ingress Controller with sticky session handling in Kubernetes.
2. Setting up an AWS ALB to distribute incoming traffic to the Kubernetes Ingress, enabling secure load balancing.
3. Implementing AWS WAF rules to protect against common web vulnerabilities.
4. Configuring Route 53 for DNS resolution to direct traffic to the ALB.
This scope ensures a fully automated, end-to-end solution for managing secure, session-aware traffic in a Kubernetes-based environment.

Here’s a complete **Terraform setup** to create an NGINX or HAProxy Ingress Controller in Kubernetes with sticky session handling, along with an AWS ALB and WAF for enhanced security.

### Project Structure
The project structure for this configuration should look like:

```
my-k8s-aws-ingress-setup/
├── main.tf
├── variables.tf
├── outputs.tf
└── provider.tf
```

### Step 1: Define Providers in `provider.tf`

Configure the AWS and Kubernetes providers.

```hcl
provider "aws" {
  region = var.aws_region
}

provider "kubernetes" {
  config_path = "~/.kube/config" # Adjust this path as needed
}
```

### Step 2: Define Variables in `variables.tf`

Define variables to customize the setup:

```hcl
variable "aws_region" {
  default = "us-east-1"
}

variable "vpc_id" {
  description = "VPC ID where the ALB and target group are deployed."
}

variable "subnets" {
  type        = list(string)
  description = "Subnets for the ALB."
}

variable "security_group_ids" {
  type        = list(string)
  description = "Security groups for ALB."
}

variable "domain_name" {
  description = "Domain name to use with Route 53."
}

variable "route53_zone_id" {
  description = "Route 53 Hosted Zone ID."
}
```

### Step 3: Define Resources in `main.tf`

This file includes resources for deploying the Kubernetes Ingress Controller, configuring sticky sessions, setting up the AWS ALB, configuring WAF, and creating DNS records.

#### 1. Deploy NGINX Ingress Controller with Sticky Sessions in Kubernetes

Use Helm to deploy NGINX with Terraform and enable sticky sessions.

```hcl
resource "kubernetes_namespace" "ingress_nginx" {
  metadata {
    name = "ingress-nginx"
  }
}

resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  namespace  = kubernetes_namespace.ingress_nginx.metadata[0].name
  values = [
    {
      controller: {
        service: {
          type = "LoadBalancer"
        }
      }
    }
  ]
}
```

2. **Add an Ingress Resource with Sticky Sessions Enabled**:
   - Define a Kubernetes `Ingress` resource with annotations to enable sticky sessions using cookies.

   ```hcl
   resource "kubernetes_ingress" "my_app_ingress" {
     metadata {
       name      = "my-app"
       namespace = "default"
       annotations = {
         "nginx.ingress.kubernetes.io/affinity"                    = "cookie"
         "nginx.ingress.kubernetes.io/session-cookie-name"         = "my_app_session"
         "nginx.ingress.kubernetes.io/session-cookie-expires"      = "1h"
         "nginx.ingress.kubernetes.io/session-cookie-max-age"      = "3600"
       }
     }

     spec {
       rule {
         host = var.domain_name
         http {
           path {
             path = "/"
             backend {
               service_name = "my-service"
               service_port = 80
             }
           }
         }
       }
     }
   }
   ```

#### 2. Create AWS Application Load Balancer and Target Group

Set up an ALB and target group in AWS using Terraform.

```hcl
resource "aws_lb" "k8s_alb" {
  name               = "k8s-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = var.security_group_ids
  subnets            = var.subnets
}

resource "aws_lb_target_group" "k8s_ingress_target_group" {
  name     = "k8s-ingress-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 3
    unhealthy_threshold = 2
    matcher             = "200-399"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.k8s_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.k8s_ingress_target_group.arn
  }
}

resource "aws_lb_target_group_attachment" "k8s_ingress_attachment" {
  target_group_arn = aws_lb_target_group.k8s_ingress_target_group.arn
  target_id        = helm_release.nginx_ingress.status[0].load_balancer.ingress[0].ip # External IP of Ingress Controller
  port             = 80
}
```

#### 3. Set Up AWS WAF Rules and Associate with ALB

```hcl
resource "aws_wafv2_web_acl" "k8s_waf_acl" {
  name        = "k8s-waf"
  description = "WAF for Kubernetes ALB"
  scope       = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "SQLInjectionProtection"
    priority = 1

    statement {
      sqli_match_statement {
        field_to_match {
          uri_path {}
        }
        text_transformation {
          priority = 0
          type     = "NONE"
        }
      }
    }

    action {
      block {}
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLInjectionProtection"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "k8s-waf"
    sampled_requests_enabled   = true
  }
}

resource "aws_wafv2_web_acl_association" "alb_waf_association" {
  resource_arn = aws_lb.k8s_alb.arn
  web_acl_arn  = aws_wafv2_web_acl.k8s_waf_acl.arn
}
```

#### 4. Create Route 53 DNS Record

Point your domain name to the ALB.

```hcl
resource "aws_route53_record" "app_dns" {
  zone_id = var.route53_zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.k8s_alb.dns_name
    zone_id                = aws_lb.k8s_alb.zone_id
    evaluate_target_health = true
  }
}
```

### Step 4: Define Outputs in `outputs.tf`

This file outputs the DNS name of the ALB and WAF ARN.

```hcl
output "alb_dns_name" {
  value = aws_lb.k8s_alb.dns_name
}

output "waf_arn" {
  value = aws_wafv2_web_acl.k8s_waf_acl.arn
}
```

### Step 5: Initialize and Apply the Terraform Configuration

1. **Initialize Terraform**:
   ```bash
   terraform init
   ```

2. **Apply the Terraform Configuration**:
   ```bash
   terraform apply
   ```
   Confirm and let Terraform create the resources.

---

### Verification and Testing

1. **ALB**: Verify that the ALB has the correct target group and health checks configured.
2. **WAF**: Check AWS WAF logs to ensure rules are functioning as expected.
3. **Sticky Sessions**: Access the application and verify session persistence across requests.
4. **DNS**: Check that the domain resolves correctly to the ALB’s DNS name.

This configuration provides a fully automated Terraform setup for deploying an NGINX Ingress Controller with sticky session handling, AWS ALB for load balancing, and AWS WAF for added security.
