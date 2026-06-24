<img width="1536" height="1024" alt="WhatsApp Image 2026-06-23 at 08 12 18 (1)" src="https://github.com/user-attachments/assets/968e4ae8-5887-469a-83f3-be4cae603ef2" />

# TechStock — Arquitetura AWS (S3 + ALB)

> Documentação de arquitetura do projeto TechStock. Para o passo-a-passo de implantação, consulte [`TOPOLOGIA-AWS-S3.md`](./TOPOLOGIA-AWS-S3.md).

---

## 1. Princípios de design

- **Frontend desacoplado do backend.** O site estático é hospedado no S3 com acesso público habilitado, ganhando alta disponibilidade sem depender da instância de aplicação.
- **Um único ponto de entrada dinâmico (ALB).** Todo tráfego de API e de observabilidade entra por um Application Load Balancer público, com roteamento por path (`/api/*`, `/grafana/*`). Isso desacopla o IP/instância do consumidor e fornece health checks e tolerância a falhas entre AZs.
- **Tudo que é estado/computação fica em subnet privada.** Backend, monitoramento e RDS não têm IP público. A entrada chega apenas pelo ALB; a saída ocorre via NAT Gateway.
- **Menor privilégio em rede.** Security Groups referenciam outros SGs (não CIDRs abertos), de modo que apenas a origem legítima alcança cada porta.
- **Sem segredos no código.** As credenciais do banco vivem no Secrets Manager (`techstock/backend`) e são lidas em runtime pela EC2 via `LabInstanceProfile`.
- **Acesso administrativo sem SSH.** Toda administração é feita por SSM Session Manager, eliminando a porta 22 aberta e a gestão de chaves.

---

## 2. Mapa de comunicação entre componentes

```
Internet ──HTTP──► S3 (GetObject) [conteúdo estático — bucket público]

Internet ──HTTP:80──► ALB
                        │
                        ├─ /api/*        ──HTTP:3000──► EC2 Backend (Node.js)
                        ├─ /grafana/*    ──HTTP:80────► EC2 Monitoring (Nginx) ──HTTP:3000──► Grafana
                        └─ /prometheus/* ──HTTP:80────► EC2 Monitoring (Nginx) ──HTTP:9090──► Prometheus

EC2 Backend          ──TCP:5432──► RDS PostgreSQL          [persistência]
EC2 Backend          ──HTTPS──►   Secrets Manager          [credenciais]
EC2 Monitoring       ──HTTP:9100──► EC2 Backend            [scrape Node Exporter]
EC2 Monitoring       ──HTTP:9100──► localhost              [auto-scrape Node Exporter]
EC2 Backend/Monitor  ──0.0.0.0/0──► NAT Gateway ──► IGW ──► Internet [saída: dnf, downloads]
```

### Quem fala com quem e por qual SG

| Origem | Destino | Porta | Autorizado por |
|---|---|---|---|
| Internet | S3 | 80/443 | Bucket policy pública |
| Internet | ALB | 80/443 | `sg-alb` inbound `0.0.0.0/0` |
| ALB | EC2 Backend | 3000 | `sg-backend` inbound ← `sg-alb` |
| ALB | EC2 Monitoring | 80 | `sg-monitoring` inbound ← `sg-alb` |
| EC2 Monitoring (Prometheus) | EC2 Backend (Node Exporter) | 9100 | `sg-backend` inbound ← `sg-monitoring` |
| EC2 Backend | EC2 Monitoring (Node Exporter) | 9100 | `sg-monitoring` inbound ← `sg-backend` |
| EC2 Backend | RDS | 5432 | `sg-rds` inbound ← `sg-backend` |

---

## 3. Fluxo de dados — requisições típicas

### 3.1. Carregamento do frontend (site estático)

1. Navegador → `GET http://<bucket>.s3-website-<region>.amazonaws.com/`
2. S3 responde diretamente com `index.html` + assets (bucket com static website hosting e acesso público habilitado).

> O bucket está configurado com **Static Website Hosting** e política de bucket que permite `s3:GetObject` para `Principal: *`.

### 3.2. Chamada de API (`/api/...`)

1. Navegador (JS do site) → `GET http://<ALB-DNS>/api/products`
2. ALB avalia regras: path `/api/*` casa a regra de prioridade 10 → Target Group `tech-backend`.
3. ALB encaminha à EC2 Backend na porta 3000.
4. Node.js processa:
   - Na inicialização já leu o segredo `techstock/backend` (Secrets Manager) e abriu pool ao RDS.
   - Executa a query no RDS PostgreSQL (porta 5432).
5. RDS responde → Node monta JSON → ALB → cliente.

### 3.3. Acesso ao Grafana (`/grafana/...`)

1. Navegador → `GET http://<ALB-DNS>/grafana/login`
2. ALB: path `/grafana/*` casa a regra de prioridade 20 → Target Group `tech-monitoring` (porta 80).
3. EC2 Monitoring: Nginx recebe em `:80`, `location /grafana/` → `proxy_pass localhost:3000/grafana/`.
4. Grafana (`serve_from_sub_path=true`, `root_url=.../grafana/`) responde corretamente no subpath.
5. Resposta volta: Grafana → Nginx → ALB → navegador.

> **Por que Nginx na frente do Grafana?** O Target Group precisa de um alvo HTTP estável na porta 80 com health check previsível (`/grafana/login`). O Nginx também concentra Grafana e Prometheus sob a mesma porta 80, permitindo que um único Target Group atenda os dois serviços via subpaths distintos.

### 3.4. Coleta de métricas (Prometheus)

1. Prometheus (EC2 Monitoring) lê `/etc/prometheus/prometheus.yml` a cada `scrape_interval` (15s).
2. Para cada target:
   - `localhost:9100` → Node Exporter da própria EC2 Monitoring
   - `10.0.10.x:9100` → Node Exporter da EC2 Backend (intra-VPC, autorizado por SG)
   - `localhost:9090` → o próprio Prometheus
3. Métricas são armazenadas no TSDB local (`/var/lib/prometheus`).
4. Grafana consulta o Prometheus (`datasource http://localhost:9090/prometheus`) para renderizar painéis.

---

## 4. Decisões de design e trade-offs

### 4.1. Por que S3 público em vez de servir o frontend pela EC2?

| Antes (EC2 serve estático) | Agora (S3 público) |
|---|---|
| Disponibilidade atrelada à instância | Alta disponibilidade gerenciada pela AWS |
| Escala limitada pela EC2 | Escala praticamente ilimitada para conteúdo estático |
| Deploy via cópia para a EC2 | Deploy direto com `aws s3 sync` |
| Custo adicional da instância | Custo mínimo (S3 cobra apenas por armazenamento e transferência) |

### 4.2. Por que ALB com path-based routing?

Um único hostname distribui o tráfego para serviços distintos conforme o path, sem precisar de múltiplos balanceadores ou IPs. Health checks por Target Group isolam falhas (o backend pode estar unhealthy sem derrubar o Grafana e vice-versa). O ALB também opera entre duas AZs, oferecendo tolerância a falha de zona.

### 4.3. Por que monitoramento dentro da mesma VPC (e não VPC Peering)?

A topologia anterior usava VPC Peering entre duas contas (monitoramento em conta separada). Aqui, monitoramento e aplicação convivem na mesma VPC (`10.0.0.0/16`):

- Comunicação Prometheus → Node Exporter é intra-VPC (sem peering, rotas mais simples).
- Menos pontos de falha de roteamento.
- O acesso humano ao Grafana é publicado de forma controlada pelo ALB em `/grafana/`, em vez de expor a porta 3000 com IP público.

> **Trade-off:** o isolamento entre "conta de observabilidade" e "conta de aplicação" se perde, mas para o escopo do laboratório a simplicidade e a segurança de exposição compensam.

### 4.4. Por que o Grafana em subpath `/grafana` (e não na raiz)?

Para reaproveitar uma só porta (80) e um só Target Group servindo Grafana e Prometheus, cada um em seu prefixo. Isso exige:

- `serve_from_sub_path = true` e `root_url = .../grafana/` no Grafana;
- `--web.route-prefix=/prometheus/` e `--web.external-url=/prometheus/` no Prometheus;
- `proxy_pass` do Nginx incluindo o subpath (`…:3000/grafana/`).

> Sem esses três alinhados, ocorre 404 (Nginx 404 / Grafana 200 só na raiz).

### 4.5. Por que NAT Gateway?

As instâncias privadas precisam de saída para `dnf install`, baixar binários do Prometheus/Node Exporter/Grafana e acessar o Secrets Manager (endpoint público). O NAT Gateway provê essa saída sem expor portas de entrada das instâncias.

> É o maior custo fixo recorrente — em produção real, considerar VPC Endpoints (Secrets Manager, S3) para reduzir tráfego pelo NAT.

### 4.6. Por que Secrets Manager + LabInstanceProfile?

Evita credenciais hardcoded no código/AMI. A EC2 assume o `LabInstanceProfile`, que concede `secretsmanager:GetSecretValue` e SSM. Rotacionar a senha do banco não exige redeploy de código — basta atualizar o segredo.

---

## 5. Resiliência, segurança e observabilidade

- **Resiliência:** ALB multi-AZ + Target Groups com health check; RDS com subnet group em 2 AZs (pronto para Multi-AZ se necessário).
- **Segurança:** sem IP público em recursos de estado/computação; SGs por referência; sem SSH (SSM); segredos no Secrets Manager.
- **Observabilidade:** Prometheus coleta métricas de SO (Node Exporter) de backend e monitoring; Grafana visualiza (dashboard Node Exporter Full, ID 1860); health checks do ALB indicam disponibilidade de cada serviço.

---

## 6. Pontos de evolução (produção)

| Tema | Laboratório (atual) | Produção recomendada |
|---|---|---|
| TLS | ALB em HTTP:80 | ACM + HTTPS:443 no ALB |
| Frontend | S3 público | S3 + CloudFront (OAC) para CDN, cache e HTTPS |
| Acesso ao S3/Secrets | via NAT | VPC Gateway/Interface Endpoints |
| Banco | RDS single-AZ | RDS Multi-AZ + read replicas |
| Métricas | TSDB local | Amazon Managed Service for Prometheus (AMP) |
| Escala | 1 EC2 por papel | Auto Scaling Group atrás do Target Group |
| Segredos | LabInstanceProfile | IAM Role dedicada de menor privilégio |
