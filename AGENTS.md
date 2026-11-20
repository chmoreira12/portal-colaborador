# AGENTS.md — Portal do Colaborador

## 1. Objetivo deste arquivo

Este arquivo estabelece as instruções permanentes para agentes de IA que trabalham no projeto Portal do Colaborador, especialmente o Codex.

O projeto deve ser desenvolvido com organização, rastreabilidade, qualidade técnica e continuidade entre sessões.

Nenhum agente deve depender exclusivamente da memória de conversas anteriores.

## 2. Contexto do projeto

O Portal do Colaborador é uma plataforma corporativa de serviços aos colaboradores de uma organização.

O objetivo da versão inicial é disponibilizar uma base funcional para gestão de colaboradores, setores, contas de acesso, ativação de usuários, autenticação e autorização.

O projeto também tem finalidade educacional e de portfólio profissional, com foco em práticas de engenharia de software aplicáveis ao mercado de desenvolvimento Java/Spring Boot.

### Tecnologias planejadas

- Java e Spring Boot
- Spring Security
- Spring Data JPA e Hibernate
- PostgreSQL
- Apache Kafka
- Docker
- Kubernetes
- Maven
- Flyway
- JUnit, Mockito e Testcontainers
- OpenAPI/Swagger
- GitHub Actions
- Ferramentas de observabilidade a serem definidas

A presença de uma tecnologia nesta lista não significa que ela já foi implementada. Versões, bibliotecas, configurações e estratégias de utilização devem ser registradas na documentação técnica.

## 3. Arquitetura inicial

O projeto prevê dois microsserviços:

### colaborador-service

Responsável pelos dados cadastrais e profissionais de gerentes e colaboradores, além da gestão de setores.

### identidade-service

Responsável pelo ciclo de vida das contas de acesso, credenciais, ativação, autenticação e autorização.

Os serviços devem possuir responsabilidades bem definidas e persistência sob controle de cada serviço.

A integração assíncrona com Apache Kafka está prevista para eventos de negócio, incluindo o provisionamento de contas após o cadastro de colaboradores.

Detalhes de contratos, consistência, segurança, tratamento de falhas e infraestrutura devem ser formalizados antes da implementação correspondente.

## 4. Regras de negócio inicialmente aprovadas

Existem somente dois papéis de acesso:

- GERENTE
- COLABORADOR

### Gerente

- Pode cadastrar outros gerentes.
- Pode cadastrar colaboradores.
- Pode cadastrar e gerenciar setores.
- Pode consultar os registros de gerentes e colaboradores.
- Possui acesso administrativo às funcionalidades disponíveis na versão inicial.
- Não precisa estar vinculado a um setor.

### Colaborador

- Deve estar vinculado a um setor.
- Pode ativar sua conta.
- Pode autenticar-se.
- Pode consultar seus próprios dados.
- Não pode consultar dados de outros usuários.
- Não possui permissões administrativas.

### Fora do escopo inicial

- Liderança ou chefia de setores.
- Papéis adicionais.
- Hierarquias organizacionais avançadas.
- Módulos de requerimentos, agendamentos e documentos funcionais.

Essas funcionalidades poderão ser avaliadas em versões futuras.

As regras detalhadas deverão ser mantidas no documento de requisitos.

## 5. Documentação oficial

Os documentos do projeto ficarão no diretório `docs/`.

Estrutura documental planejada:

- `01-VISAO-GERAL.md`
- `02-REQUISITOS.md`
- `03-ARQUITETURA.md`
- `04-SEGURANCA.md`
- `05-CONTRATOS.md`
- `06-TESTES.md`
- `07-ROADMAP.md`
- `08-PROGRESSO.md`
- `09-OPERACAO.md`
- `adr/` — registros de decisões arquiteturais

Arquivos ainda não criados não devem ser tratados como documentação existente.

Antes de implementar uma funcionalidade, consultar os documentos relevantes e verificar se as decisões necessárias estão registradas.

## 6. Procedimento obrigatório no início de cada sessão

Antes de qualquer implementação:

1. Ler este arquivo.
2. Identificar a branch atual.
3. Verificar o estado do repositório e alterações não commitadas.
4. Ler `docs/08-PROGRESSO.md`, quando existir.
5. Consultar o roadmap e os requisitos relacionados à tarefa.
6. Consultar a arquitetura, os contratos e os ADRs relevantes.
7. Inspecionar o código e os testes existentes.
8. Identificar o que está implementado, o que está pendente e o que precisa ser decidido.
9. Apresentar um resumo do estado atual antes de propor alterações.

Não presumir que uma funcionalidade existe apenas porque foi mencionada na documentação.

## 7. Regras para implementação

- Trabalhar em etapas pequenas, verificáveis e sequenciais.
- Não pular procedimentos por considerá-los intuitivos.
- Explicar o objetivo, os arquivos afetados e os impactos de cada alteração.
- Priorizar soluções profissionais apropriadas ao contexto.
- Evitar implementações provisórias que provoquem refatorações desnecessárias.
- Não adicionar funcionalidades fora do escopo aprovado.
- Não modificar contratos ou decisões arquiteturais sem analisar seus impactos.
- Não alterar código não relacionado à tarefa sem justificativa.
- Não substituir decisões registradas por preferências do agente.
- Não inventar classes, endpoints, arquivos, testes ou resultados de execução.
- Não tratar funcionalidades planejadas como concluídas.
- Preservar a separação de responsabilidades entre microsserviços.
- Considerar segurança, persistência, transações, tratamento de erros e testes nas alterações relevantes.

Quando houver ambiguidade importante, apresentar as alternativas e solicitar uma decisão antes de implementar.

## 8. Segurança

- Nunca incluir senhas, tokens reais, chaves privadas ou credenciais no repositório.
- Não utilizar senhas fixas ou credenciais administrativas inseguras.
- Não criar endpoints administrativos públicos para facilitar testes.
- Aplicar autorização no backend, independentemente das restrições da interface.
- Não confiar em identificadores enviados pelo cliente para determinar a identidade autenticada.
- Proteger dados pessoais e respeitar os princípios aplicáveis da LGPD.
- Documentar decisões relevantes de autenticação, autorização e gerenciamento de credenciais.

## 9. Testes e validação

Toda alteração deve ser validada de acordo com sua natureza.

Antes de considerar uma tarefa concluída:

1. Verificar a compilação, quando aplicável.
2. Executar os testes relevantes.
3. Verificar impactos sobre funcionalidades existentes.
4. Conferir as regras de negócio envolvidas.
5. Conferir os contratos afetados.
6. Informar quais verificações foram executadas.
7. Registrar limitações, falhas ou testes não executados.

Nunca afirmar que um teste passou sem tê-lo executado.

## 10. Atualização da documentação

A documentação deve acompanhar a evolução do código.

Ao modificar regras de negócio, arquitetura, contratos, infraestrutura ou procedimentos operacionais, atualizar os documentos correspondentes.

Decisões arquiteturais significativas devem ser registradas em ADRs, com contexto, decisão, alternativas relevantes e consequências.

Se código e documentação divergirem, identificar a divergência e solicitar orientação antes de realizar correções que envolvam decisões de produto ou arquitetura.

## 11. Controle de progresso

O arquivo `docs/08-PROGRESSO.md` será a referência para retomada do trabalho.

Após cada avanço relevante, registrar:

- Marco ou etapa atual.
- Entregas concluídas.
- Arquivos ou componentes importantes alterados.
- Testes e verificações executados.
- Pendências e bloqueios.
- Decisões tomadas.
- Próxima tarefa recomendada.
- Referência de commit, quando disponível.

Não apagar o histórico relevante de decisões e entregas.

Quando o commit da própria tarefa ainda não existir, não inventar seu hash. Registrar a referência após sua criação ou na próxima atualização apropriada.

## 12. Git e GitHub

- Verificar alterações locais antes de modificar arquivos.
- Não sobrescrever trabalho existente sem autorização.
- Propor commits pequenos e coerentes.
- Informar quando uma etapa estiver pronta para commit.
- Sugerir mensagens de commit descritivas.
- Não executar operações destrutivas de Git sem autorização explícita.
- Não realizar push automaticamente sem autorização ou instrução específica do usuário.
- Não afirmar que um commit ou push ocorreu sem confirmação.

## 13. Comunicação e mentoria

O usuário deseja compreender o desenvolvimento, não apenas receber código pronto.

Portanto:

- Explicar conceitos técnicos relevantes.
- Justificar decisões de arquitetura e implementação.
- Informar exatamente onde executar comandos.
- Informar quais arquivos criar ou modificar.
- Explicar como verificar se cada etapa funcionou.
- Trabalhar uma etapa por vez.
- Evitar respostas que agrupem várias implementações independentes sem necessidade.
- Não omitir passos importantes por considerá-los óbvios.

## 14. Encerramento de cada tarefa

Ao finalizar uma tarefa, apresentar:

1. O que foi realizado.
2. Quais arquivos foram criados ou alterados.
3. Quais testes e verificações foram executados.
4. Quais problemas ou pendências permanecem.
5. Quais documentos foram atualizados.
6. Se a tarefa está pronta para commit e push.
7. Qual é o próximo passo recomendado.

A continuidade, a rastreabilidade e a integridade arquitetural são prioridades permanentes deste projeto.