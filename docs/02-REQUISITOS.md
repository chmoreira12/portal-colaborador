# Portal do Colaborador — Requisitos consolidados

**Versão-alvo:** 1.0  
**Status:** Consolidado para planejamento incremental; decisões técnicas pendentes identificadas  
**Origem:** revisão do `REQUISITOS.md` fornecido pelo projeto.

## 1. Objetivo e convenções

Este documento define o comportamento esperado do Portal do Colaborador. Mantém os identificadores **RF-001 a RF-021** da versão original. Regras transversais são descritas uma única vez e aplicam-se a todos os requisitos pertinentes. Os **cenários de aceite** destacam verificações relevantes, sem reproduzir cada regra em outras palavras.

Os identificadores históricos RN-001 a RN-196 e CA-001 a CA-205 permanecem consultáveis no documento original; **não devem ser utilizados como referências novas** após a adoção desta versão. Para novas implementações e testes, a referência estável é o RF e o cenário de aceite associado.

## 2. Regras transversais

### 2.1 Papéis e autorização

- Existem somente os papéis **GERENTE** e **COLABORADOR** no escopo atual.
- Operações administrativas de setores, colaboradores e gerentes são exclusivas de gerentes autenticados e autorizados. Colaboradores podem consultar somente os próprios dados por operação específica.
- Autenticação não dispensa autorização por papel, propriedade do recurso, situação cadastral e bloqueios de segurança. O backend deve realizar essas verificações; dados de identidade ou papel enviados arbitrariamente pelo cliente não são confiáveis.
- A identidade do titular ou do administrador que executa a operação deve vir do contexto de segurança validado. Recursos protegidos exigem autenticação; como regra geral, ausência de autenticação válida resulta em HTTP **401** e permissão insuficiente em **403**.
- Consultas e logs devem respeitar minimização e proteção de dados pessoais. Senhas, tokens completos e segredos não podem ser expostos em respostas ou registros.

### 2.2 Integridade cadastral

- CPF válido, matrícula funcional e e-mail corporativo válido devem ser **únicos entre todas as pessoas cadastradas**, incluindo colaboradores e gerentes, tanto no cadastro quanto na atualização.
- Campos obrigatórios devem ser validados. Identificadores internos e datas de criação geradas pelo sistema são imutáveis.
- A inativação de setores, colaboradores ou gerentes é **lógica**: não exclui fisicamente registros nem apaga dados e vínculos históricos.
- O estado cadastral de uma pessoa é distinto do estado de sua conta de acesso e de bloqueios de segurança independentes.

### 2.3 Responsabilidades dos serviços e integração

- O `colaborador-service` é a fonte oficial dos dados pessoais, funcionais, setores e situação cadastral. O `identidade-service` controla contas, papéis de acesso, credenciais, autenticação, sessões e bloqueios próprios. Nenhum serviço altera diretamente a base de dados do outro.
- O cadastro de pessoas e as alterações cadastrais relevantes à identidade são comunicados **assincronamente pelo Kafka**. Eventos duplicados ou fora de ordem não podem causar duplicidade nem sobrescrever estados mais recentes. Falhas temporárias não podem perder definitivamente solicitações ou alterações.
- Alterações relevantes incluem e-mail corporativo, inativação/reativação e eventuais mudanças de papel **somente se uma operação específica vier a ser autorizada**. A atualização cadastral comum não altera papéis.
- Alterações sensíveis não podem abrir janelas de acesso indevido. Uma operação dependente de bloqueio/revogação não pode ser declarada integralmente concluída enquanto seu efeito de segurança não estiver garantido. Deve ser possível identificar integrações pendentes, concluídas e com falha.
- Eventos e operações sensíveis exigem rastreabilidade/auditoria, sem registrar credenciais. Estratégias concretas de consistência distribuída, publicação confiável, idempotência, ordenação, retentativas e observabilidade serão definidas na arquitetura.

### 2.4 Segurança de credenciais e sessões

- Somente o `identidade-service` manipula senhas, tokens e sessões. Senhas são armazenadas por hash adequado, nunca em texto puro. Tokens de convite e recuperação são criptograficamente seguros, temporários, de uso único e protegidos em armazenamento.
- A autenticação utiliza **Access Token de curta duração + Refresh Token**, com rotação e revogação controladas. O Refresh Token renova a sessão; não acessa recursos protegidos diretamente.
- Inativação, bloqueio e revogação devem impedir **novos logins, renovações e uso de Access Tokens já emitidos**, mesmo antes de expirarem. Reativação cadastral não remove bloqueios de segurança independentes.
- Processos de ativação, recuperação, alteração de senha e autenticação devem ter proteção contra abuso, e respostas públicas não devem permitir enumeração indevida de contas.
- Operações concorrentes não podem permitir reutilização de tokens de uso único ou desfazer revogações. A estratégia de revogação distribuída será definida antes da implementação.

## 3. Gestão de Setores

### RF-001 — Cadastrar setor

**Objetivo:** gerente cadastra um setor com os campos abaixo.

| Campo | Obrigatório | Regra |
|---|---|---|
| Nome | Sim | Nome completo |
| Sigla | Sim | Única entre setores |
| Descrição | Não | Informação complementar |
| Situação | Sim | Ativo ou inativo |
| Data de criação | Sim | Gerada automaticamente pelo sistema |

**Aceite essencial:** cadastro válido persiste; ausência de campo obrigatório e sigla duplicada são rejeitadas; data de criação é automática; colaborador não pode cadastrar.

### RF-002 — Consultar setores

**Objetivo:** gerente lista setores e consulta um setor por identificador, incluindo sua situação atual.

**Aceite essencial:** consultas administrativas retornam setores ativos e inativos; colaborador não acessa essas operações.

### RF-003 — Atualizar setor

**Objetivo:** gerente atualiza informações de setor existente, mantendo sigla única e data de criação original.

**Aceite essencial:** atualização válida persiste; sigla de outro setor é rejeitada; data de criação permanece inalterada.

### RF-004 — Alterar situação do setor

**Objetivo:** gerente ativa ou inativa setores, sem exclusão física nem perda dos vínculos existentes.

**Regra específica:** setor inativo não pode receber novos colaboradores, inclusive por transferência cadastral.

**Aceite essencial:** ativação e inativação funcionam; vínculos anteriores permanecem; novos vínculos com setor inativo são rejeitados; não há operação de exclusão física.

## 4. Gestão de Colaboradores

### RF-005 — Cadastrar colaborador

**Objetivo:** gerente cadastra pessoa com papel `COLABORADOR`.

| Campo | Obrigatório | Regra |
|---|---|---|
| Nome completo | Sim | Dado cadastral |
| CPF | Sim | Válido e único |
| E-mail corporativo | Sim | Válido e único; usado para convite e login |
| Matrícula funcional | Sim | Única |
| Cargo | Sim | Dado profissional |
| Setor | Sim | Exatamente um setor existente e ativo |
| Data de admissão | Sim | Dado profissional |
| Telefone | Não | Contato |
| Situação | Sim | Ativo ou inativo |

**Regras específicas:** cadastro funcional e ativação da conta são etapas distintas; o `colaborador-service` não armazena credenciais; o cadastro inicia provisionamento no `identidade-service`.

**Aceite essencial:** dados válidos criam colaborador vinculado ao setor; CPF inválido/duplicado, matrícula/e-mail duplicados e setor inexistente/inativo são rejeitados; a criação inicia provisionamento sem criar senha local.

### RF-006 — Consultar colaboradores

**Objetivo:** gerente lista e consulta colaboradores, inclusive inativos; colaborador consulta exclusivamente seus próprios dados por operação dedicada.

**Aceite essencial:** colaborador não lista nem acessa registros de terceiros; a consulta própria usa a identidade autenticada, não um ID escolhido pelo cliente; inexistentes recebem resposta apropriada conforme contrato da API.

### RF-007 — Atualizar colaborador

**Objetivo:** gerente altera dados cadastrais/profissionais permitidos, sem alterar identificador interno ou credenciais.

**Regra específica:** transferência exige setor de destino existente e ativo; alterações de e-mail corporativo seguem a sincronização entre serviços.

**Aceite essencial:** atualização válida persiste; violações de unicidade ou setor inválido são rejeitadas; identificador é preservado; sincronização necessária é iniciada e acompanhada.

### RF-008 — Inativar e reativar colaborador

**Objetivo:** gerente altera a situação cadastral sem exclusão física ou perda de vínculo e histórico.

**Regras específicas:** inativação bloqueia acesso efetivo; reativação cadastral não remove bloqueios de segurança nem dispensa ativação de conta eventualmente pendente.

**Aceite essencial:** registro permanece consultável após inativação; novas autenticações e tokens/sessões anteriores deixam de permitir acesso; falha de integração não é apresentada como bloqueio/reativação concluído.

## 5. Gestão de Gerentes

### RF-009 — Cadastrar gerente

**Objetivo:** gerente cadastra outro gerente; o primeiro gerente é criado por **bootstrap seguro e controlado**, sem endpoint público nem senha administrativa fixa.

**Dados:** os mesmos campos cadastrais do colaborador, exceto que **setor é opcional**. CPF, matrícula e e-mail seguem unicidade global. Se um setor for informado, deve existir e estar ativo.

**Regras específicas:** cadastro inicia provisionamento da conta com papel `GERENTE`, sem armazenar credenciais no `colaborador-service`; bootstrap não pode ser repetido indevidamente nem criar administradores indiscriminadamente.

**Aceite essencial:** gerente cadastra outro gerente com ou sem setor; colaborador não cria gerente; primeiro gerente só pode ser provisionado pelo procedimento autorizado; não existe endpoint público de elevação de privilégio.

### RF-010 — Proteger a continuidade administrativa

**Objetivo:** impedir a perda de administração efetiva do portal.

**Regras específicas:** gerente não ativa nem inativa a si próprio. A organização deve manter pelo menos **um gerente ativo e efetivamente apto a administrar**, considerando cadastro, conta de identidade e bloqueios. Operações concorrentes não podem violar essa garantia. Recuperação excepcional deve ocorrer por procedimento seguro, sem endpoint público de elevação de privilégios.

**Aceite essencial:** autoalteração de situação é rejeitada; inativação do último gerente apto é rejeitada; duas inativações concorrentes não eliminam a administração; conta bloqueada não conta automaticamente como gerente apto.

### RF-011 — Consultar gerentes

**Objetivo:** gerente lista e consulta dados cadastrais/profissionais de gerentes ativos e inativos, respeitando proteção de dados.

**Aceite essencial:** consulta por identificador e listagem funcionam; colaborador não acessa consultas administrativas; dados de autenticação não são expostos; inexistentes recebem resposta apropriada.

### RF-012 — Atualizar gerente

**Objetivo:** gerente atualiza dados cadastrais/profissionais permitidos de outros gerentes **e de si próprio**.

**Regras específicas:** atualização comum não altera papel, situação ativa/inativa, identificador interno ou credenciais. CPF, matrícula e e-mail permanecem únicos; dados sensíveis exigem controles adicionais a definir. Setor é opcional, mas, quando informado ou alterado, deve existir e estar ativo. Mudanças compartilhadas com identidade seguem integração definida. A edição própria identifica o titular pelo contexto de segurança.

**Aceite essencial:** atualização própria e de outros gerentes funciona para campos autorizados; tentativas de alterar papel/situação/ID/credenciais são rejeitadas; duplicidade e setor inválido são rejeitados; bloqueios existentes não são contornados.

### RF-013 — Inativar e reativar gerente

**Objetivo:** gerente ativa ou inativa **outro** gerente, preservando registros e histórico.

**Regras específicas:** aplicam-se integralmente as garantias do RF-010; inativação bloqueia a conta e todas as credenciais anteriores; reativação não remove bloqueios independentes nem dispensa eventual ativação pendente; operações são auditadas com ator, alvo, data e ação.

**Aceite essencial:** autoalteração é rejeitada; último gerente apto é protegido mesmo sob concorrência; gerente inativado perde acesso imediatamente; reativação respeita bloqueios; falha de integração não gera confirmação indevida.

## 6. Gestão de Identidade e Acesso

### RF-014 — Provisionar conta de acesso

**Objetivo:** cadastro de colaborador/gerente inicia evento Kafka para criação de **uma única conta por pessoa**, com papel correspondente e situação **pendente de ativação**.

**Regras específicas:** provisionamento é idempotente, tolerante a falhas temporárias e inicia envio de convite ao e-mail corporativo. Conta pendente não pode autenticar. Primeiro gerente respeita bootstrap seguro.

**Aceite essencial:** evento cria conta correta; repetição não duplica; falha recuperável não perde solicitação; convite é iniciado; nenhuma senha em texto puro é transmitida.

### RF-015 — Ativar conta de acesso

**Objetivo:** titular ativa conta pendente usando convite por e-mail corporativo e define sua senha.

**Regras específicas:** token seguro, temporário, revogável e de uso único; inválidos/expirados/usados são rejeitados; ativação exige cadastro funcional ativo e ausência de bloqueio impeditivo; convite expirado pode ser substituído de forma controlada; ativação não cria sessão automaticamente.

**Aceite essencial:** ativação válida habilita login posterior; token não pode ser reutilizado, inclusive sob concorrência; conta inativa/bloqueada não é ativada; novo convite substitui o expirado sem reutilizá-lo.

### RF-016 — Autenticar usuário

**Objetivo:** autenticar com e-mail corporativo e senha e emitir Access Token de curta duração e Refresh Token renovável.

**Regras específicas:** somente conta ativada, cadastro funcional ativo e ausência de bloqueios permitem login/renovação. Refresh Token é exclusivo para renovação, possui validade, rotação e revogação; tokens inválidos, expirados, revogados ou reutilizados indevidamente são rejeitados. Respostas não revelam existência de contas.

**Aceite essencial:** login válido emite as duas credenciais; login inválido ou conta bloqueada falha; renovação válida rotaciona Refresh Token; token anterior não renova novamente; bloqueio impede uso de credenciais ainda não expiradas.

### RF-017 — Autorizar acesso por papéis

**Objetivo:** proteger recursos conforme papel, identidade do titular e regras específicas da operação.

**Regras específicas:** gerentes executam funções administrativas permitidas, sem dispensar restrições como a proteção do último gerente; colaboradores só consultam os próprios dados. Cada serviço protege seus endpoints e considera revogações vigentes.

**Aceite essencial:** colaborador não executa operação administrativa nem consulta terceiros; gerente autorizado executa operação permitida; ausência de autenticação resulta em 401 e insuficiência de permissão em 403, quando aplicável; manipulação de IDs ou papéis pelo cliente não eleva privilégios.

### RF-018 — Recuperar senha de acesso

**Objetivo:** titular solicita redefinição por e-mail corporativo, usando token seguro, temporário e de uso único.

**Regras específicas:** resposta inicial é genérica, sem confirmar existência de conta. Recuperação só é disponibilizada a contas elegíveis e não ativa contas pendentes nem remove bloqueios. Nova senha segue política de segurança. Sucesso consome token, revoga **todas** as sessões/credenciais anteriores e exige novo login; não cria sessão automaticamente. Operação auditada sem segredos.

**Aceite essencial:** token válido permite redefinição; token inválido/expirado/usado ou concorrência não permite múltiplas redefinições; senha antiga deixa de autenticar; sessões anteriores deixam de funcionar; resposta inicial não enumera contas.

### RF-019 — Encerrar sessão autenticada

**Objetivo:** usuário encerra a sessão atual ou todas as suas sessões.

**Regras específicas:** logout atual revoga Refresh Token e Access Tokens da sessão, preservando outras sessões válidas. Logout global revoga todas as sessões do titular. Repetição da operação não restaura credenciais. Encerramento não altera situação cadastral; falha de revogação não pode ser apresentada como sucesso efetivo.

**Aceite essencial:** logout atual bloqueia acesso/renovação daquela sessão sem afetar outra; logout global bloqueia todas; IDs manipulados não encerram sessões alheias; revogação não confirmada não retorna conclusão indevida.

### RF-020 — Sincronizar dados de identidade

**Objetivo:** propagar por Kafka alterações relevantes do `colaborador-service` ao `identidade-service` sem violar os limites de propriedade dos dados.

**Regras específicas:** sincronizar e-mail corporativo, situação cadastral e eventuais mudanças de papel especificamente autorizadas. Processamento deve ser idempotente, resistente a eventos fora de ordem e recuperável após falhas. Estados de sincronização devem ser rastreáveis. Alterações críticas de acesso exigem confirmação efetiva do bloqueio; reativação não ignora restrições independentes.

**Aceite essencial:** novo e-mail é reconhecido após sincronização; evento duplicado/antigo não corrompe dados; falha preserva pendência recuperável; inativação impede uso de tokens anteriores; serviço não altera diretamente banco de outro serviço.

### RF-021 — Alterar senha de acesso

**Objetivo:** usuário autenticado altera **a própria senha** informando a senha atual e uma nova senha.

**Regras específicas:** exige conta ativa sem bloqueio impeditivo; senha atual correta; nova senha conforme política e diferente da atual. Sucesso revoga todas as sessões/credenciais, exige novo login e não modifica papel nem dados funcionais. Proteção contra abuso e auditoria sem segredos são obrigatórias.

**Aceite essencial:** senha atual incorreta, nova senha inválida ou igual à anterior são rejeitadas; alteração não afeta outra conta; nova senha autentica após novo login; senha antiga e tokens anteriores deixam de funcionar; falha de revogação não é confirmada como sucesso integral.

## 7. Decisões técnicas pendentes — antes da implementação correspondente

Estas definições **não representam novas funcionalidades** e devem ser resolvidas na arquitetura, segurança e contratos de API, conforme cada incremento:

1. Contratos REST: campos editáveis/retornados, paginação administrativa, códigos de erro, validação/normalização de CPF, telefone, matrícula e sigla de setor.
2. Contratos Kafka e confiabilidade: formato e versão dos eventos, publicação confiável (avaliar Transactional Outbox), idempotência, ordenação, retentativas e recuperação de falhas.
3. Consistência de segurança distribuída: bloqueio imediato de tokens, revogação de sessões, confirmação de inativação e proteção concorrente do último gerente apto.
4. Bootstrap e recuperação administrativa: execução autorizada, prevenção de repetição, validação entre serviços e procedimentos operacionais.
5. Política de credenciais: validade de convites e tokens, hash de senha, política de senhas, controles contra abuso, detecção de reutilização de Refresh Token e armazenamento seguro no cliente.
6. E-mail e mudanças sensíveis: mecanismo de entrega, reenvio, controles para alteração de CPF/matrícula/e-mail e consequências sobre sessões existentes.
7. Requisitos não funcionais: disponibilidade, desempenho, observabilidade, retenção/auditoria, proteção de dados e estratégia de testes.

**Nota de escopo:** não foi aprovado um fluxo administrativo para alterar o papel de uma pessoa existente. Qualquer referência a sincronização de papéis é condicional a uma futura especificação, não uma autorização implícita.

## 8. Orientação para desenvolvimento incremental

Implementar por incrementos verticais, preservando os requisitos de segurança e integridade aplicáveis a cada entrega. Não é necessário resolver antecipadamente todos os detalhes dos requisitos futuros; as decisões críticas para o incremento em desenvolvimento devem estar documentadas **antes de seu código**.
