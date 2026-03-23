# Case Técnico — Engenheiro de Dados (MarTech)

## Objetivo

Este projeto propõe uma solução simples e escalável para transformar os dados brutos exportados do **GA4** em uma tabela analítica mais fácil de consumir por times de negócio, BI e marketing.

A solução foi modelada em **BigQuery + Dataform**, seguindo uma estrutura em camadas:

- **Bronze**: ingestão bruta e rastreável dos eventos
- **Silver**: padronização e extração dos campos relevantes para análise
- **Gold**: camada analítica final para métricas de funil e consumo

---

## Arquitetura proposta

### Stack sugerida para este case

- **Fonte**: exportação nativa do GA4 para BigQuery
- **Armazenamento e processamento**: BigQuery
- **Transformações**: Dataform
- **Orquestração**: Dataform Workflow / BigQuery Scheduler no cenário simples; Cloud Composer no cenário produtivo com múltiplas dependências
- **Governança**: versionamento em Git, documentação de colunas, testes de qualidade e separação por camadas

### Fluxo do pipeline

1. O GA4 exporta os dados brutos diariamente para o BigQuery.
2. A tabela **bronze_events** replica os eventos brutos relevantes para o projeto, preservando rastreabilidade.
3. A tabela **silver_events** achata a estrutura nested de `event_params`, trato colunas e expõe apenas os campos analíticos necessários.
4. A camada **gold** consolida métricas de funil e indicadores para consumo analítico.
5. Testes de qualidade garantem consistência mínima antes do consumo.

---

## Estrutura do repositório

```text
definitions/
  bronze_events.sqlx
  silver_events.sqlx
  silver_events_dq.sqlx
  gold_events.sqlx
includes/
workflow_settings.yml
README.md
```

---

## Queries SQL utilizadas

## Parte 1 — Exploração dos dados

### 1.1 Volume total por `event_name`

```sql
SELECT
  event_name,
  COUNT(*) AS qtd_events
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
GROUP BY event_name
ORDER BY qtd_events DESC;
```

### 1.2 Evolução mensal por `event_name`

```sql
SELECT
  DATE_TRUNC(PARSE_DATE("%Y%m%d", event_date), MONTH) AS event_date,
  event_name,
  COUNT(*)                                            AS qtd_events
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
GROUP BY event_date, event_name
ORDER BY event_date ASC, qtd_events DESC;
```

### 1.3 Distribuição mensal por dia da semana

```sql
SELECT
  DATE_TRUNC(PARSE_DATE("%Y%m%d", event_date), MONTH) AS event_date,
  FORMAT_DATE("%A", PARSE_DATE("%Y%m%d", event_date)) AS dia_semana,
  COUNT(*)                                            AS qtd_events
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
GROUP BY event_date, dia_semana
ORDER BY event_date ASC, dia_semana ASC;
```

### 1.4 Volume diário de eventos

```sql
SELECT
  PARSE_DATE("%Y%m%d", event_date) AS event_date,
  COUNT(*)                         AS qtd_events
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
GROUP BY event_date
ORDER BY event_date ASC;
```

---

## Parte 2 — Tabela analítica

A modelagem foi feita em duas camadas principais:

### Bronze — `definitions/bronze_events`

Objetivo: preservar os dados brutos do GA4 com mínima transformação, adicionando apenas uma coluna técnica de carga (`insert_at`).

Principais características:
- leitura da origem bruta `events_*`
- carga incremental
- preservação da estrutura original para auditoria e rastreabilidade

### Silver — `definitions/silver_events`

Objetivo: transformar os dados semi-estruturados em uma tabela analítica simples, contendo apenas os campos solicitados no case:

- `event_date`
- `event_timestamp`
- `user_pseudo_id`
- `event_name`
- `source`
- `medium`
- `page_location`
- `value`
- `insert_at`

Principais características:
- filtro apenas para os eventos relevantes do funil:
  - `page_view`
  - `view_item`
  - `begin_checkout`
  - `purchase`
- extração de campos de `event_params`
- aplicação de testes de qualidade na config (assertions)

### Teste de qualidade — `definitions/silver_events_dq`

Foi criado um teste adicional para validar que eventos de `purchase` possuam `value` preenchido.

---

## Parte 3 — Métrica de funil

A partir da `silver_events`, a camada gold pode consolidar o funil de conversão com foco no usuário `definitions/gold_events`.

```sql
WITH silver_events AS (
  SELECT
    event_date,
    user_pseudo_id,
    event_name
  FROM ${ref("silver_events")}
),
count_events AS (
  SELECT
    event_date,
    COUNT(DISTINCT IF(event_name = 'view_item', user_pseudo_id, NULL)) AS usuarios_view_item,
    COUNT(DISTINCT IF(event_name = 'begin_checkout', user_pseudo_id, NULL)) AS usuarios_begin_checkout,
    COUNT(DISTINCT IF(event_name = 'purchase', user_pseudo_id, NULL)) AS usuarios_purchase
  FROM silver_events
)
SELECT
  event_date,
  usuarios_view_item,
  usuarios_begin_checkout,
  usuarios_purchase,
  SAFE_DIVIDE(usuarios_purchase, usuarios_begin_checkout) AS conversion_rate
FROM count_events;
```

Essa abordagem mede o funil em nível de usuário, evitando inflar os números com múltiplos eventos do mesmo usuário no mesmo estágio.

---

## Breve explicação das decisões tomadas

## Parte 1

A exploração inicial foi pensada para responder rapidamente duas perguntas:

1. **Distribuição dos eventos por tipo** — para verificar se os eventos esperados do funil existem e em que escala.
2. **Padrão temporal** — para observar sazonalidade, quedas abruptas ou possíveis anomalias de coleta.

As queries de apoio por mês, dia e dia da semana ajudam a validar se a base apresenta comportamento consistente antes da modelagem analítica.

## Parte 2

A escolha por separar em **bronze** e **silver** foi feita para manter uma estrutura simples, mas aderente a boas práticas:

- a **bronze** mantém o dado bruto e auditável
- a **silver** expõe o dado já pronto para análise
- a transformação nested → tabular ocorre uma única vez
- os consumidores passam a usar uma tabela muito mais amigável para SQL analítico e BI

## Parte 3

A métrica de funil foi calculada por **usuários distintos**, não por quantidade bruta de eventos. Essa escolha é importante porque o mesmo usuário pode gerar múltiplos eventos de visualização, checkout ou compra, `COUNT(*)` distorceria a conversão caso a análise fosse baseada por `Usuário`.

---

## Parte 4 — Discussão técnica

### 4.1 Pipeline em produção

Para produção, eu organizaria o pipeline da seguinte forma:

#### Ingestão
- manteria a exportação nativa do **GA4 → BigQuery**
- criaria uma camada **bronze** com objetivo de rastreabilidade e reprocessamento

#### Transformação
- usaria **Dataform** para modelagem SQL, versionamento e testes
- separaria os modelos em:
  - **bronze**: carga bruta
  - **silver**: normalização e flatten de campos nested
  - **gold**: métricas finais e agregações analíticas

#### Orquestração
- para um cenário simples, usaria **Dataform Workflow Configurations** ou agendamento nativo
- para um ambiente com múltiplas dependências externas, SLAs, alertas e integrações, usaria **Cloud Composer (Airflow)**

#### Governança
- versionamento do código em Git
- code review via pull request
- documentação de colunas e tabelas dentro do Dataform
- separação de ambientes (`dev`, `stg`, `prod`)
- testes automáticos de qualidade e integridade

#### Observabilidade
- monitoramento de falhas de execução
- alerta para ausência de carga diária
- alerta para aumento abrupto de nulos ou queda inesperada de volume

---

### 4.2 Como garantir que o pipeline fosse idempotente

Para garantir idempotência:

1. **Incremental com chave de negócio ou merge controlado**
   - evitar append puro sem deduplicação
   - usar `uniqueKey` quando aplicável

2. **Reprocessamento seguro por partição/data**
   - Nesse caso processar por `_TABLE_SUFFIX`
   - permitir reprocessar um intervalo sem duplicar registros

3. **Carga determinística**
   - a mesma entrada deve gerar sempre a mesma saída
   - evitar lógica não determinística em joins ou agregações ambíguas

4. **Janela de reprocessamento para atraso de chegada**
   - em produção, considerar reprocessar os últimos N dias para absorver eventos tardios

---

### 4.3 Como otimizar custo no BigQuery para esse pipeline

As principais ações seriam:

#### 1. Ler apenas o necessário
- evitar `SELECT *` nas camadas curadas
- projetar só as colunas realmente consumidas

#### 2. Processamento incremental
- evitar releitura completa do histórico a cada execução
- processar apenas partições ou períodos novos/recentes

#### 3. Particionamento e clustering
- particionar tabelas por `event_date`
- considerar cluster por `event_name` e/ou `user_pseudo_id` dependendo do padrão de consulta

#### 4. Reduzir leitura da origem sharded
- ao consultar `events_*`, filtrar também por `_TABLE_SUFFIX` quando possível
- isso reduz o número de tabelas lidas e o volume escaneado

#### 5. Materializar somente o que gera valor
- manter tabelas materializadas apenas quando houver ganho claro de performance ou simplicidade
- evitar múltiplas tabelas intermediárias sem necessidade

#### 6. Reuso da camada silver
- concentrar o flatten do GA4 em uma camada reutilizável
- evitar que cada dashboard ou análise refaça o mesmo `UNNEST(event_params)`
- tornar camada silver incremental se crescer o volume dos dados

---

## Conclusão

Para este case, a combinação **BigQuery + Dataform** resolve bem o problema com baixo nível de complexidade operacional e boa escalabilidade inicial.

A solução proposta organiza o dado bruto do GA4 em camadas, melhora a consumibilidade para analistas e já abre caminho para evolução futura com:

- maior governança
- testes automatizados
- orquestração mais robusta
- reprocessamento seguro
- otimização de custo no BigQuery

---
