# Documento de Implementação - Sistema de Solicitação de Acesso a Módulos

## 1. Visão Geral

Sistema corporativo para gerenciamento de solicitações de acesso a módulos, com autenticação JWT, concessão automática baseada em regras de negócio e infraestrutura containerizada com balanceamento de carga.

## 2. Stack Tecnológica

### 2.1 Obrigatórias
- **Java 21 LTS** (JDK 21)
- **Spring Boot 3.x** (recomendado: 3.2.x ou superior)
- **Spring Data JPA** - Persistência e ORM
- **Spring Validation** - Validação de dados
- **Spring Security** - Autenticação e autorização
- **PostgreSQL 17** - Banco de dados principal
- **H2 Database** - Testes unitários e integração
- **Maven** - Gerenciamento de dependências
- **Docker & Docker Compose** - Containerização
- **Nginx** - Load Balancer e Proxy Reverso
- **Lombok** - Redução de boilerplate

### 2.2 Ferramentas de Teste (Obrigatórias)
- **JUnit 5** - Framework de testes
- **Mockito** - Mocks (sem uso de `any()`)
- **MockMvc** - Testes de API
- **Spring Security Test** - Testes de segurança
- **JaCoCo** - Cobertura de código (mínimo 80%)
- **Instancio** - Geração de dados para testes

### 2.3 Documentação
- **SpringDoc OpenAPI 3** (Swagger UI)
- **JaCoCo Report** - Relatório de cobertura em PDF

### 2.4 Diferenciais
- **Flyway/Liquibase** - Migrations de banco
- **SLF4J/Logback** - Logs estruturados
- **Spring Actuator** - Health checks e métricas
- **MapStruct** - Mapeamento de DTOs

## 3. Features do Java 21 a Serem Utilizadas

### 3.1 Virtual Threads (JEP 444) ⭐ PRIORIDADE ALTA
**Aplicação no Projeto:**
- Configurar `@Async` com Virtual Threads para operações I/O
- Melhorar throughput do sistema sem overhead de threads tradicionais
- Ideal para requisições simultâneas no load balancer

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

### 3.2 Record Patterns (JEP 440)
**Aplicação no Projeto:**
- Desestruturação de Records em validações
- Pattern matching em processamento de DTOs
- Simplificação de código em Services

```java
public String processRequest(AccessRequest request) {
    return switch(request) {
        case AccessRequest(_, _, true, _) -> "Solicitação urgente processada";
        case AccessRequest(var user, var modules, false, _) when modules.size() > 2
            -> "Múltiplos módulos solicitados";
        default -> "Solicitação padrão";
    };
}
```

### 3.3 Pattern Matching for Switch (JEP 441)
**Aplicação no Projeto:**
- Validação de regras de negócio por departamento
- Processamento de status de solicitações
- Tratamento de exceções customizadas

```java
public boolean validateDepartmentAccess(String department, Module module) {
    return switch(department) {
        case "TI" -> true; // Acesso total
        case "Financeiro" -> List.of("Financeiro", "Relatórios", "Portal").contains(module.getName());
        case "RH" -> List.of("RH", "Relatórios", "Portal").contains(module.getName());
        case "Operações" -> List.of("Estoque", "Compras", "Relatórios", "Portal").contains(module.getName());
        default -> List.of("Portal", "Relatórios").contains(module.getName());
    };
}
```

### 3.4 Sequenced Collections (JEP 431)
**Aplicação no Projeto:**
- Manipulação ordenada de solicitações (mais recentes primeiro)
- Histórico de alterações em ordem cronológica
- Gestão de módulos em ordem de prioridade

```java
public SequencedCollection<AccessRequest> getRecentRequests(Long userId) {
    var requests = repository.findByUserIdOrderByCreatedAtDesc(userId);
    return new LinkedHashSet<>(requests); // Mantém ordem de inserção
}
```

### 3.5 String Templates (Preview) - JEP 430
**Aplicação no Projeto:**
- Geração de protocolos formatados
- Mensagens de erro personalizadas
- Logs estruturados

```java
// Usando text blocks e formatação
String protocol = STR."SOL-\{LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE)}-\{String.format("%04d", sequence)}";

String message = STR."""
    Solicitação criada com sucesso!
    Protocolo: \{protocol}
    Módulos: \{String.join(", ", moduleNames)}
    Seus acessos já estão disponíveis!
    """;
```

### 3.6 Unnamed Patterns and Variables (Preview) - JEP 443
**Aplicação no Projeto:**
- Ignorar valores não utilizados em pattern matching
- Simplificar código de validação

```java
// Ao invés de usar variáveis não utilizadas
if (request instanceof UrgentRequest(var user, _, var justification)) {
    // 'modules' não é necessário aqui, usando _
}
```

## 4. Arquitetura da Aplicação

### 4.1 Estrutura de Pacotes (Clean Architecture)

```
src/main/java/com/supera/accesscontrol/
├── config/                      # Configurações
│   ├── SecurityConfig.java
│   ├── AsyncConfig.java
│   ├── OpenApiConfig.java
│   └── JpaAuditConfig.java
├── domain/                      # Camada de Domínio
│   ├── entity/
│   │   ├── User.java
│   │   ├── Module.java
│   │   ├── AccessRequest.java
│   │   ├── AccessHistory.java
│   │   └── Department.java (enum)
│   ├── repository/
│   │   ├── UserRepository.java
│   │   ├── ModuleRepository.java
│   │   └── AccessRequestRepository.java
│   └── enums/
│       ├── RequestStatus.java
│       └── DepartmentType.java
├── application/                 # Camada de Aplicação
│   ├── dto/
│   │   ├── request/
│   │   │   ├── LoginRequest.java
│   │   │   ├── CreateAccessRequestDto.java
│   │   │   ├── RenewAccessRequestDto.java
│   │   │   └── CancelRequestDto.java
│   │   └── response/
│   │       ├── AccessRequestResponse.java
│   │       ├── ModuleResponse.java
│   │       ├── AuthResponse.java
│   │       └── PageResponse.java
│   ├── mapper/
│   │   ├── AccessRequestMapper.java
│   │   └── ModuleMapper.java
│   └── service/
│       ├── AuthService.java
│       ├── AccessRequestService.java
│       ├── ModuleService.java
│       ├── ValidationService.java
│       └── BusinessRuleService.java
├── infrastructure/              # Camada de Infraestrutura
│   ├── security/
│   │   ├── JwtTokenProvider.java
│   │   ├── JwtAuthenticationFilter.java
│   │   ├── UserDetailsServiceImpl.java
│   │   └── SecurityUtils.java
│   └── exception/
│       ├── GlobalExceptionHandler.java
│       ├── BusinessException.java
│       ├── UnauthorizedException.java
│       └── ValidationException.java
└── presentation/                # Camada de Apresentação
    └── controller/
        ├── AuthController.java
        ├── AccessRequestController.java
        └── ModuleController.java
```

### 4.2 Modelo de Dados (Entidades JPA)

#### 4.2.1 User
```java
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password; // BCrypt hash

    @Column(nullable = false)
    private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private DepartmentType department;

    @Column(nullable = false)
    private Boolean active = true;

    @OneToMany(mappedBy = "user")
    private List<AccessRequest> accessRequests;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

#### 4.2.2 Module
```java
@Entity
@Table(name = "modules")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Module {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    @Column(nullable = false, length = 500)
    private String description;

    @ElementCollection
    @CollectionTable(name = "module_departments")
    @Enumerated(EnumType.STRING)
    private List<DepartmentType> allowedDepartments;

    @ManyToMany
    @JoinTable(
        name = "module_incompatibilities",
        joinColumns = @JoinColumn(name = "module_id"),
        inverseJoinColumns = @JoinColumn(name = "incompatible_module_id")
    )
    private List<Module> incompatibleModules;

    @Column(nullable = false)
    private Boolean active = true;

    @CreatedDate
    private LocalDateTime createdAt;
}
```

#### 4.2.3 AccessRequest
```java
@Entity
@Table(name = "access_requests")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AccessRequest {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 20)
    private String protocol; // SOL-YYYYMMDD-NNNN

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToMany
    @JoinTable(
        name = "request_modules",
        joinColumns = @JoinColumn(name = "request_id"),
        inverseJoinColumns = @JoinColumn(name = "module_id")
    )
    private List<Module> modules;

    @Column(nullable = false, length = 500)
    private String justification;

    @Column(nullable = false)
    private Boolean urgent = false;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private RequestStatus status;

    @Column(length = 500)
    private String denialReason;

    @Column(length = 200)
    private String cancellationReason;

    @CreatedDate
    @Column(nullable = false)
    private LocalDateTime requestedAt;

    @Column
    private LocalDateTime approvedAt;

    @Column
    private LocalDateTime expiresAt; // approvedAt + 180 dias

    @Column
    private LocalDateTime cancelledAt;

    @ManyToOne
    @JoinColumn(name = "renewed_from_id")
    private AccessRequest renewedFrom;

    @OneToMany(mappedBy = "accessRequest", cascade = CascadeType.ALL)
    private List<AccessHistory> history;
}
```

#### 4.2.4 AccessHistory
```java
@Entity
@Table(name = "access_history")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AccessHistory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "access_request_id")
    private AccessRequest accessRequest;

    @Column(nullable = false)
    private String action; // CREATED, APPROVED, DENIED, CANCELLED, RENEWED

    @Column(length = 500)
    private String description;

    @CreatedDate
    @Column(nullable = false)
    private LocalDateTime occurredAt;
}
```

### 4.3 Enums

```java
public enum DepartmentType {
    TI("TI", 10),
    FINANCEIRO("Financeiro", 5),
    RH("RH", 5),
    OPERACOES("Operações", 5),
    OUTROS("Outros", 5);

    private final String displayName;
    private final int moduleLimit;

    // Constructor, getters
}

public enum RequestStatus {
    ATIVO,
    NEGADO,
    CANCELADO
}
```

## 5. Regras de Negócio (BusinessRuleService)

### 5.1 Validações Antes da Criação

```java
@Service
@RequiredArgsConstructor
public class BusinessRuleService {

    private final AccessRequestRepository requestRepository;
    private final ModuleRepository moduleRepository;

    // 1. Verificar se usuário tem solicitação ativa para o mesmo módulo
    public void validateNoDuplicateActiveRequest(Long userId, List<Long> moduleIds) {
        var activeRequests = requestRepository
            .findActiveRequestsByUserIdAndModuleIds(userId, moduleIds);
        if (!activeRequests.isEmpty()) {
            throw new BusinessException("Você já possui solicitação ativa para um ou mais módulos");
        }
    }

    // 2. Verificar se usuário já possui acesso ativo ao módulo
    public void validateNoExistingAccess(Long userId, List<Long> moduleIds) {
        var existingAccess = requestRepository
            .findActiveAccessByUserIdAndModuleIds(userId, moduleIds);
        if (!existingAccess.isEmpty()) {
            throw new BusinessException("Você já possui acesso ativo a um ou mais módulos solicitados");
        }
    }

    // 3. Validar justificativa (não genérica)
    public void validateJustification(String justification) {
        var genericTerms = List.of("teste", "aaa", "preciso", "test", "asdf");
        var lowerJustification = justification.toLowerCase().trim();

        if (genericTerms.stream().anyMatch(lowerJustification::contains) &&
            lowerJustification.length() < 30) {
            throw new BusinessException("Justificativa insuficiente ou genérica");
        }
    }

    // 4. Verificar se módulos estão ativos
    public void validateModulesActive(List<Long> moduleIds) {
        var modules = moduleRepository.findAllById(moduleIds);
        if (modules.stream().anyMatch(m -> !m.getActive())) {
            throw new BusinessException("Um ou mais módulos solicitados não estão disponíveis");
        }
    }

    // 5. Compatibilidade de departamento
    public ValidationResult validateDepartmentCompatibility(
            DepartmentType department,
            List<Module> modules) {

        for (Module module : modules) {
            if (department == DepartmentType.TI) {
                continue; // TI pode tudo
            }

            if (!module.getAllowedDepartments().contains(department)) {
                return ValidationResult.denied(
                    "Departamento sem permissão para acessar este módulo: " + module.getName()
                );
            }
        }
        return ValidationResult.approved();
    }

    // 6. Módulos mutuamente exclusivos
    public ValidationResult validateMutuallyExclusiveModules(
            Long userId,
            List<Module> requestedModules) {

        var userActiveModules = requestRepository.findActiveModulesByUserId(userId);

        for (Module requested : requestedModules) {
            for (Module active : userActiveModules) {
                if (requested.getIncompatibleModules().contains(active)) {
                    return ValidationResult.denied(
                        "Módulo incompatível com outro módulo já ativo em seu perfil: "
                        + active.getName()
                    );
                }
            }
        }
        return ValidationResult.approved();
    }

    // 7. Limite de módulos por usuário
    public ValidationResult validateModuleLimit(Long userId, DepartmentType department, int newModulesCount) {
        var currentCount = requestRepository.countActiveModulesByUserId(userId);
        var limit = department.getModuleLimit();

        if (currentCount + newModulesCount > limit) {
            return ValidationResult.denied("Limite de módulos ativos atingido");
        }
        return ValidationResult.approved();
    }
}
```

### 5.2 Geração de Protocolo

```java
@Service
public class ProtocolGeneratorService {

    @Value("${app.protocol.prefix:SOL}")
    private String prefix;

    private final AtomicInteger dailySequence = new AtomicInteger(0);
    private LocalDate lastResetDate = LocalDate.now();

    public synchronized String generateProtocol() {
        var today = LocalDate.now();

        // Reset sequence se mudou o dia
        if (!today.equals(lastResetDate)) {
            dailySequence.set(0);
            lastResetDate = today;
        }

        var sequence = dailySequence.incrementAndGet();
        var dateStr = today.format(DateTimeFormatter.BASIC_ISO_DATE);

        return String.format("%s-%s-%04d", prefix, dateStr, sequence);
    }
}
```

## 6. Segurança (JWT)

### 6.1 Configuração do Spring Security

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configure(http))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/swagger-ui/**", "/v3/api-docs/**", "/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Força 12 rounds para maior segurança
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### 6.2 JWT Token Provider

```java
@Component
@Slf4j
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration:900000}") // 15 minutos em ms
    private Long expiration;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }

    public String generateToken(UserDetails userDetails) {
        var now = new Date();
        var expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(getSigningKey(), SignatureAlgorithm.HS512)
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.error("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }

    private <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
}
```

### 6.3 JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        try {
            var token = extractToken(request);

            if (token != null && jwtTokenProvider.validateToken(token)) {
                var username = jwtTokenProvider.extractUsername(token);
                var userDetails = userDetailsService.loadUserByUsername(username);

                var authentication = new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities()
                );

                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            log.error("Cannot set user authentication: {}", e.getMessage());
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        var bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

## 7. Controllers e Endpoints

### 7.1 AuthController

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Tag(name = "Autenticação", description = "Endpoints de autenticação")
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    @Operation(summary = "Realizar login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        var response = authService.authenticate(request);
        return ResponseEntity.ok(response);
    }
}
```

### 7.2 AccessRequestController

```java
@RestController
@RequestMapping("/api/access-requests")
@RequiredArgsConstructor
@Tag(name = "Solicitações de Acesso", description = "Gerenciamento de solicitações de acesso")
public class AccessRequestController {

    private final AccessRequestService service;

    @PostMapping
    @Operation(summary = "Criar nova solicitação de acesso")
    public ResponseEntity<AccessRequestResponse> create(
            @Valid @RequestBody CreateAccessRequestDto request,
            @AuthenticationPrincipal UserDetails userDetails) {
        var response = service.create(request, userDetails.getUsername());
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping
    @Operation(summary = "Listar solicitações do usuário")
    public ResponseEntity<PageResponse<AccessRequestResponse>> list(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) RequestStatus status,
            @RequestParam(required = false) @DateTimeFormat(iso = ISO.DATE) LocalDate startDate,
            @RequestParam(required = false) @DateTimeFormat(iso = ISO.DATE) LocalDate endDate,
            @RequestParam(required = false) Boolean urgent,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @AuthenticationPrincipal UserDetails userDetails) {
        var response = service.list(search, status, startDate, endDate, urgent, page, size, userDetails.getUsername());
        return ResponseEntity.ok(response);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Visualizar detalhes de uma solicitação")
    public ResponseEntity<AccessRequestResponse> getById(
            @PathVariable Long id,
            @AuthenticationPrincipal UserDetails userDetails) {
        var response = service.getById(id, userDetails.getUsername());
        return ResponseEntity.ok(response);
    }

    @PostMapping("/{id}/renew")
    @Operation(summary = "Renovar acesso")
    public ResponseEntity<AccessRequestResponse> renew(
            @PathVariable Long id,
            @Valid @RequestBody RenewAccessRequestDto request,
            @AuthenticationPrincipal UserDetails userDetails) {
        var response = service.renew(id, request, userDetails.getUsername());
        return ResponseEntity.ok(response);
    }

    @PatchMapping("/{id}/cancel")
    @Operation(summary = "Cancelar solicitação")
    public ResponseEntity<Void> cancel(
            @PathVariable Long id,
            @Valid @RequestBody CancelRequestDto request,
            @AuthenticationPrincipal UserDetails userDetails) {
        service.cancel(id, request, userDetails.getUsername());
        return ResponseEntity.noContent().build();
    }
}
```

### 7.3 ModuleController

```java
@RestController
@RequestMapping("/api/modules")
@RequiredArgsConstructor
@Tag(name = "Módulos", description = "Consulta de módulos disponíveis")
public class ModuleController {

    private final ModuleService service;

    @GetMapping
    @Operation(summary = "Listar módulos disponíveis")
    public ResponseEntity<List<ModuleResponse>> listAll() {
        var response = service.listAll();
        return ResponseEntity.ok(response);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Detalhes de um módulo")
    public ResponseEntity<ModuleResponse> getById(@PathVariable Long id) {
        var response = service.getById(id);
        return ResponseEntity.ok(response);
    }
}
```

## 8. Testes

### 8.1 Configuração do JaCoCo (pom.xml)

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>jacoco-check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 8.2 Exemplo de Teste Unitário (Sem any())

```java
@ExtendWith(MockitoExtension.class)
class BusinessRuleServiceTest {

    @Mock
    private AccessRequestRepository requestRepository;

    @Mock
    private ModuleRepository moduleRepository;

    @InjectMocks
    private BusinessRuleService businessRuleService;

    @Test
    @DisplayName("Deve lançar exceção quando usuário já tem solicitação ativa para o módulo")
    void shouldThrowExceptionWhenUserHasDuplicateActiveRequest() {
        // Arrange
        Long userId = 1L;
        List<Long> moduleIds = List.of(1L, 2L);

        var existingRequest = Instancio.create(AccessRequest.class);
        when(requestRepository.findActiveRequestsByUserIdAndModuleIds(eq(userId), eq(moduleIds)))
            .thenReturn(List.of(existingRequest));

        // Act & Assert
        assertThatThrownBy(() -> businessRuleService.validateNoDuplicateActiveRequest(userId, moduleIds))
            .isInstanceOf(BusinessException.class)
            .hasMessageContaining("já possui solicitação ativa");

        verify(requestRepository, times(1))
            .findActiveRequestsByUserIdAndModuleIds(eq(userId), eq(moduleIds));
    }

    @Test
    @DisplayName("Deve validar com sucesso quando não há solicitações duplicadas")
    void shouldValidateSuccessfullyWhenNoDuplicates() {
        // Arrange
        Long userId = 1L;
        List<Long> moduleIds = List.of(1L, 2L);

        when(requestRepository.findActiveRequestsByUserIdAndModuleIds(eq(userId), eq(moduleIds)))
            .thenReturn(Collections.emptyList());

        // Act & Assert
        assertThatCode(() -> businessRuleService.validateNoDuplicateActiveRequest(userId, moduleIds))
            .doesNotThrowAnyException();

        verify(requestRepository, times(1))
            .findActiveRequestsByUserIdAndModuleIds(eq(userId), eq(moduleIds));
    }
}
```

### 8.3 Exemplo de Teste de Integração

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Sql(scripts = "/test-data.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
class AccessRequestControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    private String validToken;

    @BeforeEach
    void setUp() {
        var userDetails = User.builder()
            .email("test@example.com")
            .build();
        validToken = jwtTokenProvider.generateToken(userDetails);
    }

    @Test
    @DisplayName("Deve criar solicitação com sucesso quando dados válidos")
    void shouldCreateRequestSuccessfully() throws Exception {
        // Arrange
        var request = CreateAccessRequestDto.builder()
            .moduleIds(List.of(1L, 2L))
            .justification("Preciso acessar esses módulos para realizar minhas atividades diárias")
            .urgent(false)
            .build();

        // Act & Assert
        mockMvc.perform(post("/api/access-requests")
                .header("Authorization", "Bearer " + validToken)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.protocol").exists())
            .andExpect(jsonPath("$.status").value("ATIVO"))
            .andExpect(jsonPath("$.message").containsString("Seus acessos já estão disponíveis"));
    }
}
```

## 9. Infraestrutura Docker

### 9.1 Dockerfile (Multi-stage Build)

```dockerfile
# Stage 1: Build
FROM maven:3.9.6-eclipse-temurin-21-alpine AS build
WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseZGC", \
    "-XX:+ZGenerational", \
    "-Xms512m", \
    "-Xmx1024m", \
    "-jar", \
    "app.jar"]
```

### 9.2 docker-compose.yml

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:17-alpine
    container_name: access-control-db
    environment:
      POSTGRES_DB: access_control
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d access_control"]
      interval: 10s
      timeout: 5s
      retries: 5

  app1:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: access-control-app1
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/access_control
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: admin123
      JWT_SECRET: ${JWT_SECRET:-your-secret-key-here-minimum-256-bits-long}
      JWT_EXPIRATION: 900000
      SERVER_PORT: 8080
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  app2:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: access-control-app2
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/access_control
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: admin123
      JWT_SECRET: ${JWT_SECRET:-your-secret-key-here-minimum-256-bits-long}
      JWT_EXPIRATION: 900000
      SERVER_PORT: 8080
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  app3:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: access-control-app3
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/access_control
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: admin123
      JWT_SECRET: ${JWT_SECRET:-your-secret-key-here-minimum-256-bits-long}
      JWT_EXPIRATION: 900000
      SERVER_PORT: 8080
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  nginx:
    image: nginx:1.25-alpine
    container_name: access-control-lb
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app1
      - app2
      - app3
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```

### 9.3 nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        least_conn; # Algoritmo de balanceamento: menos conexões
        server app1:8080 max_fails=3 fail_timeout=30s;
        server app2:8080 max_fails=3 fail_timeout=30s;
        server app3:8080 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name localhost;

        # Logs
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        # Timeout configurations
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffer configurations
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Health check
            proxy_next_upstream error timeout http_500 http_502 http_503;
        }

        # Swagger UI
        location /swagger-ui/ {
            proxy_pass http://backend/swagger-ui/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /v3/api-docs {
            proxy_pass http://backend/v3/api-docs;
            proxy_set_header Host $host;
        }
    }
}
```

## 10. Dados Iniciais (data.sql)

```sql
-- Usuários (senhas BCrypt: "senha123")
INSERT INTO users (email, password, name, department, active, created_at, updated_at) VALUES
('ti.user@company.com', '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5gyg9LCEF3SOS', 'João Silva', 'TI', true, NOW(), NOW()),
('financeiro.user@company.com', '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5gyg9LCEF3SOS', 'Maria Santos', 'FINANCEIRO', true, NOW(), NOW()),
('rh.user@company.com', '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5gyg9LCEF3SOS', 'Pedro Costa', 'RH', true, NOW(), NOW()),
('operacoes.user@company.com', '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5gyg9LCEF3SOS', 'Ana Lima', 'OPERACOES', true, NOW(), NOW());

-- Módulos
INSERT INTO modules (name, description, active, created_at) VALUES
('Portal do Colaborador', 'Portal principal para colaboradores', true, NOW()),
('Relatórios Gerenciais', 'Acesso a relatórios gerenciais', true, NOW()),
('Gestão Financeira', 'Sistema de gestão financeira', true, NOW()),
('Aprovador Financeiro', 'Aprovação de transações financeiras', true, NOW()),
('Solicitante Financeiro', 'Solicitação de transações financeiras', true, NOW()),
('Administrador RH', 'Administração de recursos humanos', true, NOW()),
('Colaborador RH', 'Portal do colaborador de RH', true, NOW()),
('Gestão de Estoque', 'Controle de estoque', true, NOW()),
('Compras', 'Sistema de compras', true, NOW()),
('Auditoria', 'Sistema de auditoria', true, NOW());

-- Departamentos permitidos por módulo
INSERT INTO module_departments (module_id, allowed_departments) VALUES
-- Portal e Relatórios: Todos
(1, 'TI'), (1, 'FINANCEIRO'), (1, 'RH'), (1, 'OPERACOES'), (1, 'OUTROS'),
(2, 'TI'), (2, 'FINANCEIRO'), (2, 'RH'), (2, 'OPERACOES'), (2, 'OUTROS'),
-- Financeiro
(3, 'TI'), (3, 'FINANCEIRO'),
(4, 'TI'), (4, 'FINANCEIRO'),
(5, 'TI'), (5, 'FINANCEIRO'),
-- RH
(6, 'TI'), (6, 'RH'),
(7, 'TI'), (7, 'RH'),
-- Operações
(8, 'TI'), (8, 'OPERACOES'),
(9, 'TI'), (9, 'OPERACOES'),
-- Auditoria (apenas TI)
(10, 'TI');

-- Incompatibilidades
INSERT INTO module_incompatibilities (module_id, incompatible_module_id) VALUES
(4, 5), (5, 4), -- Aprovador e Solicitante Financeiro
(6, 7), (7, 6); -- Administrador e Colaborador RH
```

## 11. Application Properties

### 11.1 application.yml (Base)

```yaml
spring:
  application:
    name: access-control-system

  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true

  servlet:
    multipart:
      max-file-size: 5MB
      max-request-size: 10MB

server:
  port: 8080
  error:
    include-message: always
    include-binding-errors: always
  compression:
    enabled: true

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

app:
  protocol:
    prefix: SOL
  access:
    expiration-days: 180
    renewal-threshold-days: 30

jwt:
  expiration: 900000 # 15 minutos
```

### 11.2 application-test.yml

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect

  h2:
    console:
      enabled: true

jwt:
  secret: test-secret-key-for-testing-purposes-minimum-256-bits
  expiration: 3600000
```

### 11.3 application-prod.yml

```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 20000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

logging:
  level:
    root: INFO
    com.supera.accesscontrol: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

jwt:
  secret: ${JWT_SECRET}
  expiration: ${JWT_EXPIRATION:900000}
```

## 12. Checklist de Implementação

- [ ] Setup inicial do projeto Maven com Spring Boot 3.x
- [ ] Configurar dependências (Spring Data JPA, Security, Validation, etc)
- [ ] Criar estrutura de pacotes (Clean Architecture)
- [ ] Implementar entidades JPA e relacionamentos
- [ ] Criar repositories com queries customizadas
- [ ] Implementar configuração de segurança JWT
- [ ] Desenvolver serviços de negócio
- [ ] Implementar validações e regras de negócio
- [ ] Criar controllers REST
- [ ] Configurar Swagger/OpenAPI
- [ ] Escrever testes unitários (sem `any()`)
- [ ] Escrever testes de integração
- [ ] Configurar JaCoCo e verificar cobertura ≥ 80%
- [ ] Criar Dockerfile com multi-stage build
- [ ] Criar docker-compose.yml com 3 instâncias
- [ ] Configurar Nginx para load balancing
- [ ] Criar data.sql com dados iniciais
- [ ] Configurar profiles (dev, test, prod)
- [ ] Testar aplicação localmente com Docker
- [ ] Gerar relatório JaCoCo em PDF
- [ ] Documentar README.md completo
- [ ] Validar todos os critérios de avaliação

## 13. Comandos Úteis

```bash
# Build do projeto
mvn clean package

# Executar testes
mvn test

# Gerar relatório de cobertura
mvn jacoco:report

# Build da imagem Docker
docker build -t access-control-app .

# Subir toda a infraestrutura
docker-compose up -d

# Ver logs
docker-compose logs -f

# Parar tudo
docker-compose down

# Limpar volumes
docker-compose down -v

# Acessar Swagger
http://localhost/swagger-ui.html
```

---

**Observações Finais:**
- Este documento serve como guia técnico completo
- Todas as decisões técnicas devem ser documentadas
- Manter código limpo e seguir princípios SOLID
- Priorizar features do Java 21 (Virtual Threads, Records, Pattern Matching)
- Garantir cobertura de testes ≥ 80%
- Testar balanceamento de carga fazendo múltiplas requisições
