<img width="1024" height="559" alt="WhatsApp Image 2026-06-17 at 08 19 38 (1)" src="https://github.com/user-attachments/assets/1c82c321-4c91-405c-8d86-a1815ec4f5dc" />


# 🚀 AWS - Arquitetura Inicial Honey Badger

## 📌 Descrição da Arquitetura

Este projeto provisiona uma arquitetura na AWS com:

- VPC (10.0.0.0/16)
- Internet Gateway (IGW)
- Subnet pública
- EC2 Frontend (porta 80/443)
- EC2 Backend - ALB
- EC2 Monitoring (Prometheus 9090 / Grafana 3000)
- Security Groups com isolamento de tráfego

---

## 💰 Estimativa de Custos (Versão 1)

- 💸 Custo mensal: **$144,54**
- 📅 12 meses: **$1.734,48**

📄 Documento completo:
👉 [Ver PDF](https://github.com/Samuel-Silva05/ENTREGA-FINAL-MULTI-CLOUD-AWS-AZURE-/blob/main/My%20Estimate%20-%20Calculadora%20de%20Pre%C3%A7os%20da%20AWS.pdf).

| Serviço                | Descrição                 | Região                | Custo inicial | Custo mensal |
| ---------------------- | ------------------------- | --------------------- | ------------- | ------------ |
| Amazon EC2             | Frontend (HTML/JS/CSS)    | US East (N. Virginia) | 0,00 USD      | 13,72 USD    |
| Amazon EC2             | Backend (Node.js)         | US East (N. Virginia) | 0,00 USD      | 13,72 USD    |
| Amazon EC2             | Prometheus + Grafana      | US East (N. Virginia) | 0,00 USD      | 13,72 USD    |
| Elastic Load Balancing | Application Load Balancer | US East (N. Virginia) | 0,00 USD      | 72,00 USD    |
| Amazon VPC             | NAT Gateway               | US East (N. Virginia) | 0,00 USD      | 31,38 USD    |


## 💰 Estimativa de Custos AWS (Versão 2)

- 💸 Custo mensal: **$434,90**
- 📅 Custo anual: **$5.218,80**
- 🚀 Região: us-east-1

📄 Documento completo:
👉 [Ver PDF](https://github.com/Samuel-Silva05/ENTREGA-FINAL-MULTI-CLOUD-AWS-AZURE-/blob/main/My%20Estimate.pdf)

| Serviço                | Descrição                 | Região                  | Custo Inicial | Custo Mensal |
| ---------------------- | ------------------------- | ----------------------- | ------------- | ------------ |
| Amazon EC2             | Frontend (HTML/JS/CSS)    | us-east-1 (N. Virginia) | $0.00         | $13.72       |
| Amazon EC2             | Prometheus / Grafana      | us-east-1 (N. Virginia) | $0.00         | $13.72       |
| Amazon EC2             | Backend (Node.js)         | us-east-1 (N. Virginia) | $0.00         | $13.72       |
| Elastic Load Balancing | Application Load Balancer | us-east-1 (N. Virginia) | $0.00         | $323.03      |
| Amazon VPC             | VPC + NAT Gateway         | us-east-1 (N. Virginia) | $0.00         | $70.71       |
