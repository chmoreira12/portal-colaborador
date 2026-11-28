# Portal do Colaborador — Contratos de integração

**Versão-alvo:** 1.0  
**Status:** Contratos definidos para implementação incremental; nenhum endpoint ou consumidor é considerado implementado por este documento.  
**Referências normativas do projeto:** `02-REQUISITOS.md` (RF-001 a RF-021), `03-ARQUITETURA.md`, `04-SEGURANCA.md`.

## 1. Convenções REST

- API pública: `/api/v1`, exclusivamente pelo Gateway. Rotas internas: `/internal/v1`, nunca publicadas pelo Gateway. Endpoints técnicos OAuth2/JWKS seguem a configuração do emissor, fora do versionamento REST de negócio.
- JSON UTF-8; nomes de campos em português, camelCase, salvo campos padronizados por protocolos (JWT, OAuth2, Problem Details, Kafka envelope). Recursos em português, plural e kebab-case. Identificadores UUID representados como strings; CPF, matrícula e telefone são strings, não números.
- Datas civis em ISO 8601 (`YYYY-MM-DD`); instantes em UTC, RFC 3339 (`2026-11-28T15:00:00Z`). `situacao`: `ATIVO` ou `INATIVO`; `papel`: `GERENTE` ou `COLABORADOR`.
- Sucesso retorna o DTO diretamente, sem envelope genérico. Coleções paginadas usam o DTO da seção 3. Respostas 204 não têm corpo. `Location` referencia o recurso criado ou a operação assíncrona. URLs de retorno são relativas à origem pública, sem expor DNS interno.
- Autorização conforme `04-SEGURANCA.md`; Bearer JWT de usuário em `Authorization` para recursos protegidos. Identidade deriva de `sub`/`sid` validados. Não aceitar papel, ator ou identidade autenticada fornecidos em campos arbitrários.
- Rejeitar JSON malformado, campos desconhecidos e campos não editáveis; DTOs explícitos, Bean Validation e validações de domínio no service. Nunca serializar entidades JPA, hashes, tokens de armazenamento ou dados internos de segurança.
- `Content-Type: application/json` nas requisições JSON; `application/problem+json` nos erros. Respostas contendo dados pessoais ou credenciais: `Cache-Control: no-store`. Utilizar limites de tamanho de requisição (64 KiB para JSON de negócio, inicialmente configurável).
- OpenAPI gerado por aplicação com biblioteca compatível com a versão do Spring Boot. Documentar DTOs, exemplos fictícios, parâmetros, segurança, respostas e restrições. Especificações exportadas/versionadas em `docs/contracts/` conforme cada incremento; alterações incompatíveis exigem nova versão de contrato, análise de impacto e ADR. Documentação técnica não exposta publicamente em produção.

## 2. Erros e códigos HTTP

Utilizar RFC 9457 Problem Details, com `ProblemDetail` e tratamento centralizado (`@RestControllerAdvice`, handlers Spring Security e tratamento equivalente no Gateway). Erros de infraestrutura não expõem stack traces, SQL, hosts, nomes de tabelas ou segredos.

```json
{
  "type": "urn:portal-colaborador:problem:validacao",
  "title": "Dados inválidos",
  "status": 400,
  "detail": "Revise os campos informados.",
  "instance": "/api/v1/setores",
  "codigo": "VALIDACAO_INVALIDA",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "erros": [{"campo": "sigla", "codigo": "OBRIGATORIO", "mensagem": "Informe a sigla."}]
}
```

`codigo` é estável e consumível pelo frontend; `erros` é opcional, reservado a validações de campos, sem valores rejeitados. `instance` não contém query string sensível. `traceId` é o identificador de rastreamento quando disponível. OAuth2 mantém seu formato de erro padronizado; não converter erros do endpoint de token OAuth2 em Problem Details.

| HTTP | Aplicação | Códigos de negócio exemplificativos |
|---|---|---|
| 200 | Consulta, atualização ou autenticação concluída | — |
| 201 | Cadastro persistido | — |
| 202 | Solicitação genérica ou operação distribuída aceita e ainda não concluída | — |
| 204 | Operação concluída sem representação | — |
| 400 | JSON, parâmetros, campos, token de finalidade ou regra de entrada inválidos | `VALIDACAO_INVALIDA`, `TOKEN_INVALIDO` |
| 401 | Autenticação/sessão inválida, login/refresh recusado | `AUTENTICACAO_INVALIDA` |
| 403 | Identidade válida sem permissão; reautenticação requerida/inválida; CSRF | `ACESSO_NEGADO`, `REAUTENTICACAO_NECESSARIA` |
| 404 | Recurso não encontrado dentro do domínio autorizado | `RECURSO_NAO_ENCONTRADO` |
| 409 | Conflito de unicidade, estado ou operação concorrente | `SIGLA_DUPLICADA`, `CPF_DUPLICADO`, `EMAIL_DUPLICADO`, `MATRICULA_DUPLICADA`, `SETOR_INATIVO`, `ULTIMO_GERENTE_APTO`, `AUTOALTERACAO_SITUACAO`, `OPERACAO_PENDENTE`, `CHAVE_IDEMPOTENCIA_CONFLITANTE` |
| 412 | `If-Match` não corresponde à versão atual | `VERSAO_DESATUALIZADA` |
| 415 | Tipo de mídia não suportado | `MIDIA_NAO_SUPORTADA` |
| 428 | Pré-condição de versão ausente | `PRECONDICAO_OBRIGATORIA` |
| 429 | Rate limit excedido | `LIMITE_EXCEDIDO` |
| 500 | Erro inesperado | `ERRO_INTERNO` |
| 503 | Dependência indisponível ou segurança não confirmável | `SERVICO_INDISPONIVEL` |

Utilizar `WWW-Authenticate` nas respostas Bearer 401 quando aplicável, `Retry-After` nos limites/indisponibilidades quando houver estimativa confiável. Não revelar existência de contas em login, recuperação, reenvio ou ativação inválida; não diferenciar token expirado, consumido, desconhecido ou inelegibilidade por erro público. Conflitos de unicidade detalhados são exclusivos de operações administrativas autorizadas. Erros de autorização precedem a consulta de recursos de terceiros.

## 3. Paginação e concorrência

Listagens administrativas: `page=0`, `size=20`, máximo 100, `sort=nome,asc` (parâmetros opcionais; `sort` repetível). Rejeitar valores inválidos ou campos de ordenação não permitidos. Ordenação secundária por `id` para desempate. Consultas sem resultados retornam 200 com `items: []`, inclusive páginas além da última.

```json
{
  "items": [],
  "page": 0,
  "size": 20,
  "totalElements": 0,
  "totalPages": 0
}
```

Não expor diretamente `PageImpl` como contrato. Filtros combinados por AND. Consultas não filtradas incluem ativos e inativos. Filtros e ordenação com allowlist; sem busca por CPF, e-mail ou matrícula em query strings na versão inicial.

Utilizar versionamento otimista (`@Version` ou controle equivalente) para cadastros. DTO de detalhe contém `versao` inteira não negativa; GET de detalhe e respostas síncronas de criação/atualização retornam ETag forte opaco representando a versão do DTO (ex.: `"v3"`). `PUT` e `PATCH` cadastrais exigem `If-Match` com o valor retornado pelo GET; ausência 428, desatualização 412. Não aceitar `*` para substituir a versão. Comparar e persistir sob garantia transacional; versão cadastral não se confunde com versão de integração Kafka. DTO com dados de outro agregado não deve ser incluído na representação versionada (usar `setorId`, não nome do setor).

Alterações sensíveis que iniciam saga reservam a transição e impedem alterações conflitantes até reconciliação. Alteração de estado igual ao atual, quando não existe transição pendente, retorna o estado atual sem duplicar eventos ou efeitos, respeitadas as regras de autorização, versão e autoalteração.

## 4. Normalização e DTOs cadastrais

Valores iniciais abaixo são limites técnicos escolhidos para a versão 1, não novas regras de negócio. Aplicar validação após normalização de entrada e unicidade também no PostgreSQL, tratando concorrência/violações de constraints. `@NotBlank`, `@Size`, `@Email`, validador CPF e regras de domínio conforme o campo. Os limites de senha seguem exclusivamente `04-SEGURANCA.md`.

| Campo | Tipo e normalização |
|---|---|
| `nome` | String obrigatória, trim, 1–150 caracteres, preservar acentos e capitalização. Para pessoa, representa nome completo. |
| `sigla` | String obrigatória, trim, uppercase Locale.ROOT, 1–20 caracteres ASCII alfanuméricos, hífen ou underscore; unicidade normalizada. |
| `descricao` | String opcional, trim, até 500 caracteres; vazio normaliza para null. |
| `cpf` | String obrigatória, aceitar 11 dígitos ou máscara `000.000.000-00`, persistir 11 dígitos; validar dígitos verificadores e rejeitar sequências repetidas. |
| `emailCorporativo` | String obrigatória, trim e lowercase Locale.ROOT, até 254 caracteres, validar formato. Domínios corporativos permitidos por configuração explícita por ambiente; não inferir domínio a partir de exemplo. Unicidade da forma normalizada. |
| `matricula` | String obrigatória, trim, uppercase Locale.ROOT, 1–40 caracteres ASCII alfanuméricos, hífen ou underscore; preservar zeros à esquerda. |
| `cargo` | String obrigatória, trim, 1–100 caracteres. |
| `setorId` | UUID; obrigatório para colaborador, opcional/nulo para gerente. Validar existência e atividade na criação ou mudança de vínculo. Preservar vínculos históricos com setor inativo em atualizações que não alterem o setor. |
| `dataAdmissao` | LocalDate obrigatória, data válida, sem restrição adicional de data futura não prevista nos requisitos. |
| `telefone` | String opcional; aceitar telefone brasileiro com DDD e 10/11 dígitos ou formato internacional com `+` e até 15 dígitos; validar com libphonenumber ou equivalente, normalizar para E.164; vazio para null. |
| `situacao` | Enum obrigatória na criação; alteração somente por operação dedicada. |

### 4.1 Setor

- `SetorCreateRequest`: `nome`, `sigla`, `descricao?`, `situacao`.
- `SetorUpdateRequest`: `nome`, `sigla`, `descricao?`. PUT substitui os campos editáveis; ausência de opcional implica null.
- `SituacaoRequest`: `situacao` exclusivamente.
- `SetorResponse`: `id`, `nome`, `sigla`, `descricao`, `situacao`, `criadoEm`, `versao`.
- `SetorResumoResponse`: `id`, `nome`, `sigla`, `situacao`.

### 4.2 Pessoas

- `PessoaCreateRequest` (DTOs específicos para colaborador e gerente): `nome`, `cpf`, `emailCorporativo`, `matricula`, `cargo`, `setorId`, `dataAdmissao`, `telefone?`, `situacao`. Papel definido pela rota; não aceitar `papel` no corpo.
- `PessoaUpdateRequest` (DTOs específicos por papel): mesmos campos editáveis da criação, exceto `situacao`. PUT exige campos obrigatórios; opcionais omitidos ficam null. Para gerente, `setorId` pode ser null. Alterações de CPF, matrícula e e-mail exigem prova de reautenticação conforme seção 7.
- `PessoaResponse`: `id` (corresponde a `pessoaId` da identidade e eventos), `nome`, `cpf`, `emailCorporativo`, `matricula`, `cargo`, `setorId`, `dataAdmissao`, `telefone`, `situacao`, `papel`, `criadoEm`, `versao`.
- `PessoaResumoResponse`: `id`, `nome`, `emailCorporativo`, `matricula`, `cargo`, `setorId`, `situacao`, `papel`. Não inclui CPF ou telefone.
- `MeusDadosResponse`: mesmos campos funcionais de `PessoaResponse`, exclusivamente do titular.

Campos opcionais das respostas são retornados como null quando ausentes. Não retornar conta, senha, hash, sessões ou bloqueios de identidade em DTO cadastral. A criação de pessoa confirma somente o cadastro e a gravação de Outbox, não criação/ativação imediata da conta; consultar rastreabilidade operacional da seção 8. Não há exclusão física nem alteração de papel nesta versão.

## 5. Endpoints públicos — colaborador-service

Prefixo `/api/v1`. Todas as operações abaixo, exceto consulta própria, exigem GERENTE autenticado e apto. `PUT` e `PATCH` usam a regra `If-Match` da seção 3. `POST` de cadastro (setor ou pessoa) e alterações de pessoa usam `Idempotency-Key` conforme seção 8.

| Método | Rota | Entrada | Sucesso | RF |
|---|---|---|---|---|
| POST | `/setores` | SetorCreateRequest | 201 SetorResponse | 001 |
| GET | `/setores` | Paginação/filtros | 200 página de SetorResumoResponse | 002 |
| GET | `/setores/{id}` | UUID | 200 SetorResponse | 002 |
| PUT | `/setores/{id}` | SetorUpdateRequest | 200 SetorResponse | 003 |
| PATCH | `/setores/{id}/situacao` | SituacaoRequest | 200 SetorResponse | 004 |
| POST | `/colaboradores` | PessoaCreateRequest | 201 PessoaResponse | 005, 014 |
| GET | `/colaboradores` | Paginação/filtros | 200 página de PessoaResumoResponse | 006 |
| GET | `/colaboradores/{id}` | UUID | 200 PessoaResponse | 006 |
| PUT | `/colaboradores/{id}` | PessoaUpdateRequest | 200 PessoaResponse ou 202 OperacaoResponse | 007, 020 |
| PATCH | `/colaboradores/{id}/situacao` | SituacaoRequest | 200 PessoaResponse ou 202 OperacaoResponse | 008, 020 |
| POST | `/gerentes` | PessoaCreateRequest | 201 PessoaResponse | 009, 014 |
| GET | `/gerentes` | Paginação/filtros | 200 página de PessoaResumoResponse | 011 |
| GET | `/gerentes/{id}` | UUID | 200 PessoaResponse | 011 |
| PUT | `/gerentes/{id}` | PessoaUpdateRequest | 200 PessoaResponse ou 202 OperacaoResponse | 012, 020 |
| PUT | `/gerentes/me` | PessoaUpdateRequest | 200 PessoaResponse ou 202 OperacaoResponse | 012, 020 |
| PATCH | `/gerentes/{id}/situacao` | SituacaoRequest | 200 PessoaResponse ou 202 OperacaoResponse | 010, 013, 020 |
| GET | `/pessoas/me` | Identidade autenticada | 200 MeusDadosResponse | 006, 011, 017 |
| GET | `/operacoes/{id}` | UUID da saga | 200 OperacaoResponse | 008, 010, 013, 020 |

Para PUT/PATCH de pessoa, 200 retorna `PessoaResponse` quando a alteração termina; 202 retorna somente `OperacaoResponse` e `Location`. A consulta própria aceita ambos os papéis e não recebe identificador do titular. Gerente pode atualizar os próprios dados por `/gerentes/me`; a rota por ID aplica as mesmas regras de identidade, reautenticação e autorização. Nenhuma das rotas de situação permite autoativação/autoinativação de gerente, mesmo sem mudança efetiva. Recursos cujo papel não corresponde à rota são tratados como não encontrados para gerente autorizado.

Filtros de setores: `nome` (contém, sem distinção de maiúsculas/minúsculas), `sigla` (igualdade normalizada), `situacao`. Ordenação permitida: `nome`, `sigla`, `situacao`, `criadoEm`, `id`.

Filtros de pessoas: `nome` (contém, sem distinção de maiúsculas/minúsculas), `situacao`, `setorId`. Ordenação: `nome`, `matricula`, `cargo`, `dataAdmissao`, `situacao`, `criadoEm`, `id`. Limitar filtros textuais a 150 caracteres e usar consultas parametrizadas. Não permitir filtros que exponham credenciais ou dados de segurança.

Exemplo de cadastro (dados fictícios):

```http
POST /api/v1/setores
Authorization: Bearer <access-token>
Idempotency-Key: 1ac5fb38-2426-4a86-89be-c6541b0176e2
Content-Type: application/json

{"nome":"Tecnologia da Informação","sigla":"TI","descricao":null,"situacao":"ATIVO"}
```

```http
HTTP/1.1 201 Created
Location: /api/v1/setores/76b986de-826f-4290-a7f2-4b9d6718e530
ETag: "v0"
Content-Type: application/json

{"id":"76b986de-826f-4290-a7f2-4b9d6718e530","nome":"Tecnologia da Informação","sigla":"TI","descricao":null,"situacao":"ATIVO","criadoEm":"2026-11-28T15:00:00Z","versao":0}
```

## 6. Endpoints públicos — identidade-service

Prefixo `/api/v1/auth`. Login, recuperação e convite são operações públicas sujeitas aos controles de abuso definidos em `04-SEGURANCA.md`. O refresh utiliza cookie; logout atual/global e alteração de senha exigem identidade autenticada. Não existe endpoint público de bootstrap ou criação direta de conta.

| Método | Rota | Entrada | Sucesso | RF |
|---|---|---|---|---|
| GET | `/csrf` | Sem credenciais obrigatórias | 200 CsrfResponse | 016, 019 |
| POST | `/login` | LoginRequest | 200 TokenResponse + cookie refresh | 016 |
| POST | `/refresh` | Cookie refresh, cabeçalho CSRF | 200 TokenResponse + cookie rotacionado | 016 |
| POST | `/logout` | Bearer, CSRF | 204, expirar cookie | 019 |
| POST | `/logout-all` | Bearer, CSRF | 204, expirar cookie | 019 |
| POST | `/ativacao` | AtivacaoRequest | 204 | 015 |
| POST | `/convites/reenvio` | EmailRequest | 202, resposta genérica | 015 |
| POST | `/recuperacao-senha` | EmailRequest | 202, resposta genérica | 018 |
| POST | `/recuperacao-senha/confirmacao` | RedefinicaoSenhaRequest | 204, expirar cookie | 018 |
| POST | `/alteracao-senha` | AlteracaoSenhaRequest, Bearer, CSRF | 204, expirar cookie | 021 |
| POST | `/reautenticacao` | ReautenticacaoRequest, Bearer | 200 ProvaReautenticacaoResponse | 007, 012 |

DTOs:

- `CsrfResponse`: `headerName`, `token`; utilizar `X-XSRF-TOKEN` como cabeçalho e `XSRF-TOKEN` como cookie de CSRF legível pelo Angular, não credencial. GET emite/carrega token conforme configuração do Spring Security. Cookie CSRF acessível na raiz da SPA; o token de requisição deve corresponder ao mecanismo CSRF configurado. Renovar token após login/logout conforme framework.
- `LoginRequest`: `emailCorporativo`, `senha`.
- `TokenResponse`: `accessToken`, `tokenType` (`Bearer`), `expiresIn` (segundos efetivamente restantes). Nunca retornar refresh token em JSON.
- `EmailRequest`: `emailCorporativo`.
- `AtivacaoRequest` e `RedefinicaoSenhaRequest`: `token`, `novaSenha`.
- `AlteracaoSenhaRequest`: `senhaAtual`, `novaSenha`.
- `ReautenticacaoRequest` e `ProvaReautenticacaoResponse`: seção 7.

Exemplo de resposta de login (token omitido):

```http
HTTP/1.1 200 OK
Cache-Control: no-store
Set-Cookie: refresh_token=<opaque>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=604800
Content-Type: application/json

{"accessToken":"<jwt>","tokenType":"Bearer","expiresIn":300}
```

O cookie não tem `Domain`, possui prazo efetivo limitado à sessão e é sobrescrito a cada renovação. Em desenvolvimento HTTP local, `Secure` pode ser desabilitado exclusivamente no profile local. Expirar cookie com mesmo nome, path e atributos quando a sessão for encerrada ou considerada inválida. Em falha transitória sem confirmação de revogação, não apresentar logout concluído. O logout usa `sid` do Bearer, e não um ID de sessão informado no corpo; cookie apresentado, se houver, não autoriza revogação de sessão alheia. Cliente com token expirado deve renovar antes de solicitar logout; se já definitivamente revogado, 401 indica que não existe acesso válido, sem afirmar que outras sessões foram revogadas. Nunca retornar sucesso de logout global sem commit da revogação global.

Login, refresh, logout e demais rotas autenticadas por cookie exigem proteção CSRF e verificação de origem, conforme segurança. Operações exclusivamente Bearer ou token de finalidade não usam cookie de autenticação; somente dispensar CSRF se a configuração garantir essa propriedade. Não utilizar header Authorization para transportar refresh token. Login e refresh são operações sem retentativa automática. Em caso de refresh de resultado incerto, novo login pode ser necessário. Coordenar renovação entre abas conforme segurança.

Recuperação e reenvio retornam sempre 202 com a mesma representação `{"mensagem":"Se a conta estiver elegível, enviaremos instruções ao e-mail informado."}` para solicitações válidas, independentemente da existência da conta. Limites de abuso e indisponibilidade podem retornar 429/503 sem revelar existência. Senha atual incorreta em alteração de senha: 400 `CREDENCIAL_ATUAL_INVALIDA`; nova senha fora da política ou igual à anterior: 400 `VALIDACAO_INVALIDA`. Erros de autenticação pública e token de finalidade seguem seção 2.

Ativação/recuperação não iniciam sessão. Enviar tokens exclusivamente no corpo POST, nunca em query strings. Não registrar corpos desses endpoints. Alteração e recuperação de senha confirmam revogação global antes de 204.

## 7. Reautenticação para alterações sensíveis

Para alterações de CPF, matrícula ou e-mail corporativo, gerente obtém prova de reautenticação no `identidade-service` antes do PUT cadastral. `ReautenticacaoRequest`: `senhaAtual`, `operacao` (`ALTERAR_DADOS_SENSIVEIS`), `pessoaAlvoId` (UUID), `campos` (conjunto não vazio de `cpf`, `matricula`, `emailCorporativo`). A identidade valida senha, sessão, papel e situação, sem aceitar credenciais de outra pessoa. `ProvaReautenticacaoResponse`: `prova` (token opaco aleatório de 256 bits), `expiresIn` (inicialmente 300 segundos, configurável).

Transmitir prova no cabeçalho `X-Reauthentication-Proof` do PUT, junto ao Bearer e `Idempotency-Key`. Persistir apenas hash da prova, vinculado a ator, sessão, alvo, finalidade e campos permitidos. O colaborador-service determina os campos realmente alterados, não confia em declaração do cliente e solicita validação/consumo interno no identidade-service. Prova autoriza apenas os campos solicitados e é vinculada a uma única operação persistente; nova operação não pode reutilizá-la. Repetição idempotente da mesma operação pode recuperar resultado anterior sem novo consumo. Não repassar senha atual ao colaborador-service; prova e cabeçalhos sensíveis nunca aparecem em logs.

Se nenhum campo sensível mudar, não exigir prova. CPF/matrícula mantêm identificador interno e são auditados; não revogam sessões por si sós. Mudança de e-mail exige saga, revoga todas as sessões do alvo e invalida tokens anteriores conforme segurança. Mudança própria de e-mail poderá exigir novo login com o novo endereço após conclusão; um 202 não é confirmação de mudança. A identidade deve rejeitar transições que deixariam a organização sem gerente apto, inclusive bloqueio temporário de alteração de e-mail do último gerente apto.

## 8. Idempotência, operações distribuídas e rastreabilidade

### 8.1. Repetição de comandos públicos

Exigir `Idempotency-Key` (UUID) em POST de cadastro de setor/pessoa e PUT/PATCH de pessoa. O frontend gera uma chave para uma ação lógica e a reutiliza apenas na repetição daquela mesma ação. Persistir registro de idempotência no serviço de domínio, escopado por ator, método e recurso, com hash da entrada normalizada e pré-condição de versão. Não incluir tokens/segredos em hashes de entrada nem armazenar provas brutas. Normalizar rotas equivalentes de edição própria e por ID para o mesmo recurso lógico.

Verificar autorização atual antes de retornar qualquer resultado existente. Chave e entrada equivalentes retornam o resultado durável original ou estado atual da mesma operação; entrada divergente retorna 409. Repetição exata precede nova avaliação de ETag e novo consumo de prova, sem dispensar autorização. Persistir resultado de mutações locais e registro idempotente na mesma transação. Chaves de operações pendentes não expiram enquanto houver trabalho não concluído; reter resultados terminais inicialmente por 7 dias após conclusão. Não prometer deduplicação pública fora dessa janela; clientes não reutilizam chaves expiradas. Constraints de unicidade, versões e idempotência interna permanecem obrigatórias. Nunca persistir tokens de login/refresh em registros de idempotência.

### 8.2. Operações de saga

`OperacaoResponse`:

```json
{
  "id": "5f8921e1-6a77-4d47-bae0-9e02826120f1",
  "tipo": "INATIVAR_PESSOA",
  "pessoaId": "c4844900-5d53-472e-9867-cfe94866815c",
  "estado": "PENDENTE",
  "criadoEm": "2026-11-28T15:00:00Z",
  "atualizadoEm": "2026-11-28T15:00:00Z",
  "codigoFalha": null
}
```

Tipos: `INATIVAR_PESSOA`, `REATIVAR_PESSOA`, `ALTERAR_EMAIL`. Estados: `PENDENTE`, `EM_PROCESSAMENTO`, `CONCLUIDA`, `REJEITADA`, `INTERVENCAO_NECESSARIA`. Estado público não expõe etapas internas, CPF, e-mail, payload original nem segredos. `codigoFalha` opcional, estável e sanitizado. `REJEITADA` somente quando comprovadamente não há efeito parcial inseguro; `INTERVENCAO_NECESSARIA` preserva restrições e reservas e não indica conclusão.

PUT de pessoa que altera e-mail trata toda a atualização solicitada como uma unidade lógica: não aplicar parcialmente os demais campos antes da confirmação do bloqueio inicial; validar e reservar previamente os dados necessários. Se a operação termina dentro do orçamento HTTP, retornar 200 com representação atualizada. Se a intenção foi persistida e a conclusão ainda não é conhecida, retornar 202 com `Location: /api/v1/operacoes/{id}`. Em falha anterior à persistência da intenção, retornar erro apropriado. Nunca apresentar 200 antes da confirmação de segurança. Rejeições determinísticas conhecidas antes de aceitar a operação retornam 409/403 conforme regra. Após aceite, representar rejeições no estado consultável da operação.

GET da operação é permitido a gerentes atualmente autorizados; não permite execução ou desbloqueio. O frontend consulta a URL com polling moderado (inicialmente a cada 2 segundos, backoff até 10 segundos; parar ao sair da tela, perder autorização ou alcançar estado terminal). Em `CONCLUIDA`, obter novo GET do recurso para recuperar versão atual; em intervenção, informar pendência sem tentar desbloquear. Após revogação própria, o usuário pode precisar autenticar novamente; não relaxar autorização da consulta para permitir acompanhamento.

### 8.3. Provisionamento e sincronização

Cadastro retorna 201 quando pessoa, idempotência e Outbox estão persistidos; isso não confirma conta provisionada ou convite entregue. Registrar correlacionáveis `pessoaId`, `eventId`, `aggregateVersion`, `operationId` (quando houver), estados do Outbox, processamento no consumidor e entrega de e-mail. Distinguir publicação Kafka, aplicação na identidade e envio de convite; publicação não prova consumo. Expor rastreabilidade à operação autorizada por métricas, logs sanitizados e consultas/runbooks internos, sem adicionar painel administrativo de identidade fora do escopo. A primeira conta nasce pendente; convite é enviado somente quando elegível. Reativação de conta ainda pendente permite iniciar convite elegível, sem ativar automaticamente.

## 9. Contratos HTTP internos — identidade-service

Prefixo `/internal/v1`. Chamadas do colaborador-service por HTTP Service Clients (`@HttpExchange` + RestClient), autenticação OAuth2 Client Credentials, TLS em produção e scopes exclusivos. Gateway não roteia essas rotas. Tokens de serviço não são Bearer de usuário. Validar `aud`, identidade do cliente e scope, conforme segurança. O emissor utiliza endpoint de token OAuth2 padronizado (inicialmente `/oauth2/token`) e JWKS (inicialmente `/oauth2/jwks`); descoberta/configuração interna, sem publicar emissão de token de serviço no Gateway.

| Método | Rota | Scope | Resultado |
|---|---|---|---|
| POST | `/sessoes/validacao` | `sessoes:validar` | 200 ValidacaoSessaoResponse |
| POST | `/reautenticacoes/consumo` | `reautenticacoes:consumir` | 200 ResultadoComandoResponse |
| PUT | `/pessoas/{pessoaId}/operacoes/{operationId}/preparacao` | `identidade:sincronizar` | 200 ResultadoComandoResponse |
| PUT | `/pessoas/{pessoaId}/operacoes/{operationId}/conclusao` | `identidade:sincronizar` | 200 ResultadoComandoResponse |
| GET | `/pessoas/{pessoaId}/operacoes/{operationId}` | `identidade:sincronizar` | 200 EstadoComandoResponse |

### 9.1. Validação de sessão

`ValidacaoSessaoRequest`: `pessoaId`, `sid`, `papel`, `jti` (obtidos do JWT localmente validado). `ValidacaoSessaoResponse`: `valida` (boolean). Não retornar senha, hash, e-mail ou detalhes desnecessários de bloqueio. Consultar PostgreSQL primário, confirmar associação, papel atual e prazos. Resposta negativa leva a 401 na API pública; indisponibilidade, resposta malformada ou timeout leva a 503, sem cache positivo. 401/403 internos do cliente de serviço indicam falha de integração, não credenciais inválidas do usuário. O identidade-service faz validação local em suas próprias operações.

### 9.2. Contexto do ator e consumo de prova

Na primeira autorização de comando crítico, encaminhar JWT original do ator em cabeçalho interno dedicado `X-Actor-Token`, além de token de serviço em Authorization. O destinatário valida independentemente assinatura, claims, audiência, sessão e papel do ator. Nunca aceitar identidade de ator apenas por cabeçalhos de ID. `ConsumoProvaRequest`: `operationId`, `pessoaAlvoId`, `campos`, `prova`. Validar vinculação ao ator/sessão e consumir sob transação, persistindo resultado idempotente. Para continuidade assíncrona após aceite durável, não persistir JWT do ator; registrar ator validado, operação e escopo autorizado. Retomadas do mesmo comando usam identidade do serviço, operação previamente autorizada e vínculo de parâmetros imutáveis, sem ampliar privilégios. Se ainda não houve aceite autorizado e o token do ator expirou, não iniciar alteração de segurança automaticamente; reconciliar e exigir nova autorização quando necessário. Rejeição sem efeito parcial libera reservas com auditoria. Caso haja efeito parcial, preservar bloqueio até recuperação.

### 9.3. Preparação e conclusão

`PreparacaoOperacaoRequest`: `tipo`, `aggregateVersionEsperada`, `aggregateVersionDestino`, `novoEmailCorporativo` (somente para ALTERAR_EMAIL), `camposSensiveis` (quando aplicável). `operationId` e `pessoaId` vêm da rota. Versão destino é reservada no cadastro e vinculada ao identificador da operação. Não incluir CPF/matrícula no payload; prova já autoriza os campos correspondentes. Comandos que alteram e-mail exigem consumo confirmado da prova associada à mesma operação.

- INATIVAR_PESSOA: preparação aplica restrição cadastral, revoga sessões/tokens incompatíveis, registra fence de versão e verifica continuidade administrativa. Se a conta ainda não existe, persistir restrição pré-provisionamento.
- REATIVAR_PESSOA: preparação registra intenção/fence sem retirar bloqueios. A conclusão ocorre apenas após commit cadastral ativo.
- ALTERAR_EMAIL: preparação reserva e-mail, registra bloqueio temporário, revoga sessões e tokens, valida unicidade e continuidade administrativa. Não habilita novo login.

`ConclusaoOperacaoRequest`: `tipo`, `aggregateVersion` (versão destino confirmada), `situacao`, `emailCorporativo` (quando necessário). Enviar somente depois de commit cadastral e Outbox duráveis. Identidade verifica preparação, operação, versão e snapshot esperado; finaliza exclusivamente a restrição vinculada à operação. Para inativação mantém bloqueio; reativação remove somente restrição cadastral correspondente; e-mail aplica novo endereço, remove somente bloqueio temporário e permite convite se elegível. Não reativar sessões antigas. Eventos Kafka não substituem comando de conclusão.

`ResultadoComandoResponse`: `operationId`, `estado` (`APLICADO`), `aggregateVersionDestino`. `EstadoComandoResponse`: `operationId`, `tipo`, `preparacao` e `conclusao` (`NAO_APLICADA`, `APLICADA`, `REJEITADA`), `aggregateVersionDestino`, `codigoFalha` opcional. Persistir resultado de cada etapa na mesma transação dos efeitos; verificar parâmetros idempotentes. Se não houver registro da operação, GET retorna 404 (não deduzir que não houve efeito em caso de indisponibilidade). Divergência de parâmetros, versão antiga ou transição incompatível: 409. Rejeições de negócio são persistidas sem efeitos parciais. Sucesso de comando não pode ser retornado antes de commit.

Chamada cuja resposta se perdeu: consultar estado ou repetir exatamente o mesmo comando idempotente; nunca gerar novo `operationId`. Proteger ordens de lock e último gerente apto na identidade. Fences e resultados internos devem sobreviver ao horizonte de retentativa/replay; limpeza não permite aplicação de comando antigo. Recuperação excepcional somente pelo procedimento operacional autorizado, nunca por endpoint público.

## 10. Contrato Kafka v1

### 10.1. Tópico, produtor e consumidor

| Propriedade | Definição |
|---|---|
| Tópico lógico | `portal.pessoas.cadastro.v1` |
| Produtor | colaborador-service (Polling Publisher do Outbox) |
| Consumidor | identidade-service |
| Consumer group | `identidade-service-pessoas-v1` (isolado por ambiente) |
| Chave | UUID `pessoaId` em string UTF-8 |
| Valor | JSON UTF-8, schema v1 |
| DLT | `portal.pessoas.cadastro.v1.dlt` |
| Partições iniciais | 3 (local e produção-alvo; validar dimensionamento antes do deploy) |
| Replicação | 1 local; 3 em produção com pelo menos 3 brokers, min.insync.replicas=2 |
| Retenção inicial | 7 dias; DLT 30 dias, com revisão operacional/LGPD |

Isolar ambientes por clusters ou namespaces de tópicos e ACLs. Não enviar classe Java serializada nem depender de type headers de classes compartilhadas. Versionar JSON Schema em `docs/contracts/kafka/` antes da implementação do produtor. Propagar W3C trace context em headers Kafka; não propagar Bearer, cookies ou prova de reautenticação. Desabilitar criação automática de tópicos em produção. Produtor com `acks=all`, idempotência Kafka habilitada; isso não elimina duplicidade Outbox após falha entre ACK e commit local. Compressão e limites configuráveis após validação.

### 10.2. Envelope e eventos

```json
{
  "eventId": "8dbc6245-10d2-4b7d-ad0c-ad96578c1d82",
  "eventType": "PESSOA_CADASTRADA",
  "schemaVersion": 1,
  "occurredAt": "2026-11-28T15:00:00Z",
  "producer": "colaborador-service",
  "pessoaId": "c4844900-5d53-472e-9867-cfe94866815c",
  "aggregateVersion": 1,
  "operationId": null,
  "data": {
    "papel": "COLABORADOR",
    "emailCorporativo": "colaborador@example.invalid",
    "situacao": "ATIVO"
  }
}
```

`eventId` imutável criado na transação Outbox; não regenerar em retentativas. `aggregateVersion` inteiro positivo monotônico por pessoa, inicia em 1, incrementa somente quando há mudança relevante à identidade. `operationId` obrigatório em eventos de e-mail e situação gerados por saga; null no cadastro inicial. O snapshot inclui apenas papel, e-mail corporativo e situação, nunca CPF, matrícula, senha, convite, token ou detalhes de bloqueio. `occurredAt` corresponde à alteração persistida, não ao momento de publicação.

| eventType | Quando emitir | Efeito esperado |
|---|---|---|
| `PESSOA_CADASTRADA` | Criação de colaborador/gerente, incluindo bootstrap | Criar conta única pendente, aplicar restrições pré-provisionamento, iniciar convite se elegível |
| `PESSOA_EMAIL_ALTERADO` | Commit cadastral de saga de e-mail | Reconciliar snapshot e operação; não liberar bloqueio/novo login por evento isolado |
| `PESSOA_SITUACAO_ALTERADA` | Commit cadastral de saga de situação | Reconciliar snapshot e operação; nunca remover restrição por evento isolado |

Alterações puramente cadastrais de nome, telefone, CPF, matrícula, cargo, setor ou data de admissão não geram evento de identidade. Papel é imutável nesta versão; divergência de papel em evento é falha de contrato. Não criar tópico por tipo de evento, pois a ordem relativa por pessoa é necessária.

### 10.3. Outbox e consumo

Persistir alteração e evento na mesma transação. Polling inicial de 1 segundo, lote inicial de 100, configuráveis. Usar claim transacional/lease recuperável com exclusão mútua por agregado; não publicar versão posterior enquanto anterior estiver pendente ou com confirmação incerta. Não manter transação de negócio aberta durante envio Kafka. Marcar publicado só após ACK. Retentativa exponencial com jitter, limite de atraso de 60 segundos e alerta de pendência prolongada; não descartar evento de negócio por exceder tentativas. Reter publicados inicialmente 7 dias, sem excluir pendentes. A retenção efetiva deve considerar recuperação e LGPD.

Consumir registros individualmente; desabilitar auto-commit, confirmar offset somente após commit da transação local ou encaminhamento durável para tratamento de falha. Usar tabela de inbox/processados com constraint única por `eventId` e versão aplicada por pessoa. Registrar inbox e efeitos na mesma transação. Persistir conta única por `pessoaId` e e-mail normalizado único, separando snapshot cadastral de estado de segurança. Duplicata não repete convite, conta ou revogação. Versão inferior/igual não sobrescreve estado atual; payload divergente para mesma versão/eventId é incidente de integridade, não atualização válida.

Snapshot posterior pode permitir salto de versão apenas quando estado, operação e fences comprovam que nenhum efeito de segurança obrigatório foi perdido; caso contrário, persistir pendência e reconciliar. Se evento chega antes da conclusão síncrona, registrar snapshot observado, sem liberar bloqueio, trocar e-mail de login ou aumentar elegibilidade. Comando de conclusão pode chegar antes do evento; aplicar apenas com preparação válida e confirmação de commit cadastral, registrar versão e reconciliar evento posterior idempotentemente. Não confundir última versão de evento observada com versão de transição de segurança efetivada.

### 10.4. Retry, DLT e isolamento de agregado

Utilizar Spring Kafka com retries bloqueantes limitados por partição (3 retentativas após a tentativa inicial, backoff inicial de 1 segundo, máximo inicial de 5 segundos). Falhas transitórias: banco indisponível, timeout, erro recuperável. Falhas permanentes: schema inválido, papel divergente, inconsistência de contrato. Não utilizar retry topics não bloqueantes que permitam ultrapassagem sem controle de agregado.

Após esgotamento ou falha permanente, encaminhar ao DLT com metadados sanitizados de origem (tópico, partição, offset, eventId, pessoaId se recuperável, código de falha). Persistir quarentena por pessoa e eventos subsequentes pendentes em armazenamento durável antes de liberar o offset original; não aplicar versões posteriores de agregado em quarentena até reconciliação. Usar publicação DLT confirmada e estado persistido recuperável; tolerar duplicatas em falha entre persistência, publicação e commit do offset. Se persistência da quarentena ou publicação durável necessária falhar, não confirmar offset. Se não for possível identificar o agregado pela chave, suspender a partição afetada até investigação. Não bloquear agregados não afetados além do necessário para garantir ordenação.

Reprocessamento operacional autorizado, auditado, sem regenerar eventId, com validação de schema, versão e operação; retomar registros pendentes em ordem ou usar snapshot autoritativo comprovadamente suficiente. Não remover quarentena nem desbloquear conta apenas pela existência de evento posterior. Reter inbox/fences pelo horizonte de replay e recuperação, inicialmente ao menos 90 dias para inbox após processamento; versões e fences de segurança persistem enquanto necessários para impedir replay antigo. Retenção e descarte de payloads pessoais revisados antes de produção.

## 11. Compatibilidade e verificação de contratos

- Alterações aditivas de resposta/evento podem permanecer na v1 quando consumidores toleram campos desconhecidos e o significado dos campos existentes é preservado. Requisições REST continuam estritas; consumidores de resposta e de eventos toleram campos novos opcionais. Campos obrigatórios e semântica não mudam silenciosamente. Mudanças incompatíveis usam `/api/v2`, `/internal/v2` ou tópico `.v2` e plano de migração; não publicar em duas versões sem estratégia de deduplicação.
- Testar contratos HTTP com MockMvc/WebTestClient e WireMock, e integração real dos eventos com Testcontainers. Validar serialização, schemas, autorização, códigos, paginação, pré-condições, idempotência, status de saga, ordering, duplicação, retries e DLT. Incluir casos de corrida entre alteração cadastral e provisionamento, reautenticação e revogação, e último gerente apto.
- Manter testes que cruzam os serviços para fluxo de cadastro, convite, login, atualização sensível, inativação, reativação, recuperação e logout. Não introduzir mocks no caminho produtivo para simular dependências essenciais.
- As versões de dependências, especificações OpenAPI exportadas e JSON Schema concretos serão validadas e versionadas antes do código de cada incremento. Este documento define o contrato funcional/técnico; não afirma que os contratos executáveis já existem.

## 12. Rastreabilidade de requisitos

| Requisito | Seção deste documento |
|---|---|
| RF-001 a RF-004 | 3–5: setor, CRUD lógico, validação de vínculos |
| RF-005 a RF-008 | 3–5, 7–10: colaborador, consulta própria, alteração, integração |
| RF-009 a RF-013 | 3–5, 7–10: gerente, bootstrap por operação externa à API, continuidade administrativa |
| RF-014 | 8.3, 10: provisionamento, Outbox e convite |
| RF-015 | 6, 10: ativação e reenvio |
| RF-016 | 6, 9: login, refresh e sessão |
| RF-017 | 1–2, 5–7, 9: autorização por papel e titularidade |
| RF-018 | 6: recuperação de senha |
| RF-019 | 6: logout atual e global |
| RF-020 | 7–10: sincronização e consistência distribuída |
| RF-021 | 6: alteração de senha |

Decisões complementares de limites, versionamento, idempotência, HTTP e Kafka são escolhas técnicas dentro do escopo aprovado. Parâmetros iniciais de capacidade, retenção e timeout devem ser validados em `06-TESTES.md` e `09-OPERACAO.md`; custos AWS dependem de autorização de provisionamento. A documentação de segurança prevalece sobre qualquer leitura de contrato que permita acesso indevido ou revogação não confirmada.
