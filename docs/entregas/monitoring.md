# TechStock — Stack de Observabilidade

Documentação do projeto de monitoramento de infraestrutura AWS com Prometheus e Grafana, com duas instâncias EC2 na mesma VPC.

---

## Arquitetura

```
Internet
   │
   ▼
ALB (público)
   │
   ├─ /grafana/*    ──HTTP:80──► EC2 Monitoring (Nginx) ──► Grafana    :3000
   ├─ /prometheus/* ──HTTP:80──► EC2 Monitoring (Nginx) ──► Prometheus :9090
   └─ /api/*        ──HTTP:3000─► EC2 Backend (Node.js)

                        VPC: 10.0.0.0/16 — us-east-1
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  EC2 — Monitoring                EC2 — Backend          │
│  IP privado: 10.0.x.x            IP privado: 10.0.10.22 │
│                                                          │
│  - Prometheus :9090  ◄─────────── Node Exporter :9100   │
│  - Grafana    :3000                                      │
│  - Node Exporter :9100 (auto-scrape)                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Pré-requisitos

- Uma conta AWS com uma VPC configurada
- ALB público com path-based routing configurado (`/grafana/*`, `/prometheus/*`, `/api/*`)
- EC2 de Monitoring com Prometheus, Grafana e Nginx instalados (subnet privada)
- EC2 de Backend com Node Exporter instalado (subnet privada)
- Security Groups configurados para permitir comunicação entre as instâncias

---

## Security Groups

### EC2 Backend (Inbound)

| Tipo | Protocolo | Porta | Origem |
|------|-----------|-------|--------|
| Custom TCP | TCP | 9100 | Security Group da EC2 Monitoring |
| Custom TCP | TCP | 3000 | Security Group do ALB |

### EC2 Monitoring (Inbound)

| Tipo | Protocolo | Porta | Origem |
|------|-----------|-------|--------|
| Custom TCP | TCP | 80 | Security Group do ALB |

### EC2 Monitoring (Outbound)

| Tipo | Protocolo | Porta | Destino |
|------|-----------|-------|---------|
| All traffic | All | All | 0.0.0.0/0 |

> **Boa prática:** referencie o Security Group da EC2 Monitoring diretamente na regra de inbound da EC2 Backend (em vez de usar o CIDR da VPC), restringindo o acesso à porta 9100 apenas à instância de monitoramento.

---

## Node Exporter (EC2 Backend)

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

## Prometheus (EC2 Monitoring)

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

  - job_name: "ec2-backend"
    static_configs:
      - targets:
          - "10.0.10.22:9100"
        labels:
          instance: "ec2-backend"
          env: "production"

  - job_name: "ec2-monitoring"
    static_configs:
      - targets:
          - "localhost:9100"
        labels:
          instance: "ec2-monitoring"
          env: "production"
```

### Aplicar configuração

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

### Verificar targets

```bash
curl http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -A3 "ec2-backend"
```

Ou acesse `http://<ALB-DNS>/prometheus/targets` no browser.

---

## Grafana (EC2 Monitoring)

### Acesso

O Grafana é acessado pelo DNS do ALB no subpath `/grafana/`:

```
http://<ALB-DNS>/grafana/
```

> O Nginx na EC2 Monitoring recebe as requisições do ALB na porta 80 e faz proxy para o Grafana em `localhost:3000`. Por isso o Grafana deve estar configurado com `serve_from_sub_path = true` e `root_url = http://<ALB-DNS>/grafana/`.

### Configuração do Datasource

1. Acesse `http://<ALB-DNS>/grafana/`
2. Vá em **Connections → Data Sources → Prometheus**
3. Configure a URL como `http://localhost:9090`
4. Clique em **Save & Test**

> Use `localhost` em vez do IP da instância para evitar quebras caso o IP mude após reinicialização.

### Importar Dashboards

1. Acesse **Dashboards → Import**
2. Cole o conteúdo do arquivo JSON correspondente
3. Clique em **Load → Import**

> Antes de importar, substitua o UID do datasource nos arquivos JSON pelo UID do seu Prometheus.  
> O UID pode ser encontrado na URL: **Connections → Data Sources → Prometheus → editar → ID na URL**

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

### Verificar conectividade entre instâncias

```bash
# Da EC2 Monitoring, testa ICMP para o Backend
ping -c 3 10.0.10.22

# Da EC2 Monitoring, testa a porta do Node Exporter no Backend
curl -v --max-time 5 http://10.0.10.22:9100/metrics
```

### Possíveis erros

| Erro | Causa | Solução |
|------|-------|---------|
| `context deadline exceeded` | Security Group bloqueando porta 9100 | Adicionar inbound rule na EC2 Backend liberando porta 9100 para o SG da EC2 Monitoring |
| `Connection refused` | Node Exporter não está rodando | `sudo systemctl start node_exporter` |
| `No route to host` | Problema de roteamento na VPC | Verificar route tables e Security Groups |
| `No data` no Grafana | URL do datasource incorreta | Verificar se datasource aponta para `http://localhost:9090` |

### IP da EC2 Monitoring mudou

Ao reiniciar a EC2, o IP privado geralmente se mantém, mas caso mude:

1. O acesso ao Grafana e Prometheus continua pelo DNS do ALB — ele não é afetado
2. Atualize o Target Group no ALB para apontar para o novo IP privado da EC2 Monitoring
3. A comunicação interna (Prometheus → Node Exporter) usa IPs privados — verifique se o IP do Backend em `prometheus.yml` ainda está correto

> **Recomendação:** use IPs privados estáticos (via configuração da ENI) ou referencie as instâncias pelo ID no Target Group para evitar esse problema.

---

## Tecnologias

- **Prometheus** — coleta e armazenamento de métricas
- **Grafana** — visualização e dashboards
- **Node Exporter** — exporta métricas do sistema operacional
- **Nginx** — proxy reverso na EC2 Monitoring, expõe Grafana e Prometheus sob subpaths via porta 80
- **AWS ALB** — ponto de entrada público com path-based routing (`/grafana/*`, `/prometheus/*`, `/api/*`)
- **AWS EC2** — instâncias de computação
- **AWS VPC** — rede privada com comunicação intra-VPC entre as instâncias
