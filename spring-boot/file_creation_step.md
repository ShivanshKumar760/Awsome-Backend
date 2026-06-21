# Spring Boot JWT — File Creation Order Guide

## Overview

When building a Spring Boot JWT project, files must be created in **dependency order** — a file that depends on another must always be created after it. Creating files in the wrong order leads to compile errors, missing beans, and runtime failures.

---

## The Golden Rule

> **Always create files from the bottom of the dependency chain upward — configuration first, then models, then repositories, then services, then filters, then security, then controllers.**

---

## Phase 1: Foundation

These files have no dependencies and must exist before anything else.

### 1. `pom.xml`

**Reason:** Every single class in the project depends on libraries defined here. JWT tokens, Spring Security, JPA, and H2 all come from Maven dependencies. Nothing compiles without this.

**Key dependencies to include:**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt</artifactId>
        <version>0.9.1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

---

### 2. `application.properties`

**Reason:** Configures the database connection, JWT secret key, server port, and JPA settings. Without this, Spring Boot cannot start and beans like `@Value("${jwt.secret}")` will fail to inject.

```properties
server.port=8080

spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=update
spring.h2.console.enabled=true

jwt.secret=mySecretKey
jwt.expiration=86400000
```

---

## Phase 2: Model Layer

These are plain Java classes with no Spring dependencies. Create them early because every other layer references them.

### 3. `User.java` — JPA Entity

**Reason:** The `User` entity is the core data model. The repository, service, and JWT filter all need to know the structure of a user. Without this, `UserRepository` cannot be defined.

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;
    private String role;
    // getters and setters
}
```

---

### 4. `JwtRequest.java` — Login Request DTO

**Reason:** A simple POJO that represents the login form body. The `AuthController` needs this to accept username and password from the client. No dependencies on any other file.

```java
public class JwtRequest {
    private String username;
    private String password;
    // getters and setters
}
```

---

### 5. `JwtResponse.java` — Login Response DTO

**Reason:** A simple POJO returned to the client after a successful login. It wraps the generated JWT token. No dependencies on any other file.

```java
public class JwtResponse {
    private String token;

    public JwtResponse(String token) {
        this.token = token;
    }
    // getter
}
```

---

## Phase 3: Repository Layer

### 6. `UserRepository.java`

**Reason:** Extends `JpaRepository<User, Long>` — so it directly depends on the `User` entity. Must be created after `User.java`. The service layer depends on this repository to fetch users from the database.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    User findByUsername(String username);
}
```

---

## Phase 4: JWT Utility

### 7. `JwtUtil.java`

**Reason:** A standalone utility component that handles all JWT operations — generating tokens, extracting usernames, and validating expiry. It only depends on the `jjwt` library (already in `pom.xml`). Both the filter and the controller depend on this class.

```java
@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 86400000))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }

    public String extractUsername(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    public boolean validateToken(String token, String username) {
        return extractUsername(token).equals(username) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody()
            .getExpiration()
            .before(new Date());
    }
}
```

---

## Phase 5: Service Layer

### 8. `UserDetailsServiceImpl.java`

**Reason:** Implements Spring Security's `UserDetailsService` interface. Depends on `UserRepository` to load users from the database. This is required by both `SecurityConfig` and `JwtRequestFilter`. Must be created after `UserRepository`.

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username);
        if (user == null) throw new UsernameNotFoundException("User not found");
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            new ArrayList<>()
        );
    }
}
```

---

## Phase 6: JWT Filter

### 9. `JwtRequestFilter.java`

**Reason:** Intercepts every incoming HTTP request and validates the JWT token in the `Authorization` header. Depends on both `JwtUtil` (to validate the token) and `UserDetailsServiceImpl` (to load user details). Must be created after both of them.

```java
@Component
public class JwtRequestFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");
        String token = null;
        String username = null;

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            token = authHeader.substring(7);
            username = jwtUtil.extractUsername(token);
        }

        if (username != null &&
            SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails =
                userDetailsService.loadUserByUsername(username);
            if (jwtUtil.validateToken(token, userDetails.getUsername())) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}
```

---

## Phase 7: Security Configuration

### 10. `SecurityConfig.java`

**Reason:** This is the most complex configuration file. It wires together the JWT filter, UserDetailsService, password encoder, and authentication manager. Must be created only after the filter and service are ready, because it directly autowires both of them.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private JwtRequestFilter jwtRequestFilter;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
                .antMatchers("/authenticate", "/register").permitAll()
                .anyRequest().authenticated()
            .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);

        http.addFilterBefore(jwtRequestFilter,
            UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## Phase 8: Controller

### 11. `AuthController.java`

**Reason:** Created absolutely last. It depends on everything — `AuthenticationManager`, `JwtUtil`, `UserDetailsService`, `UserRepository`, and `PasswordEncoder`. The controller is the entry point for the client but the exit point for the developer — build everything else before touching it.

```java
@RestController
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @PostMapping("/authenticate")
    public ResponseEntity<?> login(@RequestBody JwtRequest request) throws Exception {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword()));
        String token = jwtUtil.generateToken(request.getUsername());
        return ResponseEntity.ok(new JwtResponse(token));
    }

    @PostMapping("/register")
    public ResponseEntity<?> register(@RequestBody User user) {
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        userRepository.save(user);
        return ResponseEntity.ok("User registered successfully");
    }

    @GetMapping("/hello")
    public String hello() {
        return "Hello, authenticated user!";
    }
}
```

---

## Complete File Creation Order

| # | File | Phase | Depends On |
|---|------|-------|------------|
| 1 | `pom.xml` | Foundation | Nothing |
| 2 | `application.properties` | Foundation | Nothing |
| 3 | `User.java` | Model | Nothing |
| 4 | `JwtRequest.java` | Model | Nothing |
| 5 | `JwtResponse.java` | Model | Nothing |
| 6 | `UserRepository.java` | Repository | `User.java` |
| 7 | `JwtUtil.java` | Utility | JWT library in `pom.xml` |
| 8 | `UserDetailsServiceImpl.java` | Service | `UserRepository.java` |
| 9 | `JwtRequestFilter.java` | Filter | `JwtUtil.java` + `UserDetailsServiceImpl.java` |
| 10 | `SecurityConfig.java` | Security | `JwtRequestFilter.java` + `UserDetailsServiceImpl.java` |
| 11 | `AuthController.java` | Controller | Everything above |

---

## What Happens If You Get the Order Wrong

| Wrong Order | Error You Will See |
|-------------|-------------------|
| Repository before Entity | `Cannot resolve symbol 'User'` |
| Filter before JwtUtil | `NullPointerException` on token validation |
| SecurityConfig before Filter | Filter not registered in the chain — all requests pass through |
| Controller before SecurityConfig | Endpoints exposed without authentication |
| Any file before `pom.xml` | `Package does not exist` compile errors |

---

## JWT Request Flow (After All Files Created)

```
Client Request
     │
     ▼
JwtRequestFilter        ← Validates Bearer token
     │
     ▼
SecurityConfig          ← Checks if route needs auth
     │
     ▼
AuthController          ← Handles /authenticate and /register
     │
     ▼
UserDetailsServiceImpl  ← Loads user from DB
     │
     ▼
JwtUtil                 ← Generates / validates token
     │
     ▼
UserRepository          ← Talks to H2 database
     │
     ▼
User entity             ← The data returned
```

---

## Quick Tips

- Always run `mvn clean install` after adding new dependencies to `pom.xml`
- Use `{noop}` prefix for plain text passwords in development: `.password("{noop}password")`
- For Spring Boot 2.x use `javax.servlet`, for Spring Boot 3.x use `jakarta.servlet`
- `WebSecurityConfigurerAdapter` is removed in Spring Boot 3.x — always use `SecurityFilterChain` bean instead
- JWT tokens are **stateless** — always set session policy to `STATELESS` in `SecurityConfig`
