# Documentação da Arquitetura de Dados

## Visão geral

Este projeto apresenta a documentação HTML da arquitetura corporativa de dados baseada em Medallion Architecture, Data Marts por domínio, governança, auditoria, orquestração e publicação para consumo analítico.

A proposta foi desenhada para operar inicialmente em ambiente on-premise com SQL Server, automações em Python e integrações por API, arquivos e scraping, mantendo compatibilidade conceitual com futuras evoluções para serviços em nuvem, Lakehouse, Microsoft Fabric, Databricks ou Snowflake.

## Arquivo principal

- `documentacao_arquitetura_dados_v1.html` — documentação navegável da arquitetura.

## Objetivos da arquitetura

- Separar claramente ingestão, armazenamento, tratamento, modelagem e consumo.
- Centralizar regras de negócio na camada semântica.
- Publicar dados por domínio e produto de dados.
- Garantir rastreabilidade de cargas, versões, reprocessamentos e acessos.
- Padronizar objetos, schemas, permissões e processos.
- Permitir evolução gradual sem dependência imediata de uma tecnologia específica.

## Visão de alto nível

```text
Fontes
  ├── ERP
  ├── APIs
  ├── CSV/XLSX
  ├── MicroSaaS
  └── Scraping
        ↓
DB_INGESTAO_* 
        ↓
DB_BRONZE_*
        ↓
DB_PRATA_*
        ↓
DB_OURO_*
        ↓
Camada Semântica (VW_*)
        ↓
Data Marts / Publicação (REL_*)
        ↓
Power BI | Excel | APIs | Aplicações
```

## Camadas

| Camada | Responsabilidade |
|---|---|
| Ingestão | Receber arquivos, respostas de APIs e resultados de automações; controlar fila, execução e metadados da carga. |
| Bronze | Preservar dados recebidos com mínima alteração e rastreabilidade de origem. |
| Prata | Padronizar tipos, nomenclaturas, chaves, qualidade e regras técnicas. |
| Ouro | Manter dimensões, fatos e estruturas analíticas corporativas. |
| Semântica | Centralizar regras de negócio, indicadores, descrições e cálculos reutilizáveis. |
| Publicação | Disponibilizar produtos de dados `REL_*` para consumidores autorizados. |

## Convenções de objetos

A camada é definida pelo banco de dados; por isso, não é necessário repetir `RAW` ou `STG` no nome das tabelas.

| Tipo | Padrão |
|---|---|
| Tabela | `TB_*` |
| Dimensão | `DIM_*` ou `TB_DIM_*`, conforme padrão adotado |
| Fato | `FAT_*` ou `TB_FAT_*`, conforme padrão adotado |
| View semântica | `VW_*` |
| View materializada / tabela de serving | `MV_*` quando aplicável |
| Produto de dados publicado | `REL_*` |
| Job / fluxo de orquestração | `JOB_*` |
| Role | `RL_*` |

Os schemas representam domínios ou departamentos, por exemplo: `fin`, `ctb`, `fis`, `cap`, `plf` e `mis`.

## Segurança

O acesso deve ser baseado em RBAC (Role-Based Access Control).

Exemplos:

```text
RL_DBA_PLATFORM
RL_PLATFORM_ADMIN
RL_DATA_ENGINEER
RL_OPERACIONAL
RL_BI
RL_BI_FIN
RL_BI_CTB
RL_BI_FIS
```

Consumidores devem receber acesso preferencialmente apenas aos objetos `REL_*` de seus Data Marts. Objetos técnicos das camadas Bronze, Prata, Ouro e semântica permanecem restritos a perfis técnicos e operacionais autorizados.

## Orquestração e versionamento

- **Python**: conectores de API, automações, scraping e tratamento específico de extração.
- **Worker/Kestra**: fila, agendamento, retries, dependências e observabilidade.
- **Git + CI/CD**: versionamento de SQL, Python, fluxos, documentação e deploy.
- **Auditoria**: registro de execução, origem, arquivo, hash, versão, usuário, status e mensagens de erro.

## Batch e near real-time

A arquitetura suporta:

- cargas mensais, semanais e diárias;
- reprocessamento de versões;
- ajustes manuais controlados;
- APIs consultadas em micro-batch, como a cada 60 segundos;
- eventos por webhook quando a fonte oferecer suporte.

Kafka não é obrigatório para integrações por polling de 60 segundos. Ele passa a ser relevante quando houver alto volume contínuo de eventos, múltiplos consumidores independentes ou necessidade de retenção e processamento de streams em grande escala.

## Serving Layer opcional

Quando necessário para desacoplar o consumo analítico ou melhorar desempenho, uma camada de serving pode ser publicada em ambiente separado, como PostgreSQL em EC2.

```text
Ouro / Semântica on-premise
        ↓
Carga incremental push
        ↓
Serving PostgreSQL
        ↓
Power BI / APIs
```

Essa camada deve conter tabelas materializadas, agregações e produtos de consumo. Ela não substitui a camada Ouro como fonte corporativa de verdade.

## Estrutura recomendada do repositório

```text
data-platform/
├── docs/
│   ├── documentacao_arquitetura_dados_v1.html
│   └── README_ARQUITETURA.md
├── sql/
│   ├── ingestao/
│   ├── bronze/
│   ├── prata/
│   ├── ouro/
│   ├── semantica/
│   └── publicacao/
├── python/
│   ├── extractors/
│   ├── loaders/
│   └── common/
├── orchestration/
│   ├── jobs/
│   └── flows/
├── tests/
└── deploy/
```

## Princípios operacionais

- Cada carga deve possuir identificador único de execução.
- Arquivos e extrações devem manter metadados de origem, data/hora, versão e hash.
- Ajustes não devem sobrescrever silenciosamente uma carga anterior.
- A publicação deve ocorrer somente após validações de qualidade.
- Regras de negócio não devem ser duplicadas no Power BI.
- Materialização deve ser aplicada com base em métricas de desempenho e necessidade de consumo.

## Próximos passos

1. Formalizar o catálogo de objetos e domínios.
2. Criar matriz Role × Data Mart × Produto de Dados.
3. Definir contratos de dados para cada fonte e produto publicado.
4. Implantar auditoria de cargas e qualidade.
5. Versionar todos os artefatos no Git.
6. Evoluir gradualmente a orquestração para Kestra, quando fizer sentido operacional.
