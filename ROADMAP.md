# Roadmap de Implementação - Sistema de Solicitação de Acesso

## Visão Geral

Este roadmap organiza a implementação do desafio em fases progressivas, permitindo entregas incrementais e testáveis. O prazo é de **8 dias corridos**.

## Estratégia de Desenvolvimento

- **Abordagem:** Incremental e testável
- **Prioridade:** Features obrigatórias primeiro, depois diferenciais
- **Testes:** TDD recomendado (escrever testes conforme desenvolve)
- **Commits:** Frequentes e descritivos
- **Validação:** Testar cada fase antes de avançar

---

## 📅 Fase 1: Setup e Infraestrutura Base (Dia 1)

**Objetivo:** Preparar ambiente de desenvolvimento e infraestrutura básica

### Tarefas

#### 1.1 Inicialização do Projeto
- [ ] Criar projeto Spring Boot 3.x com Spring Initializr
  - Dependências: Web, Data JPA, Security, Validation, PostgreSQL, H2, Lombok
- [ ] Configurar estrutura de pacotes (Clean Architecture)
- [ ] Configurar `.gitignore` (IDE, target/, logs/, etc)
- [ ] Criar README.md inicial

#### 1.2 Configuração do Banco de Dados
- [ ] Configurar PostgreSQL no `docker-compose.yml`
- [ ] Criar `application.yml` (base)
- [ ] Criar `application-dev.yml` (desenvolvimento)
- [ ] Criar `application-test.yml` (H2)
- [ ] Criar `application-prod.yml` (produção)
- [ ] Testar conexão com banco local

#### 1.3 Docker e Infraestrutura
- [ ] Criar `Dockerfile` com multi-stage build
- [ ] Criar `docker-compose.yml` inicial (só PostgreSQL)
- [ ] Testar build: `mvn clean package`
- [ ] Testar container: `docker-compose up postgres`

#### 1.4 Configuração de Testes
- [ ] Adicionar dependências de teste (JUnit 5, Mockito, Instancio)
- [ ] Configurar JaCoCo no `pom.xml` (mínimo 80%)
- [ ] Criar teste básico para validar configuração
- [ ] Executar: `mvn test`

**Entrega da Fase 1:**
✅ Projeto inicializado
✅ PostgreSQL rodando em container
✅ Configuração de profiles funcionando
✅ JaCoCo configurado

**Tempo Estimado:** 4-6 horas

---

## 📅 Fase 2: Modelo de Dados e Repositórios (Dia 1-2)

**Objetivo:** Criar entidades, relacionamentos e camada de persistência

### Tarefas

#### 2.1 Criar Enums
- [ ] `DepartmentType` (TI, FINANCEIRO, RH, OPERACOES, OUTROS)
- [ ] `RequestStatus` (ATIVO, NEGADO, CANCELADO)

#### 2.2 Criar Entidades JPA
- [ ] `User` (id, email, password, name, department, active, timestamps)
- [ ] `Module` (id, name, description, allowedDepartments, incompatibleModules, active)
- [ ] `AccessRequest` (id, protocol, user, modules, justification, urgent, status, etc)
- [ ] `AccessHistory` (id, accessRequest, action, description, occurredAt)
- [ ] Configurar relacionamentos (@ManyToOne, @ManyToMany, etc)
- [ ] Adicionar auditoria (@CreatedDate, @LastModifiedDate)

#### 2.3 Criar Repositories
- [ ] `UserRepository extends JpaRepository<User, Long>`
  - `Optional<User> findByEmail(String email)`
- [ ] `ModuleRepository extends JpaRepository<Module, Long>`
  - `List<Module> findByActiveTrue()`
- [ ] `AccessRequestRepository extends JpaRepository<AccessRequest, Long>`
  - `List<AccessRequest> findActiveRequestsByUserIdAndModuleIds(...)`
  - `List<AccessRequest> findActiveAccessByUserIdAndModuleIds(...)`
  - `List<Module> findActiveModulesByUserId(Long userId)`
  - `Long countActiveModulesByUserId(Long userId)`
  - Queries de busca com filtros (Specification ou @Query)

#### 2.4 Criar Dados Iniciais
- [ ] Criar `data.sql` ou migration Flyway/Liquibase
- [ ] Popular 4 usuários (um de cada departamento)
- [ ] Popular 10 módulos conforme especificação
- [ ] Popular incompatibilidades de módulos
- [ ] Popular departamentos permitidos por módulo

#### 2.5 Testes de Repository
- [ ] Testes de `UserRepository`
- [ ] Testes de `ModuleRepository`
- [ ] Testes de `AccessRequestRepository` (queries customizadas)
- [ ] Verificar cobertura com `mvn jacoco:report`

**Entrega da Fase 2:**
✅ Todas as entidades criadas e mapeadas
✅ Repositories funcionando
✅ Dados iniciais carregados
✅ Testes de repository com cobertura

**Tempo Estimado:** 6-8 horas

---

## 📅 Fase 3: Segurança e Autenticação JWT (Dia 2-3)

**Objetivo:** Implementar autenticação JWT completa

### Tarefas

#### 3.1 Configuração de Segurança
- [ ] Adicionar dependência `spring-boot-starter-security`
- [ ] Adicionar dependência `jjwt` (JWT)
- [ ] Criar `SecurityConfig` (permitir `/api/auth/**`, proteger resto)
- [ ] Configurar `PasswordEncoder` (BCrypt com 12 rounds)
- [ ] Configurar sessão STATELESS

#### 3.2 JWT Token Provider
- [ ] Criar `JwtTokenProvider`
  - `generateToken(UserDetails)`
  - `validateToken(String token)`
  - `extractUsername(String token)`
- [ ] Configurar secret e expiration (15 minutos)
- [ ] Usar variáveis de ambiente para secret

#### 3.3 Filtros e UserDetails
- [ ] Criar `JwtAuthenticationFilter extends OncePerRequestFilter`
- [ ] Criar `UserDetailsServiceImpl implements UserDetailsService`
  - Carregar usuário do banco por email
- [ ] Integrar filtro na cadeia de segurança

#### 3.4 DTOs de Autenticação
- [ ] Criar `LoginRequest` (email, password)
- [ ] Criar `AuthResponse` (token, type, expiresIn, username)

#### 3.5 Service e Controller
- [ ] Criar `AuthService`
  - `authenticate(LoginRequest)`
  - Validar credenciais
  - Gerar token JWT
- [ ] Criar `AuthController`
  - `POST /api/auth/login`

#### 3.6 Testes de Segurança
- [ ] Teste unitário `JwtTokenProvider`
- [ ] Teste unitário `AuthService`
- [ ] Teste de integração `AuthController`
  - Login com credenciais válidas (200)
  - Login com credenciais inválidas (401)
- [ ] Teste de acesso sem token (403)
- [ ] Teste de acesso com token inválido (403)

**Entrega da Fase 3:**
✅ Login funcionando
✅ JWT gerado corretamente
✅ Endpoints protegidos
✅ Testes de segurança passando

**Tempo Estimado:** 6-8 horas

---

## 📅 Fase 4: Regras de Negócio Core (Dia 3-4)

**Objetivo:** Implementar validações e regras de negócio

### Tarefas

#### 4.1 Criar DTOs de Request
- [ ] `CreateAccessRequestDto`
  - moduleIds (1-3), justification (20-500), urgent
  - Validações: @NotNull, @Size, @NotEmpty
- [ ] `RenewAccessRequestDto`
- [ ] `CancelRequestDto` (cancellationReason 10-200)

#### 4.2 Criar DTOs de Response
- [ ] `AccessRequestResponse`
  - protocol, modules, status, justification, urgent, dates, denialReason
- [ ] `ModuleResponse`
  - name, description, allowedDepartments, active, incompatibleModules
- [ ] `PageResponse<T>` (content, page, size, totalPages, totalElements)

#### 4.3 Criar Mappers (MapStruct ou manual)
- [ ] `AccessRequestMapper`
  - `toResponse(AccessRequest)`
  - `toEntity(CreateAccessRequestDto)`
- [ ] `ModuleMapper`

#### 4.4 Criar ValidationService
- [ ] Validar tamanho de listas (1-3 módulos)
- [ ] Validar justificativa não genérica
  - Bloquear: "teste", "aaa", "preciso"
  - Exigir mínimo 20 caracteres significativos
- [ ] Validar módulos ativos

#### 4.5 Criar BusinessRuleService
- [ ] `validateNoDuplicateActiveRequest(userId, moduleIds)`
  - Usuário não pode ter solicitação ativa para mesmo módulo
- [ ] `validateNoExistingAccess(userId, moduleIds)`
  - Usuário não pode solicitar módulo que já possui
- [ ] `validateDepartmentCompatibility(department, modules)`
  - TI: todos os módulos
  - Financeiro: Financeiro, Relatórios, Portal
  - RH: RH, Relatórios, Portal
  - Operações: Estoque, Compras, Relatórios, Portal
  - Outros: Portal, Relatórios
- [ ] `validateMutuallyExclusiveModules(userId, requestedModules)`
  - Aprovador Financeiro ⚔ Solicitante Financeiro
  - Administrador RH ⚔ Colaborador RH
- [ ] `validateModuleLimit(userId, department, newModulesCount)`
  - TI: máximo 10 módulos
  - Outros: máximo 5 módulos
- [ ] Retornar `ValidationResult` (approved/denied + motivo)

#### 4.6 Criar ProtocolGeneratorService
- [ ] Gerar protocolo: `SOL-YYYYMMDD-NNNN`
- [ ] Sequência diária auto-incrementada
- [ ] Thread-safe (synchronized)

#### 4.7 Testes de Regras de Negócio
- [ ] Testes de `ValidationService` (todas as validações)
- [ ] Testes de `BusinessRuleService` (todos os cenários)
  - Sem `any()`, usar `eq()` e valores específicos
  - Verificar chamadas com `verify()`
- [ ] Testes de `ProtocolGeneratorService`
- [ ] Cobertura: 90%+

**Entrega da Fase 4:**
✅ Todas as regras de negócio implementadas
✅ Validações funcionando
✅ Testes unitários completos (sem `any()`)
✅ Cobertura ≥ 90%

**Tempo Estimado:** 8-10 horas

---

## 📅 Fase 5: Services e Controllers (Dia 4-5)

**Objetivo:** Implementar lógica de aplicação e APIs REST

### Tarefas

#### 5.1 Criar AccessRequestService
- [ ] `create(CreateAccessRequestDto, username)`
  - Buscar usuário autenticado
  - Validar dados (ValidationService)
  - Executar regras de negócio (BusinessRuleService)
  - Se aprovado:
    - Gerar protocolo
    - Criar AccessRequest (status: ATIVO)
    - Definir expiração (180 dias)
    - Criar histórico (CREATED, APPROVED)
    - Retornar: "Solicitação criada! Protocolo: XXX. Acessos disponíveis!"
  - Se negado:
    - Criar AccessRequest (status: NEGADO)
    - Registrar motivo da negação
    - Criar histórico (CREATED, DENIED)
    - Retornar: "Solicitação negada. Motivo: XXX"
- [ ] `list(search, status, startDate, endDate, urgent, page, size, username)`
  - Filtrar apenas solicitações do usuário
  - Aplicar filtros (search em protocol/module, status, período, urgent)
  - Paginar (10 registros por página)
  - Ordenar: mais recentes primeiro
- [ ] `getById(id, username)`
  - Verificar propriedade (apenas suas solicitações)
  - Retornar detalhes completos + histórico
- [ ] `renew(id, request, username)`
  - Validar propriedade
  - Validar: faltam menos de 30 dias para expiração
  - Validar: status atual é ATIVO
  - Criar nova solicitação vinculada
  - Reaplicar regras de negócio
  - Gerar novo protocolo
  - Estender validade por +180 dias (se aprovado)
- [ ] `cancel(id, request, username)`
  - Validar propriedade
  - Validar: status atual é ATIVO
  - Mudar status para CANCELADO
  - Revogar acessos imediatamente
  - Registrar motivo e data no histórico

#### 5.2 Criar ModuleService
- [ ] `listAll()`
  - Retornar todos os módulos ativos
  - Incluir: name, description, allowedDepartments, active, incompatibleModules
- [ ] `getById(id)`
  - Retornar detalhes de um módulo específico

#### 5.3 Criar AccessRequestController
- [ ] `POST /api/access-requests` - Criar solicitação
- [ ] `GET /api/access-requests` - Listar (com filtros e paginação)
- [ ] `GET /api/access-requests/{id}` - Detalhes
- [ ] `POST /api/access-requests/{id}/renew` - Renovar
- [ ] `PATCH /api/access-requests/{id}/cancel` - Cancelar
- [ ] Usar `@AuthenticationPrincipal UserDetails` para obter usuário logado

#### 5.4 Criar ModuleController
- [ ] `GET /api/modules` - Listar todos
- [ ] `GET /api/modules/{id}` - Detalhes

#### 5.5 Exception Handling
- [ ] Criar `GlobalExceptionHandler` (@ControllerAdvice)
- [ ] Criar exceções customizadas:
  - `BusinessException` (400)
  - `UnauthorizedException` (401)
  - `ForbiddenException` (403)
  - `NotFoundException` (404)
  - `ValidationException` (422)
- [ ] Tratar `MethodArgumentNotValidException` (validações @Valid)
- [ ] Retornar respostas padronizadas (ErrorResponse)

#### 5.6 Testes de Service
- [ ] Testes de `AccessRequestService`
  - Criar solicitação aprovada
  - Criar solicitação negada (cada motivo)
  - Listar com filtros
  - Renovar acesso válido
  - Cancelar solicitação
  - Cenários de erro (não autorizado, etc)
- [ ] Testes de `ModuleService`
- [ ] Sem `any()`, usar `eq()` e valores específicos
- [ ] Verificar chamadas com `verify()`

#### 5.7 Testes de Controller (MockMvc)
- [ ] Testes de `AccessRequestController`
  - Criar solicitação (201)
  - Criar solicitação inválida (400/422)
  - Listar solicitações (200)
  - Renovar acesso (200)
  - Cancelar solicitação (204)
  - Acessar solicitação de outro usuário (403)
- [ ] Testes de `ModuleController`
- [ ] Usar `@WithMockUser` ou token JWT real

**Entrega da Fase 5:**
✅ Todos os endpoints funcionando
✅ CRUD completo de solicitações
✅ Validações e regras aplicadas
✅ Testes de service e controller
✅ Cobertura ≥ 80%

**Tempo Estimado:** 10-12 horas

---

## 📅 Fase 6: Testes de Integração (Dia 5-6)

**Objetivo:** Garantir funcionamento end-to-end

### Tarefas

#### 6.1 Configurar Testes de Integração
- [ ] Criar perfil `test` com H2
- [ ] Criar `test-data.sql` (dados para testes)
- [ ] Criar `cleanup.sql` (limpar após testes)
- [ ] Configurar `@SpringBootTest` e `@AutoConfigureMockMvc`

#### 6.2 Testes End-to-End
- [ ] **Fluxo Completo 1: Usuário TI**
  1. Login
  2. Criar solicitação (3 módulos)
  3. Verificar aprovação automática
  4. Listar solicitações
  5. Visualizar detalhes
- [ ] **Fluxo Completo 2: Usuário Financeiro**
  1. Login
  2. Solicitar módulo permitido (aprovado)
  3. Solicitar módulo não permitido (negado)
  4. Verificar motivo da negação
- [ ] **Fluxo Completo 3: Módulos Exclusivos**
  1. Solicitar "Aprovador Financeiro" (aprovado)
  2. Solicitar "Solicitante Financeiro" (negado)
  3. Verificar motivo: incompatibilidade
- [ ] **Fluxo Completo 4: Limite de Módulos**
  1. Criar 5 solicitações (aprovadas)
  2. Tentar 6ª solicitação (negado)
  3. Verificar motivo: limite atingido
- [ ] **Fluxo Completo 5: Renovação**
  1. Criar solicitação
  2. Simular data próxima a expiração
  3. Renovar acesso
  4. Verificar nova data de expiração
- [ ] **Fluxo Completo 6: Cancelamento**
  1. Criar solicitação
  2. Cancelar com motivo
  3. Verificar status CANCELADO
  4. Tentar renovar (erro)

#### 6.3 Testes de Segurança
- [ ] Acesso sem autenticação (401)
- [ ] Acesso com token expirado (401)
- [ ] Acesso a recurso de outro usuário (403)
- [ ] Token inválido (401)

#### 6.4 Verificar Cobertura Final
- [ ] Executar: `mvn clean test jacoco:report`
- [ ] Abrir: `target/site/jacoco/index.html`
- [ ] Verificar: cobertura ≥ 80%
- [ ] Se < 80%, adicionar testes nas áreas faltantes

**Entrega da Fase 6:**
✅ Testes de integração completos
✅ Todos os fluxos funcionando
✅ Cobertura ≥ 80%
✅ Relatório JaCoCo gerado

**Tempo Estimado:** 6-8 horas

---

## 📅 Fase 7: Documentação e Swagger (Dia 6)

**Objetivo:** Documentar API e projeto

### Tarefas

#### 7.1 Configurar Swagger/OpenAPI
- [ ] Adicionar dependência `springdoc-openapi-starter-webmvc-ui`
- [ ] Criar `OpenApiConfig`
  - Título: "Sistema de Solicitação de Acesso a Módulos"
  - Versão: 1.0.0
  - Descrição completa
  - Servidor: localhost
- [ ] Configurar segurança JWT no Swagger
  - `securitySchemes`: bearerAuth
  - `securityRequirement`: bearerAuth
- [ ] Adicionar anotações nos controllers:
  - `@Tag` (nome e descrição)
  - `@Operation` (summary, description)
  - `@ApiResponse` (códigos de status)
  - `@Parameter` (parâmetros)

#### 7.2 Testar Swagger
- [ ] Acessar: `http://localhost:8080/swagger-ui.html`
- [ ] Fazer login via Swagger
- [ ] Copiar token JWT
- [ ] Clicar em "Authorize" e inserir: `Bearer {token}`
- [ ] Testar todos os endpoints via Swagger

#### 7.3 Atualizar README.md
- [ ] **Descrição do Projeto**
- [ ] **Tecnologias Utilizadas** (versões)
- [ ] **Features do Java 21 Utilizadas**
  - Virtual Threads
  - Record Patterns
  - Pattern Matching for Switch
  - Sequenced Collections
- [ ] **Pré-requisitos**
  - Docker 24+
  - Docker Compose 2.20+
  - (Opcional) Java 21, Maven 3.9+
- [ ] **Como Executar Localmente**
  ```bash
  # Clone o repositório
  git clone ...
  cd access-control-system

  # Configure variáveis (opcional)
  export JWT_SECRET="your-secret-key-256-bits"

  # Suba a infraestrutura
  docker-compose up -d

  # Aguarde ~1 minuto para aplicações iniciarem

  # Acesse Swagger
  http://localhost/swagger-ui.html
  ```
- [ ] **Como Executar os Testes**
  ```bash
  mvn clean test
  mvn jacoco:report

  # Ver relatório
  open target/site/jacoco/index.html
  ```
- [ ] **Credenciais para Teste**
  - TI: ti.user@company.com / senha123
  - Financeiro: financeiro.user@company.com / senha123
  - RH: rh.user@company.com / senha123
  - Operações: operacoes.user@company.com / senha123
- [ ] **Exemplos de Requisições** (curl)
  - Login
  - Criar solicitação
  - Listar solicitações
  - Renovar
  - Cancelar
- [ ] **Arquitetura da Solução**
  - Diagrama de componentes (opcional)
  - Estrutura de pacotes
  - Modelo de dados (descrição)
  - Infraestrutura Docker
- [ ] **Decisões Técnicas**
  - Por que Clean Architecture
  - Uso de Virtual Threads
  - BCrypt com 12 rounds
  - JWT com expiração de 15min
  - Regras de negócio centralizadas
  - Validações em camadas

**Entrega da Fase 7:**
✅ Swagger funcionando e documentado
✅ README.md completo
✅ Exemplos de uso claros

**Tempo Estimado:** 4-6 horas

---

## 📅 Fase 8: Docker e Load Balancing (Dia 6-7)

**Objetivo:** Configurar infraestrutura completa com 3 instâncias e Nginx

### Tarefas

#### 8.1 Otimizar Dockerfile
- [ ] Multi-stage build (build + runtime)
- [ ] Stage 1: Maven build
  - Usar cache de dependências
  - `mvn dependency:go-offline`
- [ ] Stage 2: Runtime
  - Usar `eclipse-temurin:21-jre-alpine`
  - Adicionar usuário não-root
  - Configurar JVM options:
    - `-XX:+UseZGC -XX:+ZGenerational` (Generational ZGC)
    - `-Xms512m -Xmx1024m`
- [ ] Testar build: `docker build -t access-control-app .`

#### 8.2 Configurar docker-compose.yml Completo
- [ ] Serviço `postgres`:
  - PostgreSQL 17 Alpine
  - Health check configurado
  - Volume persistente
- [ ] Serviços `app1`, `app2`, `app3`:
  - Build do Dockerfile
  - Variáveis de ambiente (DB, JWT)
  - Health checks (Actuator)
  - Dependência do postgres (condition: service_healthy)
  - Sem exposição de portas (apenas rede interna)
- [ ] Serviço `nginx`:
  - Nginx 1.25 Alpine
  - Porta 80 exposta
  - Volume com nginx.conf
  - Dependência das apps
  - Health check
- [ ] Rede `app-network` (bridge)
- [ ] Volume `postgres_data`

#### 8.3 Configurar Nginx
- [ ] Criar `nginx.conf`
- [ ] Upstream `backend`:
  - `least_conn` (balanceamento por menos conexões)
  - app1:8080, app2:8080, app3:8080
  - `max_fails=3 fail_timeout=30s`
- [ ] Location `/`:
  - Proxy para upstream
  - Headers: Host, X-Real-IP, X-Forwarded-For
  - `proxy_next_upstream` para failover
- [ ] Location `/swagger-ui/` e `/v3/api-docs`
  - Acessível via proxy
- [ ] Timeouts e buffers configurados

#### 8.4 Configurar Health Checks
- [ ] Adicionar `spring-boot-starter-actuator`
- [ ] Expor endpoint `/actuator/health`
- [ ] Configurar no docker-compose (wget/curl)

#### 8.5 Testar Infraestrutura
- [ ] Executar: `docker-compose down -v` (limpar)
- [ ] Executar: `docker-compose build --no-cache`
- [ ] Executar: `docker-compose up -d`
- [ ] Verificar logs: `docker-compose logs -f`
- [ ] Aguardar health checks (verde)
- [ ] Testar acesso: `curl http://localhost/actuator/health`
- [ ] Testar Swagger: `http://localhost/swagger-ui.html`

#### 8.6 Testar Balanceamento de Carga
- [ ] Fazer múltiplas requisições de login
- [ ] Verificar logs de cada app (app1, app2, app3)
- [ ] Confirmar distribuição de requisições
- [ ] Parar uma app: `docker stop access-control-app1`
- [ ] Fazer requisições (deve continuar funcionando)
- [ ] Verificar failover automático
- [ ] Reiniciar app: `docker start access-control-app1`

**Entrega da Fase 8:**
✅ Docker Compose funcionando
✅ 3 instâncias da aplicação rodando
✅ Nginx balanceando carga
✅ Health checks configurados
✅ Failover funcionando

**Tempo Estimado:** 6-8 horas

---

## 📅 Fase 9: Refinamento e Qualidade (Dia 7)

**Objetivo:** Garantir qualidade, boas práticas e requisitos obrigatórios

### Tarefas

#### 9.1 Revisão de Código
- [ ] Verificar princípios SOLID
- [ ] Eliminar código duplicado
- [ ] Nomenclatura consistente (português OU inglês)
- [ ] Adicionar comentários JavaDoc em métodos públicos
- [ ] Remover imports não utilizados
- [ ] Formatar código (Ctrl+Alt+L no IntelliJ)

#### 9.2 Logs Estruturados
- [ ] Configurar SLF4J + Logback
- [ ] Adicionar logs em pontos críticos:
  - Início/fim de operações de negócio
  - Erros e exceções
  - Eventos de segurança (login, acesso negado)
- [ ] Usar níveis apropriados (INFO, WARN, ERROR)
- [ ] Evitar log de informações sensíveis

#### 9.3 Tratamento de Erros
- [ ] Revisar `GlobalExceptionHandler`
- [ ] Mensagens de erro claras e específicas
- [ ] Não expor stack traces em produção
- [ ] Validar todas as entradas de usuário

#### 9.4 Segurança
- [ ] Senhas criptografadas com BCrypt (12 rounds) ✓
- [ ] JWT expira em 15 minutos ✓
- [ ] Secret do JWT em variável de ambiente ✓
- [ ] Endpoints protegidos corretamente ✓
- [ ] Usuário só acessa suas próprias solicitações ✓
- [ ] Validar autorização em todos os métodos de service

#### 9.5 Performance
- [ ] Configurar Virtual Threads no Spring Boot
  ```java
  @Bean
  public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
      return protocolHandler -> {
          protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
      };
  }
  ```
- [ ] Usar `@Async` com Virtual Threads para operações I/O
- [ ] Otimizar queries N+1 (usar `@EntityGraph` ou JOIN FETCH)
- [ ] Configurar pool de conexões Hikari

#### 9.6 Verificar Requisitos Obrigatórios
- [ ] Java 21 ✓
- [ ] Spring Boot 3.x ✓
- [ ] PostgreSQL 17 ✓
- [ ] H2 para testes ✓
- [ ] JWT funcionando ✓
- [ ] Cobertura de testes ≥ 80% ✓
- [ ] Nenhum `any()` nos testes ✓
- [ ] JaCoCo configurado ✓
- [ ] Docker + Docker Compose ✓
- [ ] Nginx load balancer ✓
- [ ] 3 instâncias da aplicação ✓
- [ ] Swagger funcionando ✓
- [ ] Dados iniciais populados ✓

**Entrega da Fase 9:**
✅ Código limpo e organizado
✅ Logs estruturados
✅ Segurança validada
✅ Performance otimizada
✅ Todos os requisitos obrigatórios atendidos

**Tempo Estimado:** 4-6 horas

---

## 📅 Fase 10: Documentação Final e Entrega (Dia 8)

**Objetivo:** Preparar entrega e validação final

### Tarefas

#### 10.1 Gerar Relatório JaCoCo em PDF
- [ ] Executar: `mvn clean test jacoco:report`
- [ ] Abrir `target/site/jacoco/index.html` no navegador
- [ ] Imprimir página como PDF (Ctrl+P)
- [ ] Salvar: `JACOCO_REPORT.pdf` na raiz do projeto
- [ ] Verificar cobertura ≥ 80%

#### 10.2 Validação do Checklist de Entrega
- [ ] Todos os testes passam: `mvn test`
- [ ] Cobertura de testes ≥ 80% (verificar relatório)
- [ ] `docker-compose up -d` funciona sem erros
- [ ] Consegue fazer login via Postman/curl
- [ ] Consegue criar uma solicitação
- [ ] Nginx está balanceando entre app1, app2 e app3
- [ ] README.md está completo
- [ ] Código compila sem erros: `mvn clean package`
- [ ] Swagger está acessível: `http://localhost/swagger-ui.html`
- [ ] Dados iniciais estão populados
- [ ] `.gitignore` configurado (sem target/, IDE, etc)

#### 10.3 Testar "Do Zero"
- [ ] Clonar projeto em pasta diferente (simular avaliador)
- [ ] Executar: `docker-compose up -d`
- [ ] Aguardar apps subirem (~1-2 min)
- [ ] Acessar Swagger
- [ ] Fazer login com credenciais do README
- [ ] Criar solicitação
- [ ] Verificar funcionamento completo
- [ ] Se algo falhar, corrigir e testar novamente

#### 10.4 Preparar Repositório
- [ ] Revisar histórico de commits (descritivos e organizados)
- [ ] Verificar branch `main` funcionando
- [ ] Remover arquivos desnecessários
- [ ] Criar tag de release: `v1.0.0`
- [ ] Push final: `git push origin main --tags`

#### 10.5 Documentação Adicional (Diferenciais)
- [ ] **Diagrama C4** (opcional mas recomendado)
  - Contexto, Container, Componente
  - Salvar em `docs/architecture/`
- [ ] **ADRs** (Architecture Decision Records)
  - Por que Clean Architecture
  - Por que Virtual Threads
  - Por que BCrypt com 12 rounds
  - Salvar em `docs/adr/`
- [ ] **Guia para IA** (Claude Code, Copilot)
  - Como o projeto está estruturado
  - Padrões de código usados
  - Comandos úteis
  - Salvar em `docs/ai-guide.md`

#### 10.6 Preparar Email de Entrega
- [ ] Link do repositório (público)
- [ ] Currículo atualizado
- [ ] Informar se usou IA (sim/não e quais)
- [ ] Breve resumo da solução (opcional)

**Entrega da Fase 10:**
✅ Relatório JaCoCo em PDF
✅ Checklist validado 100%
✅ Repositório pronto para entrega
✅ Documentação completa

**Tempo Estimado:** 4-6 horas

---

## 🎁 Diferenciais Opcionais (Tempo Extra)

Se houver tempo extra, implementar diferenciais:

### Alta Prioridade
- [ ] **Flyway Migrations** (ao invés de data.sql)
  - Versionamento do schema
  - Migrations para dados iniciais
  - Rollback facilitado

- [ ] **Refresh Token**
  - Token de longa duração
  - Endpoint `/api/auth/refresh`
  - Armazenar em tabela separada

- [ ] **Profiles bem configurados**
  - Dev: H2, logs verbosos, Swagger habilitado
  - Prod: PostgreSQL, logs mínimos, Swagger desabilitado (opcional)

### Média Prioridade
- [ ] **Logs estruturados JSON**
  - Logback com encoder JSON
  - Facilita parsing e análise

- [ ] **Métricas do Actuator**
  - Prometheus endpoint
  - Monitoramento de requisições

### Baixa Prioridade (Alto Impacto)
- [ ] **Frontend simples**
  - React/Vue/Angular ou até jQuery
  - Tela de login
  - Tela de solicitações
  - Deploy com Nginx

- [ ] **Diagrama C4 completo**
  - Context, Container, Component, Code
  - Ferramentas: PlantUML, Structurizr

- [ ] **ADRs completos**
  - Documentar todas as decisões técnicas
  - Formato: Markdown

---

## 📊 Resumo de Esforço

| Fase | Descrição | Tempo Estimado | Dia |
|------|-----------|----------------|-----|
| 1 | Setup e Infraestrutura | 4-6h | 1 |
| 2 | Modelo de Dados | 6-8h | 1-2 |
| 3 | Segurança JWT | 6-8h | 2-3 |
| 4 | Regras de Negócio | 8-10h | 3-4 |
| 5 | Services e Controllers | 10-12h | 4-5 |
| 6 | Testes de Integração | 6-8h | 5-6 |
| 7 | Documentação e Swagger | 4-6h | 6 |
| 8 | Docker e Load Balancing | 6-8h | 6-7 |
| 9 | Refinamento e Qualidade | 4-6h | 7 |
| 10 | Entrega Final | 4-6h | 8 |
| **TOTAL** | | **58-78h** | **8 dias** |

---

## 🎯 Priorização

### Obrigatório (Desclassifica se não tiver)
1. ✅ Java 21
2. ✅ Aplicação sobe com Docker Compose
3. ✅ Cobertura de testes ≥ 80%
4. ✅ Balanceamento de carga funcionando
5. ✅ Tecnologias obrigatórias (Spring Boot 3, PostgreSQL 17, etc)

### Muito Importante (Impacta avaliação)
1. ✅ Regras de negócio funcionando
2. ✅ JWT funcionando corretamente
3. ✅ Testes sem `any()`
4. ✅ Código limpo e SOLID
5. ✅ README completo

### Importante (Diferencial)
1. ⭐ Flyway/Liquibase
2. ⭐ Refresh token
3. ⭐ Logs estruturados
4. ⭐ Profiles bem configurados

### Bônus (Alto impacto)
1. 🌟 Diagrama C4
2. 🌟 ADRs
3. 🌟 Frontend

---

## 🚨 Armadilhas Comuns

### Evitar:
- ❌ Usar `any()` nos testes (desclassifica)
- ❌ Cobertura < 80% (desclassifica)
- ❌ Docker Compose não funcionando (desclassifica)
- ❌ Não testar balanceamento de carga
- ❌ Deixar documentação para o final
- ❌ Não fazer commits frequentes
- ❌ Hardcode de credenciais
- ❌ Expor stack traces em produção
- ❌ Não validar autorização (segurança)
- ❌ Queries N+1 (performance)

### Fazer:
- ✅ Testar cada fase antes de avançar
- ✅ Commits pequenos e descritivos
- ✅ Documentar enquanto desenvolve
- ✅ Usar variáveis de ambiente
- ✅ Escrever testes conforme desenvolve (TDD)
- ✅ Validar checklist de entrega no final
- ✅ Testar "do zero" antes de entregar
- ✅ Usar features do Java 21
- ✅ Aplicar SOLID e Clean Code
- ✅ Manter código legível

---

## 📝 Comandos Rápidos

```bash
# Desenvolvimento
mvn clean test                    # Rodar testes
mvn jacoco:report                 # Gerar relatório cobertura
mvn spring-boot:run               # Rodar aplicação localmente
mvn clean package                 # Build do projeto

# Docker
docker-compose up -d              # Subir infraestrutura
docker-compose logs -f            # Ver logs
docker-compose down               # Parar tudo
docker-compose down -v            # Parar e limpar volumes
docker-compose ps                 # Status dos containers
docker-compose build --no-cache   # Rebuild sem cache

# Testes
curl -X POST http://localhost/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ti.user@company.com","password":"senha123"}'

# Verificar balanceamento
for i in {1..10}; do curl http://localhost/api/modules; done
```

---

## ✅ Checklist Final de Validação

Antes de enviar, verificar:

**Código:**
- [ ] Compila sem erros
- [ ] Testes passam 100%
- [ ] Cobertura ≥ 80%
- [ ] Sem `any()` nos testes
- [ ] Código limpo e legível
- [ ] SOLID aplicado
- [ ] Features Java 21 utilizadas

**Funcionalidades:**
- [ ] Login funcionando
- [ ] Criar solicitação (aprovada/negada)
- [ ] Listar solicitações (filtros e paginação)
- [ ] Visualizar detalhes
- [ ] Renovar acesso
- [ ] Cancelar solicitação
- [ ] Consultar módulos
- [ ] Regras de negócio aplicadas
- [ ] Validações funcionando

**Infraestrutura:**
- [ ] Docker Compose funciona
- [ ] 3 apps rodando
- [ ] Nginx balanceando
- [ ] PostgreSQL funcionando
- [ ] Health checks ok
- [ ] Failover testado

**Documentação:**
- [ ] README completo
- [ ] Swagger acessível
- [ ] Exemplos de uso
- [ ] Credenciais documentadas
- [ ] Relatório JaCoCo em PDF
- [ ] Decisões técnicas explicadas

**Repositório:**
- [ ] `.gitignore` configurado
- [ ] Commits descritivos
- [ ] Branch `main` funcionando
- [ ] Sem arquivos desnecessários
- [ ] Repositório público

---

**Boa sorte! 🚀**

Siga o roadmap, teste cada fase e você terá uma solução completa e de qualidade!
