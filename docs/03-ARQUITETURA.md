# 03 — Arquitetura

## 1. Visão arquitetural

O Portal do Colaborador adotará uma arquitetura de microsserviços, com aplicações independentes, persistência isolada, comunicação síncrona via HTTP REST e comunicação assíncrona via Apache Kafka.

A arquitetura deverá garantir:

- Independência de desenvolvimento, implantação e escalabilidade das aplicações.
- Separação das responsabilidades de negócio.
- Consistência dos dados distribuídos.
- Resiliência a falhas.
- Segurança em múltiplas camadas.
- Observabilidade centralizada.
- Automação de testes e implantações.

## 2. Stack tecnológica

| Categoria | Tecnologia |
|---|---|
| Linguagem | Java 21 LTS |
| Framework | Spring Boot 4.x |
| APIs REST | Spring MVC |
| Segurança | Spring Security |
| Persistência | Spring Data JPA e Hibernate |
| Banco de dados | PostgreSQL |
| Migrações | Flyway |
| Mensageria | Apache Kafka e Spring Kafka |
| API Gateway | Spring Cloud Gateway |
| Cliente HTTP | HTTP Service Clients com RestClient |
| Resiliência | Resilience4j |
| Gerenciamento de dependências | Maven |
| Documentação de APIs | OpenAPI e Swagger |
| Containers | Docker |
| Ambiente local | Docker Compose |
| Orquestração | Kubernetes |
| Gerenciamento de deployments | Helm |
| CI/CD | GitHub Actions |
| Registro de imagens | GitHub Container Registry |
| Qualidade de código | SonarQube Cloud |
| Análise de vulnerabilidades | Trivy |

As versões exatas das dependências deverão ser definidas antes da implementação, considerando estabilidade e compatibilidade com Java 21, Spring Boot 4.x e Spring Cloud.

## 3. Aplicações e responsabilidades

### 3.1. api-gateway

Responsável pelo gerenciamento do tráfego externo.

- Roteamento de requisições.
- Validação inicial dos tokens de acesso.
- Configuração centralizada de CORS.
- Rate limiting.
- Propagação de identificadores de rastreamento.

Não deverá possuir regras de negócio nem acessar diretamente os bancos de dados dos microsserviços.

### 3.2. colaborador-service

Responsável pelo domínio cadastral e organizacional.

- Gerenciamento de colaboradores.
- Gerenciamento de gerentes.
- Gerenciamento de setores.
- Validação das regras de negócio cadastrais e organizacionais.
- Gerenciamento do estado cadastral das pessoas.
- Publicação dos eventos relacionados ao seu domínio.

Será a fonte de verdade dos dados cadastrais e organizacionais.

### 3.3. identidade-service

Responsável pelo domínio de identidade e acesso.

- Gerenciamento de contas de acesso.
- Autenticação.
- Credenciais.
- Emissão e validação de tokens.
- Gerenciamento e revogação de sessões.
- Convites para definição inicial de senha.
- Recuperação e alteração de senha.
- Bloqueio e desbloqueio de acesso.
- Consumo dos eventos relacionados à sincronização de identidade.

Será a fonte de verdade das credenciais, sessões e estados de segurança das contas.

## 4. Organização do monorepo

O projeto será mantido em um único repositório Git.

```text
portal-colaborador/
├── api-gateway/
├── colaborador-service/
├── identidade-service/
├── infrastructure/
│   ├── docker/
│   └── helm/
├── .github/
│   └── workflows/
├── docs/
│   └── adr/
├── compose.yml
├── AGENTS.md
└── README.md
```

Cada aplicação deverá possuir:

- Projeto Spring Boot independente.
- Arquivo `pom.xml` próprio.
- Código-fonte e testes independentes.
- Dockerfile próprio.
- Configurações específicas.
- Ciclo de build e implantação independente.

Inicialmente, não serão criados módulos Maven compartilhados entre as aplicações.

A infraestrutura deverá ser versionada no próprio repositório.

## 5. Arquitetura interna

Os microsserviços de negócio deverão utilizar arquitetura em camadas organizada por funcionalidade (*Package by Feature*).

Exemplo de organização:

```text
colaborador/
├── controller/
├── service/
├── repository/
├── entity/
├── dto/
└── mapper/

setor/
├── controller/
├── service/
├── repository/
├── entity/
├── dto/
└── mapper/
```

### 5.1. Responsabilidades das camadas

| Camada | Responsabilidade |
|---|---|
| Controller | Recebimento de requisições HTTP e definição das respostas. |
| Service | Regras de negócio, casos de uso e gerenciamento transacional. |
| Repository | Acesso à persistência. |
| Entity | Representação das entidades persistidas. |
| DTO | Representação dos dados de entrada e saída. |
| Mapper | Conversão entre entidades e DTOs. |

Componentes transversais, como configurações, segurança, tratamento de exceções e infraestrutura de mensageria, deverão possuir organização própria.

### 5.2. Diretrizes

- Utilizar injeção de dependências por construtor.
- Manter regras de negócio fora dos controllers.
- Não expor entidades JPA diretamente nos contratos REST.
- Definir limites transacionais na camada de serviço.
- Evitar dependências circulares.
- Não compartilhar entidades ou repositories entre microsserviços.

## 6. Persistência e migrações

### 6.1. Isolamento dos bancos de dados

Cada microsserviço de negócio possuirá seu próprio banco PostgreSQL.

| Microsserviço | Banco de dados |
|---|---|
| colaborador-service | `colaborador_db` |
| identidade-service | `identidade_db` |

Cada microsserviço será responsável exclusivamente pelo gerenciamento de seu banco.

Não será permitido:

- Acesso direto ao banco de outro microsserviço.
- Relacionamentos JPA entre entidades de microsserviços diferentes.
- Foreign keys entre bancos de microsserviços diferentes.
- Consultas ou alterações diretas em tabelas pertencentes a outro domínio.

Referências entre domínios deverão utilizar identificadores e mecanismos de comunicação entre serviços.

### 6.2. Migrações

O Flyway será responsável pelo gerenciamento das alterações estruturais dos bancos de dados.

- As migrações deverão ser versionadas por scripts SQL.
- Cada microsserviço possuirá suas próprias migrações.
- Migrações já aplicadas não deverão ser modificadas.
- Alterações posteriores deverão utilizar novas migrações.
- O Hibernate deverá validar o esquema existente utilizando `ddl-auto=validate`.
- O Hibernate não deverá criar ou alterar automaticamente a estrutura dos bancos.

## 7. Comunicação síncrona

A comunicação síncrona entre microsserviços utilizará HTTP REST.

Os clientes HTTP serão implementados utilizando HTTP Service Clients, com interfaces declarativas baseadas em `@HttpExchange` e `RestClient` como implementação subjacente.

A comunicação deverá contemplar:

- Autenticação entre microsserviços.
- Timeouts de conexão e resposta.
- Tratamento padronizado de erros.
- Propagação do contexto de rastreamento.
- Mecanismos de resiliência.

A comunicação interna ocorrerá diretamente entre os microsserviços, sem passar pelo API Gateway.

Operações que exigem confirmação imediata deverão utilizar comunicação síncrona.

Operações críticas de segurança somente poderão retornar sucesso após a confirmação das condições de segurança exigidas pelos requisitos funcionais.

Falhas de comunicação não deverão ser interpretadas como sucesso.

## 8. Comunicação assíncrona

O Apache Kafka será utilizado para publicação e consumo de eventos de negócio.

A comunicação assíncrona será utilizada principalmente para:

- Provisionamento de contas de acesso.
- Sincronização dos dados relevantes à identidade.
- Propagação de alterações cadastrais entre domínios.

A integração com Kafka deverá utilizar Spring Kafka.

### 8.1. Transactional Outbox

A publicação confiável de eventos deverá utilizar o padrão Transactional Outbox.

O microsserviço produtor deverá persistir a alteração de negócio e o respectivo evento Outbox na mesma transação do banco de dados.

A publicação no Kafka ocorrerá posteriormente, por meio de um publicador independente da transação de negócio.

Não deverá existir dependência de uma transação distribuída entre PostgreSQL e Kafka.

### 8.2. Polling Publisher

Os eventos Outbox serão publicados utilizando Polling Publisher, implementado com o mecanismo de agendamento do Spring.

O intervalo inicial será de 1 segundo, configurável externamente.

O publicador deverá:

- Consultar eventos pendentes utilizando consultas indexadas.
- Processar eventos em lotes.
- Continuar o processamento de lotes enquanto houver eventos pendentes elegíveis.
- Controlar a concorrência entre réplicas.
- Preservar a ordem de publicação por agregado.
- Realizar retentativas em caso de falha.
- Marcar eventos como publicados somente após confirmação do Kafka.
- Permitir recuperação de eventos cujo processamento tenha sido interrompido.
- Realizar limpeza periódica dos eventos publicados, conforme política de retenção.

O processamento concorrente não deverá permitir que versões posteriores de um mesmo agregado sejam publicadas antes de versões anteriores pendentes.

O intervalo, tamanho dos lotes e parâmetros de retentativa deverão ser configuráveis.

Inicialmente, não serão utilizados Debezium, Kafka Connect ou CDC.

### 8.3. Estrutura dos eventos

Os eventos deverão possuir, no mínimo:

| Campo | Responsabilidade |
|---|---|
| `eventId` | Identificador único do evento. |
| `eventType` | Identificação do tipo de evento. |
| `pessoaId` | Identificação da pessoa associada ao evento. |
| `aggregateVersion` | Versão do estado do agregado. |
| `data` | Dados necessários ao processamento. |

A estrutura definitiva dos eventos será estabelecida em `05-CONTRATOS.md`.

### 8.4. Particionamento e ordenação

O identificador `pessoaId` deverá ser utilizado como chave de particionamento dos eventos relacionados à pessoa.

Eventos cuja ordenação relativa seja necessária deverão ser publicados no mesmo tópico e direcionados à mesma partição.

A versão do agregado deverá:

- Ser persistida no banco de dados do produtor.
- Ser incrementada de maneira consistente a cada alteração relevante.
- Ser incluída nos eventos publicados.
- Permitir a identificação de eventos desatualizados.

Alterações na quantidade de partições deverão considerar seus impactos sobre a ordenação dos eventos.

### 8.5. Conteúdo dos eventos

Os eventos de sincronização de identidade deverão transportar snapshots dos dados relevantes ao serviço consumidor.

O snapshot deverá representar o estado necessário à sincronização, independentemente dos campos modificados na operação que originou o evento.

Não deverão ser compartilhados dados pessoais desnecessários, credenciais ou informações que não pertençam ao escopo da integração.

Os snapshots cadastrais não poderão sobrescrever estados de segurança gerenciados exclusivamente pelo `identidade-service`.

### 8.6. Idempotência

A arquitetura deverá considerar a possibilidade de entrega duplicada de eventos (*at-least-once*).

Os consumidores deverão:

- Identificar eventos pelo `eventId`.
- Registrar os eventos processados de forma persistente.
- Registrar o processamento e aplicar as alterações de negócio na mesma transação.
- Comparar a versão recebida com a última versão aplicada.
- Impedir efeitos duplicados.
- Impedir que eventos desatualizados sobrescrevam estados mais recentes.

A confirmação do processamento ao Kafka deverá ocorrer somente após a conclusão bem-sucedida da transação local.

Saltos de versão poderão ser aceitos quando o snapshot for suficiente para reconstruir o estado necessário e todas as regras de negócio forem preservadas.

### 8.7. Tratamento de falhas

Os consumidores deverão utilizar os mecanismos de Retry e Dead Letter Topic disponibilizados pelo Spring Kafka.

- Falhas transitórias deverão ser submetidas a retentativas.
- Falhas não recuperáveis deverão ser encaminhadas ao DLT.
- Eventos que esgotarem as retentativas deverão ser encaminhados ao DLT.
- Falhas deverão ser registradas e monitoradas.
- O reprocessamento deverá preservar a idempotência e as regras de versionamento.

O tratamento de falhas não deverá permitir que eventos posteriores comprometam a consistência do agregado.

A estratégia de retentativas, isolamento de agregados afetados e reprocessamento deverá ser detalhada antes da implementação dos consumidores.

## 9. Resiliência

A comunicação síncrona deverá utilizar Resilience4j.

### 9.1. Circuit Breaker

Deverá interromper temporariamente chamadas a serviços que apresentem falhas recorrentes, evitando falhas em cascata.

### 9.2. Retry

Deverá realizar retentativas limitadas em operações que possam ser repetidas com segurança.

Operações não idempotentes não deverão ser repetidas automaticamente sem mecanismos que garantam a segurança da repetição.

### 9.3. Bulkhead

Deverá limitar a utilização concorrente de recursos destinados à comunicação entre microsserviços.

### 9.4. Configuração

Os parâmetros de resiliência deverão ser configuráveis externamente e ajustados com base em testes.

Não deverão existir mecanismos de fallback que retornem sucesso indevido ou comprometam regras de segurança e consistência.

## 10. API Gateway e Service Discovery

### 10.1. API Gateway

O Spring Cloud Gateway será o ponto único de entrada das requisições externas.

Os microsserviços de negócio não deverão ser expostos diretamente ao tráfego externo.

A validação de tokens realizada pelo Gateway não substituirá os mecanismos de autenticação e autorização dos microsserviços.

### 10.2. Service Discovery

A descoberta de serviços utilizará os mecanismos nativos do Kubernetes.

A comunicação deverá utilizar os nomes DNS dos Kubernetes Services, sem dependência de endereços IP fixos dos Pods.

O balanceamento de carga interno será realizado pela infraestrutura do Kubernetes.

Não será utilizado Netflix Eureka.

## 11. Configurações e ambientes

### 11.1. Gerenciamento de configurações

| Recurso | Responsabilidade |
|---|---|
| `application.yml` | Configurações padrão. |
| Spring Profiles | Configurações específicas por ambiente. |
| Kubernetes ConfigMap | Configurações externas não sensíveis. |
| Kubernetes Secret | Credenciais e informações sensíveis. |

As configurações externas deverão ser disponibilizadas às aplicações por meio de variáveis de ambiente.

Informações sensíveis não deverão ser versionadas no Git nem incorporadas às imagens Docker.

Em produção, os Secrets deverão possuir criptografia em repouso e controle de acesso por RBAC.

Não será utilizado Spring Cloud Config.

### 11.2. Ambientes

| Ambiente | Profile | Infraestrutura |
|---|---|---|
| Desenvolvimento | `dev` | Docker Compose |
| Homologação | `staging` | Kubernetes |
| Produção | `prod` | Kubernetes |

Os ambientes deverão possuir isolamento de bancos de dados, credenciais, recursos Kafka e configurações.

Dados reais de produção não deverão ser utilizados nos ambientes de desenvolvimento e homologação.

A mesma imagem Docker validada em homologação deverá ser promovida para produção, sem nova compilação.

## 12. Observabilidade

A observabilidade deverá contemplar métricas, logs e rastreamento distribuído.

| Tecnologia | Responsabilidade |
|---|---|
| Spring Boot Actuator | Health checks e informações operacionais. |
| Micrometer | Instrumentação de métricas. |
| Prometheus | Coleta e armazenamento de métricas. |
| Grafana | Visualização e dashboards. |
| Loki | Centralização de logs. |
| OpenTelemetry | Instrumentação e propagação de rastreamento. |
| Tempo | Armazenamento de traces distribuídos. |

### 12.1. Métricas

Deverão ser monitorados:

- Latência das requisições.
- Taxa de erros.
- Consumo de CPU e memória.
- Pool de conexões dos bancos de dados.
- Quantidade de eventos pendentes no Outbox.
- Tempo de permanência dos eventos no Outbox.
- Falhas de publicação e consumo.
- Consumer lag do Kafka.
- Eventos encaminhados ao DLT.
- Estado dos Circuit Breakers.

### 12.2. Logs e rastreamento

Os logs deverão permitir correlação com os traces distribuídos.

O contexto de rastreamento deverá ser propagado nas comunicações HTTP e Kafka.

Senhas, tokens, credenciais e dados pessoais sensíveis não deverão ser registrados nos logs.

As aplicações deverão disponibilizar health checks adequados à execução em Kubernetes.

## 13. Estratégia de testes

A implementação deverá contemplar testes unitários, de integração, de contratos e End-to-End.

| Tecnologia | Responsabilidade |
|---|---|
| JUnit 5 | Testes automatizados. |
| Mockito | Isolamento de dependências. |
| Testcontainers | Testes com PostgreSQL e Kafka reais. |
| MockMvc | Testes dos endpoints Spring MVC. |
| WireMock | Simulação de integrações HTTP. |
| JaCoCo | Análise de cobertura de código. |

O H2 não deverá ser utilizado como substituto do PostgreSQL nos testes de integração.

Os cenários obrigatórios, critérios de cobertura e ferramentas complementares serão definidos em `06-TESTES.md`.

A compatibilidade das versões das ferramentas de testes com o Spring Boot adotado deverá ser validada antes da implementação.

## 14. Infraestrutura e CI/CD

### 14.1. Docker

Cada aplicação deverá possuir sua própria imagem Docker.

O Docker Compose será utilizado para disponibilizar a infraestrutura necessária ao desenvolvimento local.

### 14.2. Kubernetes

Os ambientes de homologação e produção utilizarão Kubernetes.

A infraestrutura deverá permitir:

- Implantação independente das aplicações.
- Escalabilidade horizontal.
- Balanceamento de carga.
- Health checks.
- Gerenciamento externo de configurações.
- Isolamento dos serviços internos.

Os recursos de implantação deverão ser gerenciados utilizando Helm.

### 14.3. Continuous Integration

O GitHub Actions deverá automatizar:

- Compilação.
- Execução de testes.
- Análise de qualidade de código.
- Verificação de vulnerabilidades.

O SonarQube Cloud será utilizado para análise estática e qualidade de código.

O Trivy será utilizado para análise de vulnerabilidades em dependências e imagens Docker.

### 14.4. Continuous Delivery

O pipeline deverá:

- Construir as imagens Docker das aplicações afetadas.
- Publicar as imagens no GitHub Container Registry.
- Implantar automaticamente as versões aprovadas em homologação.
- Exigir aprovação manual para implantação em produção.
- Promover para produção a mesma imagem validada em homologação.

Os pipelines deverão identificar as aplicações afetadas pelas alterações, incluindo impactos decorrentes de mudanças na infraestrutura.

Os detalhes operacionais serão definidos em `09-OPERACAO.md`.

## 15. Restrições arquiteturais

- Cada microsserviço deverá possuir autonomia sobre seu domínio e sua persistência.
- Não deverá existir compartilhamento direto de bancos de dados entre microsserviços.
- A comunicação assíncrona não deverá substituir confirmações síncronas exigidas por operações críticas de segurança.
- A arquitetura deverá tolerar duplicidade de eventos sem produzir efeitos duplicados.
- As aplicações não deverão depender exclusivamente do API Gateway para garantir segurança.
- Configurações sensíveis não deverão ser versionadas no repositório.
- Alterações arquiteturais relevantes deverão ser documentadas antes da implementação.
- Decisões arquiteturais relevantes deverão ser registradas em `docs/adr/`.

## 16. Documentos complementares

| Documento | Responsabilidade |
|---|---|
| `01-VISAO-GERAL.md` | Contexto e objetivos do projeto. |
| `02-REQUISITOS.md` | Requisitos funcionais e regras de negócio. |
| `04-SEGURANCA.md` | Arquitetura e mecanismos de segurança. |
| `05-CONTRATOS.md` | Contratos REST e eventos Kafka. |
| `06-TESTES.md` | Estratégia detalhada de testes. |
| `07-ROADMAP.md` | Planejamento incremental da implementação. |
| `08-PROGRESSO.md` | Acompanhamento da evolução do projeto. |
| `09-OPERACAO.md` | Infraestrutura, implantação e procedimentos operacionais. |

As decisões específicas desses documentos deverão respeitar as diretrizes arquiteturais estabelecidas neste documento.