# Stack de Monitoramento de Infraestrutura

Este repositório contém uma stack de monitoramento completa usando Prometheus, Grafana, Node Exporter e AlertManager, configurada para enviar alertas via Telegram.

## 📁 Estrutura do Projeto

```
monitoramento/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   └── rules.yml
└── alertmanager/
    └── config.yml
```

## 🚀 Tecnologias

- Prometheus (Coleta e armazenamento de métricas)
- Grafana (Visualização de dados)
- Node Exporter (Exportador de métricas do sistema)
- AlertManager (Gerenciamento de alertas)
- Telegram (Notificações de alertas)

## 📋 Pré-requisitos

- Docker e Docker Compose instalados
- Bot do Telegram criado e configurado
- Portas disponíveis: 9090, 3000, 9093 e 9100

## 🔧 Instalação

1. Clone o repositório:
```bash
git clone https://github.com/BrendoTrindade/monitoramento-infraestrutura.git
cd monitoramento-infraestrutura
```

2. Configure o bot do Telegram:
   - Crie um bot no Telegram através do @BotFather
   - Obtenha o token do bot
   - Obtenha o chat_id do seu grupo/canal do Telegram

3. Configure o AlertManager:
   - Edite o arquivo `alertmanager/config.yml`
   - Substitua as configurações do Telegram com suas credenciais

4. Inicie os serviços:
```bash
docker-compose up -d
```

## 🖥️ Serviços e Portas

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (admin/admin)
- AlertManager: http://localhost:9093
- Node Exporter: http://localhost:9100

## ⚙️ Configurações

### Docker Compose
O arquivo `docker-compose.yml` contém a configuração completa dos serviços:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"
    restart: always

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: always

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
    ports:
      - "9093:9093"
    restart: always

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    restart: always

volumes:
  prometheus_data:
  grafana_data:
```

### Prometheus
O arquivo `prometheus/prometheus.yml` com as configurações básicas:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

### AlertManager com Telegram
O arquivo `alertmanager/config.yml` configurado para Telegram:

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'telegram-notifications'

receivers:
- name: 'telegram-notifications'
  telegram_configs:
  - bot_token: 'SEU_BOT_TOKEN'  # Substitua pelo seu token do Telegram
    chat_id: SEU_CHAT_ID        # Substitua pelo seu chat_id
    parse_mode: 'HTML'
    api_url: 'https://api.telegram.org'
```

### Regras de Alerta
O arquivo `prometheus/rules.yml` com regras de monitoramento:

```yaml
groups:
  - name: node_alerts
    rules:
      - alert: HostHighCpuLoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host alto uso de CPU (instance {{ $labels.instance }})"
          description: "CPU está acima de 80%\n  VALUE = {{ $value }}%\n  LABELS: {{ $labels }}"

      - alert: HostOutOfMemory
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host sem memória (instance {{ $labels.instance }})"
          description: "Node memory is filling up (< 10% left)\n  VALUE = {{ $value }}%\n  LABELS: {{ $labels }}"

      - alert: HostOutOfDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/"}  * 100) / node_filesystem_size_bytes{mountpoint="/"} < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host sem espaço em disco (instance {{ $labels.instance }})"
          description: "Disco está quase cheio (< 10% left)\n  VALUE = {{ $value }}%\n  LABELS: {{ $labels }}"

      - alert: HostHighLoad
        expr: node_load1 > (count without (cpu, mode) (node_cpu_seconds_total{mode="idle"})) * 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host sob alta carga (instance {{ $labels.instance }})"
          description: "Load está alto\n  VALUE = {{ $value }}%\n  LABELS: {{ $labels }}"
```

### Dashboard Grafana
Importe o arquivo `monitoramento-telegram.json` para o Grafana:

1. Acesse o Grafana (http://localhost:3000)
2. Vá para Dashboard > Import
3. Clique em "Upload JSON file"
4. Selecione o arquivo `monitoramento-telegram.json`
5. Selecione o datasource Prometheus
6. Clique em Import

Ou importe os dashboards recomendados usando os IDs:
- Node Exporter Full (ID: 1860)
- Node Exporter: Server Metrics (ID: 405)

## 🔍 Monitoramento

### Métricas Configuradas
> ⚠️ **Observação**: As configurações abaixo são apenas para testes e estudos. Ajuste os valores de acordo com as necessidades do seu ambiente de produção.

- CPU Load (Alto Uso De CPU) (>50% por 30s)
- Memória (Alto Uso De Memoria) (>50% por 30s)
- Espaço em Disco (Alto Uso de Disco) (>15% por 30s)

### Alertas via Telegram
1. Crie um bot no Telegram:
   - Abra o Telegram e procure por @BotFather
   - Digite /newbot e siga as instruções
   - Guarde o token fornecido

2. Obtenha o chat_id:
   - Adicione o bot ao grupo/canal
   - Acesse: https://api.telegram.org/bot<SEU_TOKEN>/getUpdates
   - Procure por "chat":{"id": NÚMERO}

3. Configure o AlertManager:
   - Substitua 'SEU_BOT_TOKEN' pelo token do bot
   - Substitua 'SEU_CHAT_ID' pelo ID do chat

## 🛠️ Manutenção

### Atualização
```bash
docker-compose pull
docker-compose up -d
```

### Logs
```bash
# Todos os serviços
docker-compose logs

# Serviço específico
docker-compose logs prometheus
docker-compose logs alertmanager
docker-compose logs grafana
docker-compose logs node-exporter
```

### Reiniciar Serviços
```bash
# Reiniciar todos os serviços
docker-compose restart

# Reiniciar serviço específico
docker-compose restart prometheus
```

## 🔒 Segurança

1. Proteção de Endpoints:
   - Altere a senha padrão do Grafana (admin/admin)
   - Configure autenticação básica no Prometheus (opcional)
   - Nunca exponha as portas diretamente à internet
   - Use HTTPS se expor externamente

2. Dados Sensíveis:
   - Não compartilhe o token do Telegram
   - Mantenha o chat_id privado
   - Use variáveis de ambiente para credenciais

## ❗ Troubleshooting

1. Verificar containers:
```bash
docker-compose ps
docker stats
```

2. Problemas comuns e soluções:

- **Telegram não envia alertas:**
  ```bash
  # Verifique os logs do AlertManager
  docker-compose logs alertmanager
  # Confirme se o token e chat_id estão corretos
  # Teste o bot manualmente via API
  ```

- **Métricas não aparecem:**
  ```bash
  # Verifique se o Node Exporter está acessível
  curl localhost:9100/metrics
  # Verifique os targets no Prometheus
  # http://localhost:9090/targets
  ```

- **Grafana não conecta ao Prometheus:**
  ```bash
  # Verifique a URL do datasource
  # Deve ser: http://prometheus:9090
  # Teste a conexão via UI do Grafana
  ```

## 📚 Referências

- [Documentação Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Documentação Grafana](https://grafana.com/docs/)
- [Documentação AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Node Exporter](https://github.com/prometheus/node_exporter)

## 🤝 Contribuindo

1. Fork o projeto
2. Crie sua branch de feature (`git checkout -b feature/AmazingFeature`)
3. Commit suas mudanças (`git commit -m 'Add some AmazingFeature'`)
4. Push para a branch (`git push origin feature/AmazingFeature`)
5. Abra um Pull Request
