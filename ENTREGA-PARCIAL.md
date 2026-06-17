# 🚀 AWS 3-Tier Architecture com Terraform (IaC)

## 📌 Descrição da Arquitetura

Este projeto provisiona uma arquitetura na AWS com:

- VPC (10.0.0.0/16)
- Internet Gateway (IGW)
- Subnet pública e privada
- Application Load Balancer (ALB)
- EC2 Frontend (porta 80)
- EC2 Backend (porta 3000)
- EC2 Monitoring (Prometheus 9090 / Grafana 3000)
- Security Groups com isolamento de tráfego

---

## 🔄 Fluxo de Tráfego

- **Tráfego externo:**  
  Internet → IGW → ALB → Frontend (80) / Backend (3000)

- **Monitoramento:**  
  Frontend + Backend → Monitoring (9090 / 3000)

---

## ▶️ Como usar

```bash
terraform init
terraform plan
terraform apply
