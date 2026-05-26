# LAB-04 — Observabilidade e Monitoramento

![Story](https://img.shields.io/badge/type-Story-0052CC?style=flat-square)
![Medium Priority](https://img.shields.io/badge/priority-Medium-F5A623?style=flat-square)
![Observability](https://img.shields.io/badge/epic-Observability-6B48C7?style=flat-square)
![Status](https://img.shields.io/badge/status-In%20Progress-F5A623?style=flat-square)
![Sprint](https://img.shields.io/badge/sprint-Sprint%202-0052CC?style=flat-square)
![Estimate](https://img.shields.io/badge/estimate-3--4%20dias-555555?style=flat-square)

---

## Identificação

| Campo       | Valor                        |
|-------------|------------------------------|
| ID          | `INFRA-004`                  |
| Epic        | Portfólio Enterprise         |
| Tipo        | Story                        |
| Prioridade  | Medium                       |
| Sprint      | Sprint 2                     |
| Estimativa  | 3–4 dias                     |
| Assignee    | @seu-usuario                 |
| Repositório | `observability-monitoring-lab`        |
| Status      | In Progress                  |

---

## Objetivo

Construir uma stack de observabilidade completa que monitore os ambientes dos labs anteriores (LAB-01 e LAB-02), demonstrando visão operacional moderna com coleta de métricas, visualização em dashboards e sistema de alertas. O resultado deve parecer um NOC (Network Operations Center) de empresa, não um tutorial de documentação oficial.

---

## Descrição

A stack é provisionada via Docker Compose e integrada à rede do LAB-01. Composta por Prometheus (coleta via scrape), Grafana (dashboards pré-provisionados), Alertmanager (roteamento de alertas) e MailHog (recepção de notificações local).

**Exporters configurados:**

| Exporter                     | Alvo          | Métricas coletadas                          |
|------------------------------|---------------|---------------------------------------------|
| `node_exporter`              | Hosts Linux   | CPU, memória, disco, rede, load             |
| `nginx-prometheus-exporter`  | NGINX         | RPS, conexões ativas, status codes          |
| HAProxy stats via socket     | HAProxy       | Backends UP/DOWN, RPS, bytes in/out         |
| `jmx_exporter` (embedded)    | WildFly       | Heap, GC pauses, threads, datasource pool   |
| `mssql-exporter`             | SQL Server    | Conexões, queries/s, deadlocks, espaço      |

**Dashboards Grafana (5 no total):**
1. Infraestrutura geral — CPU, memória, disco por host
2. NGINX — requisições/s, latência p95, erros 4xx/5xx
3. HAProxy — backends UP/DOWN, RPS por backend, bytes
4. WildFly JVM — heap, GC pauses, threads, pool usage
5. Disponibilidade — uptime SLA, incidentes simulados

**Regras de alerta Prometheus (8 no total):**

| Alerta                    | Condição                                  | Severidade |
|---------------------------|-------------------------------------------|------------|
| `HostDown`                | Host sem resposta por > 1 min             | critical   |
| `HighCPU`                 | CPU > 80% por 5 minutos                   | warning    |
| `HighMemory`              | Memória > 85%                             | warning    |
| `BackendDown`             | Backend HAProxy marcado DOWN              | critical   |
| `HighJvmHeap`             | Heap JVM > 80% do máximo                 | warning    |
| `ConnectionPoolExhausted` | Pool WildFly esgotado                    | critical   |
| `HighNginxErrorRate`      | Taxa de erros NGINX > 5%                 | warning    |
| `LowDiskSpace`            | Espaço em disco < 15%                    | warning    |

---

## Arquitetura

```
┌─────────────────────────────────────────────────────┐
│                   LAB-01 Network                    │
│                                                     │
│  HAProxy ──► NGINX ──► WildFly ──► SQL Server       │
│     │           │          │                        │
│     │ stats     │ /metrics │ jmx_exporter           │
│     └───────────┴──────────┘                        │
│                  │                                  │
│         ┌────────▼────────┐                         │
│         │   Prometheus    │  scrape / eval rules    │
│         └────────┬────────┘                         │
│                  │ alerts                           │
│         ┌────────▼────────┐                         │
│         │  Alertmanager   │ ──► MailHog (SMTP local)│
│         └─────────────────┘                         │
│                  │ datasource                       │
│         ┌────────▼────────┐                         │
│         │     Grafana     │  dashboards + alerting  │
│         └─────────────────┘                         │
└─────────────────────────────────────────────────────┘
```

---

## Stack técnica

![Prometheus](https://img.shields.io/badge/Prometheus-2.x-E6522C?style=flat-square&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-10-F46800?style=flat-square&logo=grafana&logoColor=white)
![Alertmanager](https://img.shields.io/badge/Alertmanager-0.26-E6522C?style=flat-square)
![MailHog](https://img.shields.io/badge/MailHog-SMTP%20local-8B89CC?style=flat-square)
![Docker](https://img.shields.io/badge/Docker%20Compose-v3.9-2496ED?style=flat-square&logo=docker&logoColor=white)

---

## Pré-requisitos

- LAB-01 em execução (esta stack conecta na mesma rede Docker)
- Portas livres: `9090` (Prometheus), `3000` (Grafana), `9093` (Alertmanager), `8025` (MailHog UI), `8025` SMTP
- Mínimo 2 GB de RAM adicional para a stack de observabilidade

---

## Como executar

```bash
git clone https://github.com/seu-usuario/observability-monitoring-lab.git
cd observability-monitoring-lab

# Certifique-se de que o LAB-01 está rodando
docker network ls | grep lab-enterprise

# Suba a stack de observabilidade
docker-compose up -d

# Acesse os serviços
# Grafana:      http://localhost:3000  (admin / admin)
# Prometheus:   http://localhost:9090
# Alertmanager: http://localhost:9093
# MailHog:      http://localhost:8025

# Simule um incidente para testar alertas
./scripts/simulate-incident.sh
```

---

## Estrutura do repositório

```
observability-monitoring-lab/
├── prometheus/
│   ├── prometheus.yml         ← scrape configs
│   └── alerts.yml             ← 8 regras de alerta
├── alertmanager/
│   └── alertmanager.yml       ← roteamento por severidade
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── prometheus.yml
│       └── dashboards/
│           ├── dashboards.yml
│           ├── infra-general.json
│           ├── nginx.json
│           ├── haproxy.json
│           ├── wildfly-jvm.json
│           └── availability.json
├── exporters/
│   └── jmx_exporter/
│       └── config.yaml
├── scripts/
│   └── simulate-incident.sh
├── docs/
│   └── monitoring-runbook.md
├── docker-compose.yml
└── README.md
```

---

## Tarefas de implementação

- [ ] Montar `docker-compose-observability.yml` na mesma rede do LAB-01
- [ ] Configurar `prometheus.yml` com todos os scrape jobs e intervalos
- [ ] Escrever `alerts.yml` com as 8 regras e severidade classificada
- [ ] Configurar `alertmanager.yml` com roteamento por severidade e receptor MailHog
- [ ] Criar provisionamento automático do Grafana (datasource + dashboards)
- [ ] Construir os 5 dashboards e exportar como JSON versionado
- [ ] Script `simulate-incident.sh`: derrubar container, capturar alerta, restaurar
- [ ] Documento `monitoring-runbook.md` com ação para cada alerta
- [ ] Capturas de tela de todos os dashboards com dados reais e alerta disparado

---

## Critérios de aceite

- [ ] Prometheus com todos os targets em status UP em `/targets`
- [ ] Cinco dashboards funcionais sem painéis com "No data"
- [ ] Alerta `HostDown` disparado em menos de 2 minutos e notificação no MailHog
- [ ] Dashboards provisionados automaticamente após `docker-compose up`
- [ ] Runbook cobre todos os 8 alertas com ação de resposta documentada

---

## Troubleshooting

### Target no Prometheus em estado DOWN
```bash
# Verifique se o exporter está acessível
curl http://localhost:9100/metrics | head -5   # node_exporter
curl http://localhost:9113/metrics | head -5   # nginx exporter

# Verifique o log do Prometheus
docker-compose logs prometheus | grep -i "scrape\|error"

# Causa comum: exporter não está na mesma rede Docker que o Prometheus
```

### Dashboard Grafana sem dados
```bash
# Verifique a query PromQL diretamente no Prometheus
# http://localhost:9090/graph
# Execute: up{job="nginx"}

# Se retorna vazio, o scrape job está falhando (ver acima)
# Se retorna dados, o problema é na query do dashboard — edite o painel
```

### Alerta não chega no MailHog
```bash
# Verifique se o Alertmanager está recebendo alertas
curl http://localhost:9093/api/v2/alerts

# Verifique o roteamento no alertmanager.yml
docker-compose logs alertmanager

# Confirme que o MailHog está acessível na porta SMTP 1025
docker-compose exec alertmanager nc -v mailhog 1025
```

---

## Referências

- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Grafana dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Alertmanager routing](https://prometheus.io/docs/alerting/latest/configuration/)
- [jmx_exporter configuration](https://github.com/prometheus/jmx_exporter)
