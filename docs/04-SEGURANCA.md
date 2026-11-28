# Portal do Colaborador — Segurança

**Versão-alvo:** 1.0  
**Status:** Diretrizes aprovadas e decisões técnicas consolidadas para implementação incremental.

## 1. Princípios e responsabilidades

- O `identidade-service` controla contas, credenciais, sessões, bloqueios, emissão de tokens e autorização das operações de identidade.
- O `colaborador-service` controla dados cadastrais e situação funcional, autoriza operações do seu domínio e coordena alterações cadastrais sensíveis.
- O `api-gateway` realiza roteamento, validação inicial de JWT, rate limiting e controles de entrada. Cada serviço de destino valida autenticação e autorização independentemente do Gateway.
- PostgreSQL é a fonte de verdade dos estados de segurança. Redis não é fonte de verdade de permissões, contas ou sessões.
- Negar acesso por padrão. Não considerar dados de identidade fornecidos arbitrariamente pelo cliente como confiáveis.
- Separar estado cadastral, ativação da conta e bloqueios de segurança independentes.

## 2. Autenticação de usuários

Utilizar Spring Security, autenticação por e-mail corporativo normalizado e senha, `AuthenticationManager`, `DaoAuthenticationProvider` e `PasswordEncoder`. O login é uma operação REST própria do portal, não o grant OAuth2 Resource Owner Password Credentials.

Somente contas ativadas, com situação funcional ativa e sem bloqueios impeditivos podem autenticar ou renovar sessões. Respostas de falha de login são genéricas. Para contas inexistentes, executar verificação de hash equivalente para reduzir diferenças de tempo observáveis.

Após autenticação, criar uma sessão persistente e emitir Access Token JWT e Refresh Token opaco. A criação da sessão deve verificar novamente o estado da conta e a versão das credenciais sob controle transacional, impedindo que login concorrente com revogação utilize credenciais desatualizadas.

## 3. Access Token

Utilizar Spring Security OAuth2 Resource Server e suporte JOSE/Nimbus. Emitir JWT assinado com RS256, chave RSA de no mínimo 2048 bits e identificador `kid`. Somente o `identidade-service` acessa a chave privada. Disponibilizar chaves públicas via JWKS.

| Claim | Conteúdo |
|---|---|
| `sub` | `pessoaId` imutável |
| `iss` | Emissor configurado por ambiente |
| `aud` | Audiência autorizada da API do portal |
| `iat` | Instante de emissão |
| `exp` | Instante de expiração |
| `jti` | Identificador único do token |
| `sid` | Identificador da sessão |
| `roles` | Papel `GERENTE` ou `COLABORADOR` |

Não incluir dados pessoais desnecessários. Validar algoritmo permitido, assinatura, emissor, audiência, expiração, consistência temporal e presença/formato das claims obrigatórias. Validar `nbf` se presente. Configurar tolerância de relógio limitada (inicialmente 30 segundos); não estender a validade efetiva de sessões revogadas ou expiradas. Diferenciar tokens de usuário e serviço por audiência e perfil de claims obrigatório, sem aceitar um no contexto do outro.

Chaves privadas devem ser persistentes entre reinicializações e compartilhadas de maneira segura entre réplicas autorizadas do emissor. Rotacionar usando novo `kid`, publicar a chave nova antes de começar a assinar e manter a pública anterior durante a validade dos tokens emitidos e tolerância de relógio. Em comprometimento, interromper uso e confiança na chave afetada, invalidar sessões pertinentes e garantir atualização dos validadores; não depender apenas da expiração do cache JWKS.

## 4. Sessões e Refresh Token

| Parâmetro | Valor inicial |
|---|---|
| Access Token | 5 minutos |
| Refresh Token | 7 dias |
| Duração absoluta da sessão | 30 dias |

Parâmetros configuráveis externamente. Aplicar expiração deslizante com limite absoluto; nenhum token deve ultrapassar o limite da sessão, inclusive seu prazo de inatividade.

Manter sessões independentes por login. Persistir identificador da sessão, conta associada, criação, última renovação, expiração por inatividade, expiração absoluta, revogação e motivo. Persistir histórico de Refresh Tokens com hash único, sessão, emissão, expiração e consumo/revogação. Nunca persistir Refresh Tokens em texto puro.

Gerar Refresh Tokens opacos com `SecureRandom`, 256 bits de entropia e codificação Base64 URL-safe. Armazenar SHA-256 do token. O token não pode acessar diretamente recursos protegidos.

A cada renovação, validar conta, sessão e token; consumir o token e persistir o sucessor na mesma transação. Serializar operações concorrentes de segurança por conta e sessão, com ordem consistente de locks. Em caso de reutilização de token previamente consumido de uma sessão ainda válida, revogar toda a sessão, registrar evento de segurança e exigir login. Confirmar a transação de revogação mesmo quando a resposta HTTP for erro de autenticação. Tokens desconhecidos não podem revogar sessões arbitrárias. Não repetir automaticamente uma renovação cujo resultado seja incerto; exigir novo login se necessário.

Manter histórico de tokens consumidos pelo menos até o fim da validade absoluta da sessão. Após esse prazo, aplicar política de limpeza e retenção de auditoria.

Logout atual revoga exclusivamente a sessão identificada pelo `sid`; logout global revoga todas as sessões da conta. Alteração e recuperação de senha revogam todas as sessões. Nenhuma operação de revogação retorna sucesso antes do commit. Logout e alterações de conta não podem restaurar tokens revogados.

## 5. Validação de sessão e autorização

A cada requisição protegida de usuário, o serviço de destino valida o JWT e consulta o estado atual da sessão e conta. O `identidade-service` realiza a consulta localmente; o `colaborador-service` usa endpoint interno autenticado do `identidade-service`. Confirmar associação entre `sub`, `sid` e conta, situação funcional, ativação, bloqueios, papel vigente e prazos da sessão. Consultar a base primária, não réplica de leitura sujeita a atraso.

Aplicar Fail Closed: falha de integração, timeout ou indisponibilidade não autoriza acesso. Não utilizar cache positivo de sessões ou permissões. A validação local do JWT no Gateway não substitui a verificação no serviço de destino.

Utilizar `SecurityFilterChain` nos serviços MVC, `SecurityWebFilterChain` no Gateway reativo, `SessionCreationPolicy.STATELESS` nas APIs servlet e `@EnableMethodSecurity` / `@PreAuthorize` nos serviços de negócio. Converter `roles` em authorities `ROLE_GERENTE` e `ROLE_COLABORADOR`.

- GERENTE: operações administrativas autorizadas, respeitando restrições de negócio.
- COLABORADOR: consulta exclusivamente dos próprios dados funcionais por operação dedicada, além das funcionalidades de identidade permitidas ao titular.
- Identidade e propriedade derivam do contexto autenticado, nunca de um identificador arbitrário recebido do cliente.
- Tokens de serviço não conferem permissões administrativas de usuário.

HTTP 401 para credenciais inválidas, expiradas ou revogadas; 403 para identidade válida sem autorização; 429 para limites excedidos; 503 para impossibilidade temporária de verificar segurança. Padronizar erros em `05-CONTRATOS.md`, sem detalhes internos.

A revogação impede novas autorizações após seu commit, não cancela retroativamente requisições em execução. Operações críticas de alteração de segurança devem verificar novamente as condições relevantes no ponto de efetivação, com proteção transacional no serviço responsável.

## 6. Credenciais e política de senhas

Utilizar `DelegatingPasswordEncoder` com BCrypt como algoritmo de escrita padrão, identificador `{bcrypt}`, salt aleatório automático e fator de custo inicial 12. Validar desempenho antes de produção, mantendo custo mínimo 10. Permitir atualização de hash após autenticação bem-sucedida sem sobrescrever alterações concorrentes de senha.

- Mínimo de 15 caracteres e máximo de 72 bytes em UTF-8; rejeitar entradas maiores sem truncamento.
- Permitir espaços, Unicode e colagem de senhas. Não impor composição artificial nem troca periódica sem motivo de segurança.
- Rejeitar senhas comuns/comprometidas mediante lista local versionada ou mecanismo equivalente que não envie senhas em texto puro a terceiros.
- Não normalizar silenciosamente, registrar ou transmitir senhas em eventos.
- Alteração exige senha atual correta e nova senha diferente, conforme RF-021.

BCrypt foi escolhido por maturidade e integração com o ecossistema Java; a limitação de 72 bytes e a alternativa Argon2id devem constar na decisão arquitetural.

## 7. Ativação, recuperação e e-mail

| Configuração | Valor inicial |
|---|---|
| Convite de ativação | 24 horas |
| Token de recuperação | 15 minutos |
| Entropia | 256 bits (`SecureRandom`) |
| Persistência | SHA-256 do token |

Tokens são opacos, específicos por finalidade e conta, revogáveis e de uso único. Validar elegibilidade da conta, expiração e finalidade. Reemissão invalida tokens anteriores da mesma finalidade. Consumo e alteração de credenciais ocorrem na mesma transação. Ativação e recuperação não criam sessão automaticamente. Recuperação não ativa conta pendente nem remove bloqueios. Alteração de senha invalida tokens pendentes de recuperação.

Solicitação pública de recuperação ou reenvio deve retornar resposta genérica e não revelar existência, situação ou elegibilidade da conta. Aplicar rate limiting por origem e identificador normalizado.

Utilizar Spring Boot Mail, SMTP, Mailpit no desenvolvimento e Amazon SES em produção. Envio assíncrono com fila persistente de notificações (outbox local), retentativas limitadas e monitoramento. Para permitir reenvio após falha sem armazenar tokens em claro, proteger o conteúdo temporário necessário ao envio com criptografia autenticada e chave externa; excluir o conteúdo sensível após envio confirmado ou expiração. Não incluir tokens em eventos Kafka, logs ou auditoria. Entrega de e-mail pode ocorrer mais de uma vez, sem gerar novos tokens a cada tentativa de envio.

Links de ativação e recuperação apontam para origem HTTPS configurada pelo servidor; não construir links a partir de cabeçalhos arbitrários. Preferir token no fragmento da URL, lido pelo frontend e removido do histórico após captura; enviá-lo ao backend em corpo POST. Sem scripts de terceiros nessas páginas e com `Referrer-Policy: no-referrer`.

## 8. Segurança do frontend Angular

Utilizar SPA Angular e publicar frontend e API sob a mesma origem em produção. O cliente utiliza Access Token em memória e cabeçalho `Authorization: Bearer`. Refresh Token enviado exclusivamente por cookie `HttpOnly`, `Secure` em produção, `SameSite=Strict`, sem atributo Domain, com Path restrito ao conjunto de endpoints de autenticação que necessitam dele. Não armazenar credenciais em localStorage ou sessionStorage. Respostas de autenticação devem utilizar `Cache-Control: no-store`.

Proteger login, refresh e logout e outras operações autenticadas por cookie contra CSRF: verificar origem permitida e utilizar mecanismo CSRF do Spring Security com token enviado em cabeçalho pelo Angular. Disponibilizar obtenção inicial do token CSRF e renová-lo conforme fluxo. Não desativar CSRF globalmente por utilizar JWT; exceções somente para endpoints exclusivamente Bearer sem autenticação automática por cookies.

Implementar interceptor HTTP para anexar Access Token apenas à própria API. Coordenar renovação single-flight, inclusive entre abas (Web Locks e comunicação entre abas sem persistir tokens); em clientes sem coordenação suportada, falhar de forma segura e solicitar novo login quando necessário. Evitar retries automáticos de operações mutáveis não idempotentes. Guards e ocultação de componentes são controles de experiência, não autorização.

Restringir CORS a origens explicitamente autorizadas em desenvolvimento. Não utilizar origem curinga com credenciais. Aplicar CSP adequada ao Angular, proteção contra framing, HSTS em produção, `X-Content-Type-Options` e evitar bypasses de sanitização. Limpar estado local e cookie no encerramento de sessão conforme confirmação do servidor.

## 9. Comunicação autenticada entre microsserviços

Utilizar OAuth2 Client Credentials para chamadas internas, com suporte de Authorization Server do Spring Security compatível com Spring Boot 4 (Spring Authorization Server), OAuth2 Client e Resource Server. O `identidade-service` hospeda a emissão de tokens de serviço. Não utilizar password grant. O login REST próprio de usuários e a emissão de seus JWTs compartilham infraestrutura de chaves, mas possuem fluxos e validação distintos.

Cada aplicação possui client ID e secret exclusivos, registro persistente e scopes mínimos por operação. Armazenar secrets de clientes por hash no emissor e disponibilizar as credenciais originais somente ao cliente autorizado por gestão externa de segredos. Utilizar token JWT de serviço com RS256, audiência interna específica, subject identificando o cliente, scopes e validade inicial de 5 minutos, sem refresh token ou sessão de usuário. Reutilizar token de serviço até próximo da expiração; não emitir a cada chamada.

Exigir audiência, scopes e identidade do serviço autorizado em endpoints internos. Para alterações iniciadas por usuário, encaminhar contexto do ator de forma autenticada; operações críticas devem validar sua identidade/permissão no serviço responsável, sem confiar em cabeçalhos públicos. Rotação e comprometimento de credenciais de serviço exigem procedimento de bloqueio do cliente e invalidação efetiva de tokens internos, com verificação de cliente habilitado para operações críticas.

Proteger comunicação HTTP com TLS, inclusive interna em produção. Restringir rede por NetworkPolicy e Security Groups. Não rotear endpoints internos nem o endpoint de emissão de tokens de serviço pelo Gateway público.

## 10. Consistência distribuída de segurança

Aplicar Saga Orquestrada persistente no `colaborador-service`, com identificador de operação, estado, versão, dados mínimos de reconciliação e retentativas. Não manter transação de banco aberta durante chamada HTTP. Serializar alterações sensíveis por pessoa e impedir operações conflitantes enquanto existir transição pendente. Registrar solicitações com idempotência e rejeitar repetição com parâmetros divergentes.

### 10.1. Inativação

1. Registrar intenção persistente, reservando a transição e validando regras cadastrais.
2. Solicitar ao `identidade-service` bloqueio cadastral por operação/versionamento via HTTP autenticado. A identidade registra o bloqueio, revoga todas as sessões e invalida tokens de finalidade incompatível na mesma transação. Proteger continuidade administrativa conforme seção 11.
3. Após confirmação durável, concluir situação cadastral inativa e evento Outbox em transação local.
4. Publicar evento Kafka e reconciliar o estado final; preservar possibilidade de recuperação de qualquer etapa.

Falhas ou respostas perdidas não permitem inferir sucesso. Consultar resultado da operação idempotente e retomar. Não desbloquear automaticamente uma conta após falha parcial. Se a conta ainda não foi provisionada, persistir a restrição cadastral/versionamento de modo que provisionamento posterior não habilite acesso indevido.

### 10.2. Reativação

Concluir a alteração cadastral ativa e evento Outbox antes de retirar o bloqueio cadastral correspondente na identidade. A identidade só permite acesso após confirmação da versão e transição esperadas. Preservar bloqueios independentes e situação pendente de ativação; não restaurar sessões anteriores. Reportar conclusão apenas após confirmação da identidade.

### 10.3. Alterações sensíveis

CPF e matrícula: autorização administrativa, validação de unicidade, reautenticação recente do ator (senha atual, prova de curta duração vinculada à sessão e operação, sem repassar senha a outro serviço) e auditoria. Não alteram identificador interno.

E-mail corporativo: tratar por saga, com reserva da unicidade no cadastro e na identidade, bloqueio temporário de alteração, revogação de sessões e de convites/recuperações anteriores antes da conclusão. Após commit cadastral, aplicar novo e-mail na identidade, retirar somente a restrição temporária correspondente e emitir notificações sem credenciais. Conta previamente ativada permanece ativada; conta pendente recebe novo convite se elegível. Não disponibilizar login pelo novo e-mail antes da conclusão cadastral. Manter bloqueios independentes. Caso falhe parcialmente, manter estado seguro e reconciliar, sem liberar reservas prematuramente.

### 10.4. Kafka e recuperação

Eventos cadastrais não podem retirar restrições de segurança ou concluir transições críticas por si sós. Usar `operationId` e versões monotônicas para vincular confirmação síncrona, eventos e reconciliação. Consumidores idempotentes ignoram estados antigos e preservam bloqueios próprios. Persistir resultados de comandos idempotentes até não haver possibilidade de repetição efetiva; rejeitar comandos antigos mesmo após limpeza. Monitorar operações pendentes e encaminhar falhas persistentes para intervenção administrativa auditada.

## 11. Continuidade administrativa e bootstrap

O `identidade-service` é o ponto de serialização das mudanças de elegibilidade administrativa: manter uma linha de controle transacional da organização, bloqueada antes de alterações que reduzam o conjunto de gerentes aptos. Usar locks PostgreSQL/JPA pessimistas e ordem de aquisição definida. Gerente apto possui conta ativada, estado funcional efetivamente ativo, papel GERENTE e ausência de bloqueio impeditivo ou transição que suspenda acesso; não depende de possuir sessão aberta.

Rejeitar operações administrativas que deixem a organização sem gerente apto e impedir autoativação/autoinativação. Incluir todas as transições que afetem elegibilidade, não apenas inativação cadastral. Revogar sessões por logout ou senha não retira elegibilidade se a conta ainda puder autenticar. Limites de tentativas devem ser temporários e não bloquear permanentemente o último gerente. Em incidente real de comprometimento, não manter credenciais comprometidas ativas para satisfazer a contagem: acionar recuperação excepcional auditada. A garantia de continuidade aplica-se às operações administrativas controladas após inicialização; não garante disponibilidade durante falhas externas ou incidentes.

Bootstrap: Kubernetes Job executa `ApplicationRunner` do `colaborador-service` com profile dedicado e sem servidor web. Usar configuração protegida, trava/registro único de inicialização persistente, validações cadastrais e Outbox. Criar somente o primeiro gerente, com conta provisionada via Kafka e convite por e-mail. Antes da ativação inicial, permitir apenas esse fluxo controlado; não expor administração pública. Reexecução idempotente não cria outro gerente. Em desenvolvimento, usar a mesma rotina no Docker Compose.

Recuperação excepcional: rotina operacional separada, executada por operador autorizado com IAM/RBAC restritos, autorização explícita, referência de incidente e auditoria; restaurar acesso por convite/recuperação controlada, nunca por senha fixa ou elevação pública. Detalhar runbook em `09-OPERACAO.md` antes de produção.

## 12. Redis e proteção contra abuso

Utilizar Spring Data Redis e rate limiting distribuído no Spring Cloud Gateway (Redis RateLimiter). Complementar no `identidade-service` por conta/identificador e finalidade, com operações atômicas e TTL. Não utilizar IP como único critério; confiar em cabeçalhos de encaminhamento somente de proxies autorizados.

Parâmetros iniciais configuráveis: login 5 falhas por identificador em 15 minutos e 30 por IP em 15 minutos; recuperação/reenvio 3 por identificador por hora e 20 por IP por hora; ativação, refresh e reautenticação com limites por IP e sessão/identificador aplicável, inicialmente 30 por minuto por IP e 10 por minuto por sessão/identificador. Implementar proteção progressiva e temporária, sem bloqueio permanente induzido por terceiros. Não revelar existência de conta. Em falha do limitador crítico, rejeitar temporariamente novas tentativas sensíveis com 503. Ajustar limites por testes e monitoramento; não tratar os valores como metas de capacidade.

Utilizar Spring Cache e Redis para consultas de setores, inicialmente detalhes por ID e TTL de 60 segundos, com invalidação após commit, chaves separadas por ambiente/domínio e fallback de leitura ao PostgreSQL. Autorizar antes de acessar o cache. Validações de situação ativa, integridade, unicidade, sessões e decisões de segurança consultam dados autoritativos. Não usar cache de dados pessoais na primeira entrega. Documentar consistência eventual e estratégia de invalidação em `03-ARQUITETURA.md`; ampliar cache somente após medição.

## 13. Infraestrutura, AWS e segredos

- Desenvolvimento: Docker Compose com PostgreSQL, Kafka, Redis e Mailpit; credenciais locais não reutilizadas fora de desenvolvimento.
- Produção-alvo: Amazon EKS, VPC com serviços e dados em sub-redes privadas, balanceador HTTPS (ALB), ACM, RDS PostgreSQL, MSK, ElastiCache compatível com Redis OSS e SES. Manter apenas entrada autorizada publicamente. Registry AWS ECR como destino de publicação para deploy na AWS; promover o mesmo digest de imagem aprovado. A arquitetura existente deverá ser alinhada quanto a GHCR/ECR.
- Utilizar Terraform para infraestrutura e Helm para recursos Kubernetes. Isolar ambientes e contas/credenciais; não utilizar dados reais de produção em testes.
- IAM de menor privilégio, EKS Pod Identity ou mecanismo equivalente, RBAC Kubernetes restrito e GitHub Actions autenticado na AWS via OIDC, sem chaves AWS permanentes no pipeline.
- AWS Secrets Manager e KMS para segredos e criptografia; disponibilizar secrets somente aos workloads autorizados, preferencialmente por volume CSI, e prever rotação/reload. Não versionar secrets, arquivos de configuração sensíveis ou state Terraform contendo credenciais. Proteger state remoto com criptografia, controle de acesso e locking.
- TLS para HTTP, PostgreSQL, Kafka e Redis em produção, autenticação de clientes e permissões próprias por aplicação. ACLs Kafka restritas aos tópicos necessários. Habilitar criptografia em repouso e backups protegidos.
- Pods não root, privilégios mínimos, NetworkPolicy com negação padrão e regras explícitas para fluxos necessários. Actuator e interfaces administrativas restritos à rede operacional.
- Configurar AWS Budgets e alertas antes de provisionar; aprovar custos e dimensionamento em `09-OPERACAO.md`. Serviços gerenciados podem gerar custos contínuos. Infraestrutura de aprendizado deve poder ser destruída com segurança, preservando dados necessários.

## 14. Auditoria, privacidade e monitoramento

Registrar login e falhas, ativação, recuperação, alteração de senha, logout, reutilização de token, bloqueios, mudanças sensíveis, bootstrap e recuperação administrativa. Registrar tipo, instante UTC, ator, alvo, resultado, motivo não sensível, identificador da operação e trace ID. Para mudanças de estado, persistir auditoria local na mesma transação da alteração; encaminhamento externo pode ser assíncrono. Restringir acesso e alteração dos registros.

Não registrar senhas, hashes de senhas, tokens completos, cookies, cabeçalhos Authorization, segredos ou dados pessoais desnecessários. Evitar identificadores pessoais em labels de métricas. Configurar alertas para falhas de segurança, reutilização de refresh, operações de saga pendentes, falhas de e-mail, indisponibilidade de validação e abuso de endpoints. Política inicial de retenção de auditoria de 180 dias, configurável e sujeita à avaliação de finalidade, LGPD e requisitos institucionais antes de produção.

## 15. Testes de segurança obrigatórios

Utilizar Spring Security Test, JUnit, Mockito, MockMvc/WebTestClient, Testcontainers (PostgreSQL, Kafka, Redis) e WireMock conforme a natureza do teste.

- Login válido/inválido, conta pendente/inativa/bloqueada, respostas genéricas e limites de abuso.
- JWT inválido, expirado, assinatura incorreta, `iss`/`aud` incorretos, claims ausentes, troca indevida de tokens de serviço e usuário.
- RBAC e acesso a recursos alheios; tentativa de contornar o Gateway; scopes internos insuficientes.
- Refresh simultâneo, reutilização, expiração absoluta/deslizante, logout individual/global e corridas entre login/refresh e revogação.
- Ativação/recuperação de uso único sob concorrência e revogação após troca de senha.
- Saga interrompida em cada etapa, timeout com commit remoto, repetição de comandos e eventos antigos/duplicados, reativação com bloqueio independente, provisionamento concorrente com inativação.
- Inativação concorrente de gerentes, último gerente apto, bootstrap repetido e recuperação excepcional autorizada.
- Falha de Redis e do serviço de identidade, invalidação de cache cadastral, CSRF/CORS/cookies e rotas internas inacessíveis externamente.
- Rotação de chaves, isolamento de ambientes, ausência de segredos em logs, imagens e respostas.

## 16. Referências e implementação incremental

- Requisitos: `02-REQUISITOS.md`, RF-001 a RF-021.
- Arquitetura: `03-ARQUITETURA.md` (atualizar para refletir frontend, Redis, Authorization Server e AWS).
- Spring Security: https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html
- Spring Boot OAuth2: https://docs.spring.io/spring-boot/reference/security/oauth2.html
- OWASP Password Storage: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- OWASP Session Management: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- AWS EKS Security: https://docs.aws.amazon.com/eks/latest/best-practices/security.html

Implementar somente os mecanismos exigidos pelo incremento correspondente. Definir contratos REST/Kafka e testes de aceite antes do código que os utiliza. Registrar ADRs de decisões significativas. Validar versões compatíveis das dependências, parâmetros de desempenho, configurações definitivas de infraestrutura e custos antes de cada entrega aplicável. Este documento não indica que mecanismos já foram implementados ou testados.
