

# Relatório de Implementação: Observabilidade e Monitoramento

**Projeto:** Coffee Shop App
**Repositório:** [https://github.com/ellyson100/coffee-shop](https://github.com/ellyson100/coffee-shop)

## 1. Visão Geral do Projeto

Este documento detalha a arquitetura, implementação e as soluções técnicas adotadas para a construção da stack de observabilidade do projeto **Coffee Shop**, hospedado em um cluster Kubernetes (k3s). O objetivo alcançado foi a criação de um ambiente de monitoramento unificado ("Master Dashboard") no Grafana, correlacionando três pilares fundamentais:

1. **Métricas de Infraestrutura e Hardware:** Gerenciadas via Zabbix.
2. **Métricas de Aplicação e Cluster (Runtime):** Coletadas via Prometheus, Node Exporter e Kube-State-Metrics.
3. **Centralização de Logs (Log Analytics):** Indexadas e processadas via OpenSearch.

---

## 2. Provisionamento e Implantação da Infraestrutura

Para garantir a reprodutibilidade, escalabilidade e seguir as melhores práticas de *Infrastructure as Code (IaC)*, a construção do ambiente foi dividida utilizando as ferramentas mais adequadas para cada camada:

### 2.1. Cluster Kubernetes (k3s) via Ansible

A base da infraestrutura foi automatizada utilizando **Ansible**. A escolha por provisionar o cluster k3s via playbooks do Ansible garantiu a padronização na configuração dos nós (control plane e workers), automatização da instalação de dependências do sistema operacional e agilidade na subida do ambiente, eliminando configurações manuais (human error) e permitindo a recriação rápida do cluster se necessário.

### 2.2. Grafana e Zabbix via Helm

Para os componentes de visualização e monitoramento de infraestrutura, optou-se pela utilização do **Helm** (o gerenciador de pacotes do Kubernetes).

* A instalação do **Grafana** e do **Zabbix** via *Helm Charts* permitiu uma injeção fácil de variáveis de ambiente, gerenciamento simplificado de persistência de dados (PVCs) e controle de versão das releases instaladas no cluster, acelerando consideravelmente o setup inicial dessas ferramentas.

### 2.3. OpenSearch via Manifestos Nativos (YAML)

Ao contrário das ferramentas acima, a stack do **OpenSearch** (e seus componentes de indexação) foi intencionalmente implantada utilizando manifestos Kubernetes tradicionais (YAMLs) em vez de Helm.

* **Justificativa Técnica:** O deploy do OpenSearch exigiu ajustes muito finos e granulares de permissões, variáveis de JVM, alocação de memória e parametrizações de storage e rede que os Helm Charts padrão frequentemente abstraem ou dificultam a customização pontual. O uso de manifestos puros garantiu controle absoluto sobre a topologia e o comportamento dos pods do OpenSearch no cluster.

---

## 3. Endereços de Acesso e Credenciais

*(Nota para o avaliador: Substitua `<IP_DO_CLUSTER>` pelo IP público ou local do nó do Kubernetes).*

| Serviço | URL Externa (NodePort) | Credenciais (User / Password) |
| --- | --- | --- |
| **Grafana** | `http://<IP_DO_CLUSTER>:30000` | `admin` / `admin` |
| **Zabbix Web** | `http://<IP_DO_CLUSTER>:31080` | `Admin` / `zabbix` *(Case Sensitive)* |
| **Prometheus** | `http://<IP_DO_CLUSTER>:30518` | Sem autenticação |
| **OpenSearch Dashboards** | `http://<IP_DO_CLUSTER>:31957` | Sem autenticação |

**Comunicação Interna (Data Sources no Grafana):**

* **Zabbix API:** `[http://zabbix-zabbix-web.monitoring.svc.cluster.local/api_jsonrpc.php](http://zabbix-zabbix-web.monitoring.svc.cluster.local/api_jsonrpc.php)`
* **OpenSearch API:** `[http://opensearch-svc.monitoring.svc.cluster.local:9200](http://opensearch-svc.monitoring.svc.cluster.local:9200)`

---

## 4. Arquitetura de Dashboards e Queries (PromQL/Lucene)

Os dashboards foram customizados para extrair o máximo de precisão do cluster, com atenção especial à formatação e conversão de unidades (Bytes IEC, Percentuais e Rates).

### 4.1. Monitoramento de Infraestrutura (Prometheus/Node Exporter)

Para evitar a poluição de dados gerada pelos discos virtuais do Kubernetes e capturar o processamento real, as seguintes métricas foram otimizadas:

* **Uso Geral de CPU (%) baseada na ociosidade (idle):**
```promql
1 - avg(rate(node_cpu_seconds_total{instance=~"$node", mode="idle"}[$__rate_interval]))

```


* **Capacidade de Memória Total (Hardware real):**
```promql
sum(node_memory_MemTotal_bytes{instance=~"$node"})

```


* **Uso de Disco Físico (%) (Filtrando overlay/tmpfs):**
```promql
1 - (sum(node_filesystem_avail_bytes{instance=~"$node", fstype=~"ext4|xfs"}) / sum(node_filesystem_size_bytes{instance=~"$node", fstype=~"ext4|xfs"}))

```


* **Prevenção de Esgotamento (Disk Pressure nativo do k8s):**
```promql
sum(kube_node_status_condition{condition="DiskPressure", status="true"})

```



### 4.2. Monitoramento da Aplicação (Kube-State-Metrics / cAdvisor)

* **Ocupação do Cluster (Capacidade vs Rodando):**
```promql
sum(kube_pod_status_phase{phase="Running"}) / sum(kube_node_status_capacity{resource="pods"})

```


* **Uso de Memória RAM Real por Pod (Working Set):**
```promql
sum(container_memory_working_set_bytes{container!="", container!="POD", pod!=""}) by (pod)

```


* **Memória Solicitada (Requests) por Container:**
```promql
sum(kube_pod_container_resource_requests_memory_bytes{namespace=~"$namespace", node=~"$node", container!=""}) by (pod, container)

```



### 4.3. Log Analytics (OpenSearch)

A coleta de logs foi configurada visando resiliência contra as variações de formatação de `stdout`/`stderr` dos containers do Kubernetes.

* **Filtro Abrangente de Erros:** Utilização da sintaxe Lucene `*ERROR*` e `*error*` ao invés de buscar por labels estritos (como `level:"ERROR"`), garantindo a captura de exceções mesmo dentro do corpo da mensagem do log.

---

## 5. Desafios Técnicos e Resoluções Aplicadas

Durante a implementação, diversos bloqueios inerentes a ambientes de laboratório e orquestração de containers foram identificados e solucionados:

### 5.1. Injeção e Habilitação do Plugin do Zabbix no Grafana

* **Problema:** Restrições de rede (egress) impediam a instalação automática do plugin via `grafana-cli`, além da UI do Grafana bloquear plugins não-assinados por padrão.
* **Solução:** O binário do plugin `alexanderzobnin-zabbix-datasource` foi transferido manualmente para dentro do volume do pod do Grafana utilizando o comando `kubectl cp`. Posteriormente, o plugin foi ativado diretamente na interface e a integração concluída utilizando a resolução de DNS interno do Kubernetes apontando para a API via `/api_jsonrpc.php`.

### 5.2. Refatoração do Data Source de Logs (Elasticsearch vs OpenSearch)

* **Problema:** O uso temporário do plugin do Elasticsearch para ler dados do OpenSearch gerava instabilidades na tradução das queries. Ao migrar para o plugin nativo do OpenSearch, ocorreu o erro JavaScript `Cannot read properties of undefined (reading '0')` ao tentar salvar o Data Source.
* **Solução:** O erro ocorria porque a configuração apontava incorretamente para a porta `5601` (Dashboard/HTML) ao invés da API REST. A porta foi ajustada para `9200`, o Index Pattern foi simplificado para forçar a validação e o "Version Dialect" da API foi atualizado clicando em *Get Version and Save*.

### 5.3. Correção do Apontamento de Data Sources (Ajuste Fino)

* **Problema:** Painéis cruciais de saúde da aplicação ("Coffee Shop Instance Status") e de Logs exibiam *No Data*, enquanto os recursos na CLI se mostravam saudáveis.
* **Solução:** Identificou-se que a importação do JSON gerou um vínculo cruzado onde consultas de PromQL (Prometheus) e Lucene (OpenSearch) estavam sendo disparadas contra a API do Zabbix. A correção foi feita editando os painéis individualmente e reapontando a variável `${datasource}` para os respectivos bancos de dados corretos.

### 5.4. Evolução de Métricas Depreciadas

* **Problema:** As queries de monitoramento de armazenamento retornavam vazio pois o `node-exporter` do ambiente utilizava uma versão atualizada.
* **Solução:** As queries legadas (`node_filesystem_size`) foram refatoradas para a padronização atual do Prometheus (`node_filesystem_size_bytes`), reestabelecendo a coleta de telemetria dos discos.

---

## 6. Conclusão e Correlação de Dados

O ambiente encontra-se 100% operacional. A implementação da etapa 3.3 (Correlação) foi alcançada com sucesso. Através de variáveis de template (`$node`, `$pod`), um evento de anomalia (ex: pico de CPU visto nas métricas do Prometheus) pode ser instantaneamente filtrado no painel do OpenSearch (Logs), permitindo identificar o *stack trace* do erro no mesmo recorte de tempo, finalizando o circuito completo de observabilidade exigido para a sustentação da Coffee Shop.