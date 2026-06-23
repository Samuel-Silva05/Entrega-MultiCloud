<img width="2720" height="2240" alt="techstock_aws_topology_v2" src="https://github.com/user-attachments/assets/9e8ee674-7a89-49a6-83da-400404d07c9f" />


# Arquitetura TechStock (resumo do ambiente implantado)
 
Região AWS: us-east-1  
Contexto: ambiente de laboratório (Learner Lab). Frontend estático armazenado no S3 (privado) e servido indiretamente através do backend em EC2. Monitoramento será movido para Azure como etapa posterior.

---

## Visão geral (topologia)
Usuário → Internet → Application Load Balancer (ALB, público) → EC2 Backend (privado, Node.js)
- ALB roteia `/api/*` para o backend (porta 3000).  
- EC2 serve a API e também entrega os arquivos estáticos do frontend sincronizados internamente a partir do S3.
- RDS PostgreSQL (techstock-db) em sub-rede privada fornece persistência.
- Secrets Manager armazena credenciais do banco (secret name: `techstock/backend`).
- NAT Gateway permite saída das instâncias privadas; S3 é acessado via VPC Gateway Endpoint (sem passar pela internet).
- IAM Instance Profile (LabInstanceProfile) concede SSM e acesso a Secrets Manager na EC2.
- Segurança controlada por Security Groups (ALB, Backend, RDS).

### Componentes principais (conexões e motivo)

#### 1) Application Load Balancer (ALB)
- Função: ponto público de entrada (DNS público).
- Conexão: Listener HTTP(80) → Target Group que encaminha para instâncias EC2 na porta 3000.
- Por que: fornece DNS estável, health checks e escalabilidade; desacopla IP público das instâncias.

#### 2) EC2 Backend (Node.js)
- Local: sub-rede privada.
- Responsabilidades:
  - Servir API em `http://localhost:3000/api/*`.
  - Servir arquivos estáticos do frontend a partir de `/opt/techstock/public` (conteúdo sincronizado com o S3).
  - Ler segredos do Secrets Manager para conectar ao RDS.
- Por que: no laboratório evita-se CloudFront/ACM inicialmente; EC2 age como proxy/servidor estático e API para simplificar SSL/infra.

#### 3) S3 Bucket (frontend) — `techstock-frontend-fps` (privado)
- Uso: armazena build estático do frontend (index.html, assets).
- Acesso: privado; EC2 sincroniza via `aws s3 sync` usando role/endpoint.
- Por que: separa artefatos de frontend do backend e viabiliza deploy simples (s3 sync).

#### 4) VPC Gateway Endpoint para S3
- Uso: permite acesso privado ao S3 sem egress pela internet.
- Por que: mais seguro, evita depender do NAT para transferência de objetos S3.

#### 5) NAT Gateway
- Uso: permite que instâncias privadas façam requisições de saída (npm install, atualizações, etc.).
- Por que: manter instâncias privadas sem expor portas de entrada, mas com saída controlada.

#### 6) RDS PostgreSQL (techstock-db)
- Local: sub-rede privada, não-publicly-accessible.
- Conexão: só aceita 5432 a partir do Security Group do backend.
- Por que: segurança e gerenciamento (backups, snapshots).

#### 7) AWS Secrets Manager
- Nome do segredo usado: `techstock/backend`
- Chaves esperadas (variantes aceitas pelo código): `DB_HOST`, `DB_USER` / `username`, `DB_PASS` / `password`, `DB_NAME`, `DB_PORT`.
- Por que: credenciais seguras, sem hardcode; facilita rotação.

#### 8) IAM Instance Profile (LabInstanceProfile)
- Concede permissões mínimas: SSM (para sessão), SecretsManager:GetSecretValue (e possivelmente kms:Decrypt).
- Por que: acesso seguro a segredos e gerenciamento remoto sem SSH.

#### 9) Security Groups (resumo das regras)
- `sg-alb`: permite HTTP (80) e/ou HTTPS (443) de 0.0.0.0/0.
- `sg-backend`: permite entrada TCP 3000 vindo de `sg-alb`; permite saída a RDS (5432) e para internet via NAT.
- `sg-rds`: permite TCP 5432 apenas com origem em `sg-backend`.
- Por que: restringir acessos por origem reduz superfície de ataque.

#### 10) Health check / Target Group
- Health check: `/api/health` (porta 3000).
- Por que: valida que a API está operacional antes do ALB encaminhar tráfego.

---

## Como os elementos se conectam — cadeia de requisição típica
1. O usuário abre `http://<ALB-DNS>/` no navegador.
2. ALB recebe a requisição e, para `/api/*`, encaminha ao Target Group (instância EC2:3000).
3. EC2 recebe e o Node.js responde JSON (por ex. `/api/health`).
4. Para servir o frontend:
   - Os arquivos estáticos estão sincronizados no EC2 a partir do S3 (via `aws s3 sync` e VPC endpoint).
   - O Node serve `index.html` e assets diretamente.
5. A aplicação no EC2 lê credenciais do Secrets Manager e se conecta ao RDS para leituras/gravações.
6. Para qualquer saída externa (ex.: npm install), a EC2 usa o NAT Gateway (se necessário).

---

## Arquivos e locais importantes no repositório
- `backend/` — código do Node (API, entidades, migrations).
- `frontend/` — código front-end (build produzido e enviado para S3).
- `/opt/techstock` (na EC2) — local onde o backend e os assets são instalados:
  - `/opt/techstock/server.js` — entrypoint do Node usado em systemd.
  - `/opt/techstock/public` — frontend sincronizado a partir de S3.
- Scripts de setup (usados no lab): `setup-backend-v4-fullsecrets.sh` (configura serviço, instala dependências, registra service).

---

## Comandos úteis (deploy / verificação)
- Sincronizar frontend (EC2):
  ```bash
  sudo aws s3 sync s3://techstock-frontend-fps /opt/techstock/public --region us-east-1



# TechStock — Stack de Observabilidade

Documentação do projeto de monitoramento de infraestrutura AWS com Prometheus e Grafana, utilizando VPC Peering entre contas.

---

## Arquitetura

```
Conta A (Monitoramento)               Conta B (Aplicação)
┌──────────────────────────┐          ┌──────────────────────────┐
│  EC2 — Monitoring         │          │  EC2 — Backend           │
│  IP: 172.31.9.247        │◄────────►│  IP: 10.0.10.22          │
│                          │  VPC     │                          │
│  - Prometheus :9090      │ Peering  │  - Node Exporter :9100   │
│  - Grafana    :3000      │          │  - Node.js App           │
└──────────────────────────┘          └──────────────────────────┘
     VPC: 172.31.0.0/16                    VPC: 10.0.0.0/16
     Região: us-east-1                     Região: us-east-1
```

---

## Pré-requisitos

- Duas contas AWS com VPC Peering configurado e ativo
- Route tables de ambas as VPCs com rotas para o CIDR da outra conta
- EC2 na Conta A com Prometheus e Grafana instalados
- EC2 na Conta B com Node Exporter instalado

---

## Configuração do VPC Peering

### Route Table — Conta A

| Destination | Target |
|-------------|--------|
| 172.31.0.0/16 | local |
| 10.0.0.0/16 | pcx-xxxxxxxxx |
| 0.0.0.0/0 | igw-xxxxxxxxx |

### Route Table — Conta B (subnet privada)

| Destination | Target |
|-------------|--------|
| 10.0.0.0/16 | local |
| 172.31.0.0/16 | pcx-xxxxxxxxx |
| 0.0.0.0/16 | nat-xxxxxxxxx |

> **Atenção:** a rota do Peering na Conta B deve apontar para o CIDR específico da Conta A (`172.31.0.0/16`), não para `0.0.0.0/0`.

---

## Security Groups

### Conta B — EC2 Backend (Inbound)

| Tipo | Protocolo | Porta | Origem |
|------|-----------|-------|--------|
| Custom TCP | TCP | 9100 | 172.31.0.0/16 (CIDR da Conta A) |
| SSH | TCP | 22 | 0.0.0.0/0 |
| All ICMP - IPv4 | ICMP | All | 0.0.0.0/0 |

### Conta A — EC2 Monitoring (Outbound)

| Tipo | Protocolo | Porta | Destino |
|------|-----------|-------|---------|
| All traffic | All | All | 0.0.0.0/0 |

---

## Node Exporter (Conta B)

### Instalação

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvf node_exporter-1.8.0.linux-amd64.tar.gz
sudo mv node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
```

### Serviço systemd

```bash
sudo tee /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

### Verificação

```bash
sudo systemctl status node_exporter
curl http://localhost:9100/metrics | head -5
```

---

## Prometheus (Conta A)

### Arquivo de configuração

`/etc/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:

rule_files:

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "ec2-conta-b"
    static_configs:
      - targets:
          - "10.0.10.22:9100"
        labels:
          instance: "ec2-conta-b"
          env: "production"
```

### Aplicar configuração

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

### Verificar targets

```bash
curl http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -A3 "ec2-conta-b"
```

Ou acesse `http://<IP-publico>:9090/targets` no browser.

---

## Grafana (Conta A)

### Configuração do Datasource

1. Acesse **Connections → Data Sources → Prometheus**
2. Configure a URL como `http://localhost:9090`
3. Clique em **Save & Test**

> Use `localhost` em vez do IP público para evitar quebras quando o IP da instância mudar.

### Importar Dashboards

1. Acesse **Dashboards → Import**
2. Cole o conteúdo do arquivo JSON correspondente
3. Clique em **Load → Import**

> Antes de importar, substitua o UID do datasource nos arquivos JSON pelo UID do seu Prometheus.
> O UID pode ser encontrado na URL: `Connections → Data Sources → Prometheus → editar → ID na URL`

---

## Dashboards

### 1. Stack de Observabilidade (`techstock-dashboard.json`)

Monitora a saúde do próprio Prometheus e Grafana.

| Painel | Métrica |
|--------|---------|
| Séries Temporais Ativas | `prometheus_tsdb_head_series` |
| Taxa de Ingestão | `rate(prometheus_tsdb_head_samples_appended_total[5m])` |
| Targets UP / Total | `sum(up) / count(up)` |
| Tamanho TSDB | `prometheus_tsdb_head_chunks_storage_size_bytes` |
| Memória Heap | `go_memstats_heap_alloc_bytes` |
| Duração dos Scrapes | `scrape_duration_seconds` |

### 2. EC2 Nodes — Node Exporter (`techstock-ec2-dashboard.json`)

Monitora recursos das instâncias EC2 via Node Exporter.

| Painel | Métrica |
|--------|---------|
| CPU Usage (%) | `node_cpu_seconds_total{mode="idle"}` |
| Memória Usada (%) | `node_memory_MemAvailable_bytes` |
| Disco Usado (%) | `node_filesystem_avail_bytes` |
| Uptime | `node_time_seconds - node_boot_time_seconds` |
| Rede RX/TX | `node_network_receive/transmit_bytes_total` |
| Disco Leitura/Escrita | `node_disk_read/written_bytes_total` |
| Load Average | `node_load1`, `node_load5`, `node_load15` |

### 3. Infraestrutura EC2 (`techstock-infra-ec2.json`)

Visão consolidada de CPU, memória, disco e rede por instância (Backend e Monitoring).

---

## Diagnóstico e Troubleshooting

### Verificar conectividade entre contas

```bash
# Da Conta A, testa ICMP
ping -c 3 10.0.10.22

# Da Conta A, testa a porta do Node Exporter
curl -v --max-time 5 http://10.0.10.22:9100/metrics
```

### Possíveis erros

| Erro | Causa | Solução |
|------|-------|---------|
| `context deadline exceeded` | SG bloqueando porta 9100 | Adicionar inbound rule na Conta B |
| `Connection refused` | Node Exporter não está rodando | `sudo systemctl start node_exporter` |
| `No route to host` | Rota do VPC Peering ausente | Adicionar rota na route table |
| `No data` no Grafana | URL do datasource desatualizada | Trocar IP público por `localhost` |

### IP público mudou

Ao reiniciar a EC2, o IP público muda. Para evitar perda de acesso:

1. Acesse o Grafana pela nova URL `http://<novo-ip>:3000`
2. Atualize o datasource Prometheus para `http://localhost:9090`
3. A comunicação interna (Prometheus → Node Exporter) usa IPs privados e não é afetada

> **Recomendação:** associe um Elastic IP à EC2 de monitoramento para evitar esse problema.

---

## Estrutura do Repositório

```
.
├── README.md
├── prometheus/
│   └── prometheus.yml
├── node_exporter/
│   └── node_exporter.service
└── dashboards/
    ├── techstock-dashboard.json
    ├── techstock-ec2-dashboard.json
    └── techstock-infra-ec2.json
```

---

## Tecnologias

- **Prometheus** — coleta e armazenamento de métricas
- **Grafana** — visualização e dashboards
- **Node Exporter** — exporta métricas do sistema operacional
- **AWS EC2** — instâncias de computação
- **AWS VPC Peering** — comunicação privada entre contas AWS
