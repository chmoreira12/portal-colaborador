# AGENTS.md — Portal do Colaborador

## 1. Objetivo deste arquivo

Este arquivo estabelece as instruções permanentes para agentes de IA que trabalham no Portal do Colaborador, especialmente o Codex.

Desenvolver o projeto com organização, rastreabilidade, qualidade técnica e continuidade entre sessões. O repositório, sua documentação versionada e o estado verificável do código são as fontes oficiais de contexto. Não depender exclusivamente da memória de conversas anteriores.

## 2. Contexto do projeto

O Portal do Colaborador é uma plataforma corporativa de serviços aos colaboradores de uma organização. A versão inicial contempla gestão de colaboradores, gerentes, setores, contas de acesso, ativação, autenticação e autorização.

O projeto tem finalidade educacional e de portfólio profissional, com foco na preparação para atuação Java/Spring Boot Pleno e no ciclo completo de desenvolvimento, do código à produção. Priorizar tecnologias maduras, compatíveis entre si e relevantes para o mercado corporativo brasileiro. Não adicionar complexidade apenas para demonstrar ferramentas.

### Tecnologias aprovadas

- Java 21 LTS, Spring Boot 4.x, Maven, Spring MVC.
- Spring Security, OAuth2 Resource Server, OAuth2 Client, Authorization Server, JWT e BCrypt.
- Spring Data JPA, Hibernate, PostgreSQL e Flyway.
- Apache Kafka, Spring Kafka e Transactional Outbox.
- Redis, Spring Cache e Spring Data Redis.
- Spring Cloud Gateway, HTTP Service Clients, RestClient e Resilience4j.
- Angular, TypeScript, Angular Material, RxJS, npm e Nginx.
- Spring Boot Mail, SMTP e Mailpit.
- JUnit Jupiter, Mockito, Testcontainers, MockMvc, WireMock e JaCoCo.
- OpenAPI/Swagger, Docker, Docker Compose, Kubernetes e Helm.
- GitHub Actions, Amazon ECR, SonarQube Cloud e Trivy.
- Spring Boot Actuator, Micrometer, Prometheus, Grafana, Loki, OpenTelemetry e Tempo.
- Terraform e AWS: EKS, RDS PostgreSQL, MSK, ElastiCache compatível com Redis OSS, SES, VPC, IAM, Secrets Manager, KMS, ALB, ACM, S3 e AWS Budgets.

A presença de uma tecnologia nesta lista não significa que esteja implementada. Consultar `docs/03-ARQUITETURA.md` e `docs/04-SEGURANCA.md` para diretrizes detalhadas. Verificar versões estáveis e compatíveis nas fontes oficiais antes de incorporar dependências; registrar as versões efetivamente adotadas nos arquivos de build e na documentação pertinente. Não substituir tecnologias aprovadas silenciosamente.

## 3. Arquitetura

- `colaborador-service`: dados cadastrais e profissionais de gerentes e colaboradores e gestão de setores; fonte de verdade cadastral.
- `identidade-service`: contas, credenciais, ativação, autenticação, autorização, sessões e estados de segurança; fonte de verdade da identidade.
- `api-gateway`: entrada pública das APIs, roteamento e controles de entrada, sem regras de negócio ou acesso direto aos bancos.
- `frontend`: SPA Angular, consumindo a API pública do Gateway; controles visuais não substituem autorização no backend.

Manter aplicações independentes, persistência isolada, comunicação síncrona HTTP e eventos Kafka conforme arquitetura e contratos. Não acessar diretamente bancos de outros serviços. Aplicar Outbox, idempotência, versionamento e Saga Orquestrada onde definidos. Redis não é fonte de verdade de segurança.

Detalhes de contratos, consistência, segurança, tratamento de falhas e infraestrutura devem ser formalizados antes da implementação correspondente.

## 4. Regras de negócio inicialmente aprovadas

Existem somente dois papéis: GERENTE e COLABORADOR.

### Gerente

- Pode cadastrar outros gerentes e colaboradores.
- Pode cadastrar e gerenciar setores.
- Pode consultar registros de gerentes e colaboradores.
- Possui acesso administrativo às funcionalidades da versão inicial, respeitando as restrições dos requisitos.
- Não precisa estar vinculado a um setor.

### Colaborador

- Deve estar vinculado a um setor.
- Pode ativar sua conta, autenticar-se e consultar seus próprios dados.
- Não pode consultar dados de outros usuários nem possui permissões administrativas.

### Fora do escopo inicial

- Liderança ou chefia de setores.
- Papéis adicionais.
- Hierarquias organizacionais avançadas.
- Módulos de requerimentos, agendamentos e documentos funcionais.

Essas funcionalidades poderão ser avaliadas em versões futuras. As regras detalhadas e exceções constam em `docs/02-REQUISITOS.md`, que prevalece sobre este resumo funcional.

## 5. Documentação oficial

- `docs/01-VISAO-GERAL.md`: contexto, objetivos e escopo.
- `docs/02-REQUISITOS.md`: requisitos e regras de negócio.
- `docs/03-ARQUITETURA.md`: decisões arquiteturais e tecnologias.
- `docs/04-SEGURANCA.md`: autenticação, autorização e segurança distribuída.
- `docs/05-CONTRATOS.md`: contratos REST, internos e Kafka.
- `docs/06-TESTES.md`: estratégia e critérios de validação.
- `docs/07-ROADMAP.md`: sequência dos incrementos e critérios de conclusão.
- `docs/08-PROGRESSO.md`: estado real, histórico e próxima tarefa.
- `docs/09-OPERACAO.md`: infraestrutura, deploy, operação e recuperação.
- `docs/adr/`: registros de decisões arquiteturais significativas.

Arquivos ainda não criados não devem ser tratados como documentação existente. Antes de implementar, consultar os documentos relevantes e garantir que as decisões necessárias estejam registradas.

O progresso não redefine requisitos nem arquitetura. ADRs históricos superados devem indicar a decisão que os substituiu. Em divergências, identificar as fontes e o estado real, sem resolver silenciosamente conflitos importantes.

## 6. Procedimento obrigatório no início de cada sessão

Antes de implementar:

1. Ler este arquivo.
2. Identificar branch, HEAD, estado do repositório e alterações não commitadas, quando houver acesso ao ambiente local.
3. Ler `docs/08-PROGRESSO.md`, quando existir, e verificar o estado registrado contra o código e histórico Git.
4. Consultar roadmap e requisitos relacionados à tarefa.
5. Consultar arquitetura, segurança, contratos, estratégia de testes, operação e ADRs relevantes.
6. Inspecionar o código e os testes existentes.
7. Identificar o que está implementado, pendente ou bloqueado.
8. Apresentar resumo objetivo do estado e da próxima etapa antes de propor alterações.

Não presumir que uma funcionalidade existe porque foi documentada. Se o acesso for exclusivamente remoto, distinguir o estado da branch consultada do estado local, que não pode ser verificado remotamente. Informar documentos ou verificações indisponíveis.

## 7. Regras para implementação e autonomia

- Trabalhar em etapas pequenas, verificáveis e sequenciais, de acordo com o roadmap.
- Explicar objetivo, arquivos afetados e impactos; não pular procedimentos importantes.
- Priorizar soluções profissionais apropriadas ao contexto, evitando implementações provisórias e refatorações desnecessárias.
- Não adicionar funcionalidades fora do escopo aprovado.
- Não modificar contratos ou decisões arquiteturais sem analisar seus impactos e atualizar a documentação pertinente.
- Não alterar código não relacionado à tarefa sem justificativa.
- Não substituir decisões registradas por preferências do agente.
- Não inventar classes, endpoints, arquivos, testes ou resultados; não tratar funcionalidades planejadas como concluídas.
- Preservar a separação de responsabilidades entre microsserviços.
- Considerar segurança, persistência, transações, tratamento de erros, observabilidade e testes nas alterações relevantes.

O mentor técnico possui autonomia para definir decisões arquiteturais restantes dentro do escopo e critérios aprovados, sem exigir aprovação individual a cada escolha. O Codex deve respeitar as decisões documentadas e resolver detalhes ordinários de implementação dentro delas; não redefinir a arquitetura por conta própria. Lacunas arquiteturais relevantes devem ser encaminhadas ao mentor para definição e registro antes da implementação afetada. Solicitar orientação do usuário quando a decisão alterar significativamente escopo, regras de negócio ou custos, envolver operações destrutivas ou exigir autorização de execução ainda não concedida.

Autonomia técnica não constitui autorização para modificar arquivos, repositórios, ambientes ou recursos externos. O mentor entrega orientações, documentos e instruções para o Codex; não executa alterações diretamente sem pedido explícito do usuário. O Codex pode alterar arquivos no ambiente de trabalho somente no escopo da tarefa de implementação expressamente autorizada. Não realizar deploy, provisionamento pago ou push por inferência.

## 8. Segurança

- Seguir `docs/04-SEGURANCA.md` e os requisitos aplicáveis.
- Nunca versionar senhas, tokens reais, chaves privadas ou credenciais.
- Não utilizar senhas fixas ou credenciais administrativas inseguras.
- Não criar endpoints administrativos públicos para facilitar testes.
- Aplicar autorização no backend, independentemente da interface.
- Não confiar em identificadores enviados pelo cliente para determinar a identidade autenticada.
- Proteger dados pessoais e respeitar os princípios aplicáveis da LGPD.
- Documentar decisões relevantes de autenticação, autorização e gerenciamento de credenciais.

## 9. Testes e validação

Toda alteração deve ser validada de acordo com sua natureza e `docs/06-TESTES.md`.

Antes de considerar uma tarefa concluída:

1. Verificar compilação, quando aplicável.
2. Executar os testes relevantes.
3. Verificar impactos sobre funcionalidades existentes.
4. Conferir regras de negócio e contratos afetados.
5. Informar quais verificações foram executadas e os respectivos resultados.
6. Registrar limitações, falhas e testes não executados.

Nunca afirmar que um teste passou sem executá-lo. Não utilizar H2 como substituto do PostgreSQL em testes de integração. Não desativar controles de qualidade ou segurança para fazer o pipeline passar.

## 10. Atualização da documentação

A documentação deve acompanhar a evolução do código. Alterações autorizadas em regras de negócio, arquitetura, contratos, infraestrutura ou procedimentos operacionais devem atualizar os documentos correspondentes.

Decisões arquiteturais significativas devem ser registradas em ADRs com contexto, decisão, alternativas relevantes, consequências e status. Não reescrever ADRs históricos de modo a ocultar decisões anteriores.

Se código e documentação divergirem, identificar a divergência e sua causa. Encaminhar decisões arquiteturais ao mentor e questões de produto ao usuário quando necessário; não efetuar mudanças conflitantes sem orientação e autorização aplicáveis.

## 11. Controle de progresso

`docs/08-PROGRESSO.md` é a referência operacional para retomada do trabalho. Atualizá-lo no mesmo incremento de cada avanço relevante, inclusive quando o avanço for exclusivamente documental.

Registrar:

- Marco atual e tarefas concluídas, em andamento ou pendentes.
- Arquivos ou componentes importantes alterados.
- Testes e verificações executados, com resultados observados.
- Pendências, bloqueios e decisões tomadas, com referências aos documentos/ADRs.
- Próxima tarefa específica, dependências e critério de conclusão.
- Referência de commit, quando disponível.

Manter um resumo atual no início do arquivo e histórico cronológico dos avanços relevantes. Não apagar histórico útil. Não marcar como concluído algo apenas planejado ou não validado; explicitar o estado de validação. Quando o commit da tarefa ainda não existir, não inventar seu hash: registrar referência em atualização posterior quando apropriado.

## 12. Git e GitHub

- Verificar alterações locais antes de modificar arquivos; preservar trabalho do usuário.
- Propor commits pequenos e coerentes, informando quando a etapa estiver pronta para commit.
- Usar mensagens Conventional Commits em inglês (ex.: `feat: add sector registration`, `docs: define API contracts`).
- O usuário executa commits e pushes, salvo autorização explícita em contrário.
- O mentor pode consultar o GitHub, mas não modificar arquivos, branches, commits ou outros recursos remotos sem autorização explícita.
- Não executar operações destrutivas de Git sem autorização explícita.
- Não afirmar que commit ou push ocorreu sem confirmação.

## 13. Comunicação e mentoria

O usuário deseja compreender o desenvolvimento, não apenas receber código pronto. O planejamento arquitetural pode ser concluído previamente; o aprofundamento didático ocorre durante a implementação de cada incremento.

Em cada etapa:

1. Explicar o objetivo, o conceito técnico aplicado e o problema que resolve, com conexão à prática profissional.
2. Informar os arquivos, comandos e ações necessários e preparar instruções claras para o Codex.
3. Implementar somente o incremento autorizado, sem misturar tarefas independentes.
4. Analisar o código real produzido, explicando classes, métodos, anotações, configurações, padrões e decisões relevantes.
5. Validar o comportamento com testes e verificações, explicando o que demonstram.
6. Atualizar a documentação e o progresso e orientar commit/push.

Utilizar linguagem objetiva, clara e didática, evitando repetição e teoria desconectada da tarefa. Não exigir que o usuário aprove escolhas arquiteturais ordinárias já delegadas. Não omitir passos importantes por considerá-los óbvios. A finalidade é que o usuário consiga implementar, diagnosticar e defender tecnicamente as decisões em entrevistas e no trabalho profissional.

## 14. Encerramento de cada tarefa

Ao finalizar, apresentar:

1. O que foi realizado.
2. Arquivos criados ou alterados.
3. Testes e verificações executados e seus resultados.
4. Problemas, limitações e pendências restantes.
5. Documentos atualizados, incluindo o progresso quando aplicável.
6. Se a tarefa está pronta para commit e push, com comandos sugeridos.
7. Próxima tarefa recomendada, sem iniciar implementação independente sem autorização.

A continuidade, a rastreabilidade e a integridade arquitetural são prioridades permanentes deste projeto.
