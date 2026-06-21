# In-Memory Blog API — Spring Boot + JWT

## Project Structure

```
src/main/java/com/example/blog/
├── BlogApplication.java
├── model/
│   ├── User.java
│   ├── Post.java
│   ├── Comment.java
│   ├── JwtRequest.java
│   └── JwtResponse.java
├── repository/
│   ├── UserRepository.java
│   ├── PostRepository.java
│   └── CommentRepository.java
├── security/
│   ├── JwtUtil.java
│   ├── JwtRequestFilter.java
│   └── SecurityConfig.java
├── service/
│   ├── UserDetailsServiceImpl.java
│   ├── PostService.java
│   └── CommentService.java
└── controller/
    ├── AuthController.java
    ├── PostController.java
    └── CommentController.java
```

---

## Step 1: `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.5</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>blog</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>blog</name>

    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>
        <!-- Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.1</version>
        </dependency>

        <!-- JAXB for Java 11+ -->
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
            <version>2.3.1</version>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Step 2: `application.properties`

```properties
server.port=8080
jwt.secret=blogSecretKey2024
jwt.expiration=86400000
```

---

## Step 3: `BlogApplication.java`

```java
package com.example.blog;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BlogApplication {
    public static void main(String[] args) {
        SpringApplication.run(BlogApplication.class, args);
    }
}
```

---

## Step 4: Models

### `User.java`

```java
package com.example.blog.model;

public class User {
    private Long id;
    private String username;
    private String password;
    private String role; // "ROLE_USER" or "ROLE_ADMIN"

    public User() {}

    public User(Long id, String username, String password, String role) {
        this.id = id;
        this.username = username;
        this.password = password;
        this.role = role;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }
}
```

---

### `Post.java`

```java
package com.example.blog.model;

import java.time.LocalDateTime;

public class Post {
    private Long id;
    private String title;
    private String content;
    private String author;        // username of creator
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public Post() {}

    public Post(Long id, String title, String content, String author) {
        this.id = id;
        this.title = title;
        this.content = content;
        this.author = author;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }

    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }

    public String getAuthor() { return author; }
    public void setAuthor(String author) { this.author = author; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

---

### `Comment.java`

```java
package com.example.blog.model;

import java.time.LocalDateTime;

public class Comment {
    private Long id;
    private Long postId;
    private String content;
    private String author;        // username of commenter
    private LocalDateTime createdAt;

    public Comment() {}

    public Comment(Long id, Long postId, String content, String author) {
        this.id = id;
        this.postId = postId;
        this.content = content;
        this.author = author;
        this.createdAt = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public Long getPostId() { return postId; }
    public void setPostId(Long postId) { this.postId = postId; }

    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }

    public String getAuthor() { return author; }
    public void setAuthor(String author) { this.author = author; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}
```

---

### `JwtRequest.java`

```java
package com.example.blog.model;

public class JwtRequest {
    private String username;
    private String password;

    public JwtRequest() {}

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

---

### `JwtResponse.java`

```java
package com.example.blog.model;

public class JwtResponse {
    private String token;
    private String username;
    private String role;

    public JwtResponse(String token, String username, String role) {
        this.token = token;
        this.username = username;
        this.role = role;
    }

    public String getToken() { return token; }
    public String getUsername() { return username; }
    public String getRole() { return role; }
}
```

---

## Step 5: Repositories (In-Memory)

### `UserRepository.java`

```java
package com.example.blog.repository;

import com.example.blog.model.User;
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class UserRepository {

    private final List<User> users = new ArrayList<>();
    private final AtomicLong idCounter = new AtomicLong(1);

    // Pre-load users
    public UserRepository() {
        users.add(new User(idCounter.getAndIncrement(), "admin",
            "{noop}admin123", "ROLE_ADMIN"));
        users.add(new User(idCounter.getAndIncrement(), "user1",
            "{noop}user123", "ROLE_USER"));
        users.add(new User(idCounter.getAndIncrement(), "user2",
            "{noop}user123", "ROLE_USER"));
    }

    public Optional<User> findByUsername(String username) {
        return users.stream()
            .filter(u -> u.getUsername().equals(username))
            .findFirst();
    }

    public User save(User user) {
        user.setId(idCounter.getAndIncrement());
        users.add(user);
        return user;
    }

    public List<User> findAll() {
        return users;
    }

    public boolean existsByUsername(String username) {
        return users.stream().anyMatch(u -> u.getUsername().equals(username));
    }
}
```

---

### `PostRepository.java`

```java
package com.example.blog.repository;

import com.example.blog.model.Post;
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

@Repository
public class PostRepository {

    private final List<Post> posts = new ArrayList<>();
    private final AtomicLong idCounter = new AtomicLong(1);

    // Pre-load some posts
    public PostRepository() {
        posts.add(new Post(idCounter.getAndIncrement(),
            "Getting Started with Spring Boot",
            "Spring Boot makes it easy to create stand-alone, production-grade applications.",
            "admin"));
        posts.add(new Post(idCounter.getAndIncrement(),
            "JWT Authentication Explained",
            "JWT (JSON Web Token) is a compact, URL-safe means of representing claims.",
            "user1"));
        posts.add(new Post(idCounter.getAndIncrement(),
            "In-Memory Databases for Testing",
            "H2 is a great in-memory database for development and testing.",
            "user2"));
    }

    public List<Post> findAll() {
        return new ArrayList<>(posts);
    }

    public Optional<Post> findById(Long id) {
        return posts.stream().filter(p -> p.getId().equals(id)).findFirst();
    }

    public List<Post> findByAuthor(String author) {
        return posts.stream()
            .filter(p -> p.getAuthor().equals(author))
            .collect(Collectors.toList());
    }

    public Post save(Post post) {
        post.setId(idCounter.getAndIncrement());
        posts.add(post);
        return post;
    }

    public Post update(Post post) {
        posts.removeIf(p -> p.getId().equals(post.getId()));
        posts.add(post);
        return post;
    }

    public boolean deleteById(Long id) {
        return posts.removeIf(p -> p.getId().equals(id));
    }
}
```

---

### `CommentRepository.java`

```java
package com.example.blog.repository;

import com.example.blog.model.Comment;
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

@Repository
public class CommentRepository {

    private final List<Comment> comments = new ArrayList<>();
    private final AtomicLong idCounter = new AtomicLong(1);

    public List<Comment> findByPostId(Long postId) {
        return comments.stream()
            .filter(c -> c.getPostId().equals(postId))
            .collect(Collectors.toList());
    }

    public Optional<Comment> findById(Long id) {
        return comments.stream().filter(c -> c.getId().equals(id)).findFirst();
    }

    public Comment save(Comment comment) {
        comment.setId(idCounter.getAndIncrement());
        comments.add(comment);
        return comment;
    }

    public boolean deleteById(Long id) {
        return comments.removeIf(c -> c.getId().equals(id));
    }

    public void deleteByPostId(Long postId) {
        comments.removeIf(c -> c.getPostId().equals(postId));
    }
}
```

---

## Step 6: JWT Security

### `JwtUtil.java`

```java
package com.example.blog.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.Date;

@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;

    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }

    public String extractUsername(String token) {
        return getClaims(token).getSubject();
    }

    public boolean validateToken(String token, String username) {
        return extractUsername(token).equals(username) && !isExpired(token);
    }

    private boolean isExpired(String token) {
        return getClaims(token).getExpiration().before(new Date());
    }

    private Claims getClaims(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
    }
}
```

---

### `JwtRequestFilter.java`

```java
package com.example.blog.security;

import com.example.blog.service.UserDetailsServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

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
            try {
                username = jwtUtil.extractUsername(token);
            } catch (Exception e) {
                logger.warn("JWT token extraction failed: " + e.getMessage());
            }
        }

        if (username != null &&
                SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
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

### `SecurityConfig.java`

```java
package com.example.blog.security;

import com.example.blog.service.UserDetailsServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.password.NoOpPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Autowired
    private JwtRequestFilter jwtRequestFilter;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
                // Public routes
                .antMatchers("/api/auth/**").permitAll()
                .antMatchers(HttpMethod.GET, "/api/posts/**").permitAll()
                .antMatchers(HttpMethod.GET, "/api/posts/*/comments").permitAll()
                // Protected routes
                .antMatchers(HttpMethod.POST, "/api/posts").authenticated()
                .antMatchers(HttpMethod.PUT, "/api/posts/**").authenticated()
                .antMatchers(HttpMethod.DELETE, "/api/posts/**").authenticated()
                .antMatchers("/api/posts/*/comments/**").authenticated()
                .anyRequest().authenticated()
            .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);

        http.addFilterBefore(jwtRequestFilter,
            UsernamePasswordAuthenticationFilter.class);
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance(); // Use BCrypt in production
    }
}
```

---

## Step 7: Services

### `UserDetailsServiceImpl.java`

```java
package com.example.blog.service;

import com.example.blog.model.User;
import com.example.blog.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.Collections;

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));

        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            Collections.singletonList(
                new SimpleGrantedAuthority(user.getRole()))
        );
    }
}
```

---

### `PostService.java`

```java
package com.example.blog.service;

import com.example.blog.model.Post;
import com.example.blog.repository.CommentRepository;
import com.example.blog.repository.PostRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Service
public class PostService {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private CommentRepository commentRepository;

    public List<Post> getAllPosts() {
        return postRepository.findAll();
    }

    public Optional<Post> getPostById(Long id) {
        return postRepository.findById(id);
    }

    public List<Post> getPostsByAuthor(String author) {
        return postRepository.findByAuthor(author);
    }

    public Post createPost(String title, String content, String author) {
        Post post = new Post();
        post.setTitle(title);
        post.setContent(content);
        post.setAuthor(author);
        post.setCreatedAt(LocalDateTime.now());
        post.setUpdatedAt(LocalDateTime.now());
        return postRepository.save(post);
    }

    public Optional<Post> updatePost(Long id, String title,
            String content, String requestingUser) {
        return postRepository.findById(id).map(post -> {
            // Only author can update
            if (!post.getAuthor().equals(requestingUser)) {
                throw new RuntimeException("Unauthorized: not the author");
            }
            post.setTitle(title);
            post.setContent(content);
            post.setUpdatedAt(LocalDateTime.now());
            return postRepository.update(post);
        });
    }

    public boolean deletePost(Long id, String requestingUser, boolean isAdmin) {
        return postRepository.findById(id).map(post -> {
            // Author or admin can delete
            if (!post.getAuthor().equals(requestingUser) && !isAdmin) {
                throw new RuntimeException("Unauthorized: not the author or admin");
            }
            commentRepository.deleteByPostId(id);
            return postRepository.deleteById(id);
        }).orElse(false);
    }
}
```

---

### `CommentService.java`

```java
package com.example.blog.service;

import com.example.blog.model.Comment;
import com.example.blog.repository.CommentRepository;
import com.example.blog.repository.PostRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

@Service
public class CommentService {

    @Autowired
    private CommentRepository commentRepository;

    @Autowired
    private PostRepository postRepository;

    public List<Comment> getCommentsByPostId(Long postId) {
        return commentRepository.findByPostId(postId);
    }

    public Optional<Comment> addComment(Long postId,
            String content, String author) {
        return postRepository.findById(postId).map(post -> {
            Comment comment = new Comment();
            comment.setPostId(postId);
            comment.setContent(content);
            comment.setAuthor(author);
            return commentRepository.save(comment);
        });
    }

    public boolean deleteComment(Long commentId,
            String requestingUser, boolean isAdmin) {
        return commentRepository.findById(commentId).map(comment -> {
            if (!comment.getAuthor().equals(requestingUser) && !isAdmin) {
                throw new RuntimeException("Unauthorized");
            }
            return commentRepository.deleteById(commentId);
        }).orElse(false);
    }
}
```

---

## Step 8: Controllers

### `AuthController.java`

```java
package com.example.blog.controller;

import com.example.blog.model.JwtRequest;
import com.example.blog.model.JwtResponse;
import com.example.blog.model.User;
import com.example.blog.repository.UserRepository;
import com.example.blog.security.JwtUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private UserRepository userRepository;

    // POST /api/auth/login
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody JwtRequest request) {
        try {
            authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                    request.getUsername(), request.getPassword()));
        } catch (BadCredentialsException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body("Invalid username or password");
        }

        User user = userRepository.findByUsername(request.getUsername())
            .orElseThrow();
        String token = jwtUtil.generateToken(request.getUsername());
        return ResponseEntity.ok(
            new JwtResponse(token, user.getUsername(), user.getRole()));
    }

    // POST /api/auth/register
    @PostMapping("/register")
    public ResponseEntity<?> register(@RequestBody User user) {
        if (userRepository.existsByUsername(user.getUsername())) {
            return ResponseEntity.badRequest()
                .body("Username already exists");
        }
        user.setRole("ROLE_USER");
        User saved = userRepository.save(user);
        String token = jwtUtil.generateToken(saved.getUsername());
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new JwtResponse(token, saved.getUsername(), saved.getRole()));
    }
}
```

---

### `PostController.java`

```java
package com.example.blog.controller;

import com.example.blog.model.Post;
import com.example.blog.service.PostService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/posts")
public class PostController {

    @Autowired
    private PostService postService;

    // GET /api/posts — public
    @GetMapping
    public ResponseEntity<List<Post>> getAllPosts() {
        return ResponseEntity.ok(postService.getAllPosts());
    }

    // GET /api/posts/{id} — public
    @GetMapping("/{id}")
    public ResponseEntity<?> getPostById(@PathVariable Long id) {
        return postService.getPostById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // GET /api/posts/author/{username} — public
    @GetMapping("/author/{username}")
    public ResponseEntity<List<Post>> getPostsByAuthor(
            @PathVariable String username) {
        return ResponseEntity.ok(postService.getPostsByAuthor(username));
    }

    // POST /api/posts — requires auth
    @PostMapping
    public ResponseEntity<?> createPost(
            @RequestBody Map<String, String> body,
            Authentication auth) {
        String title = body.get("title");
        String content = body.get("content");
        if (title == null || content == null) {
            return ResponseEntity.badRequest()
                .body("Title and content are required");
        }
        Post post = postService.createPost(title, content, auth.getName());
        return ResponseEntity.status(HttpStatus.CREATED).body(post);
    }

    // PUT /api/posts/{id} — requires auth (author only)
    @PutMapping("/{id}")
    public ResponseEntity<?> updatePost(
            @PathVariable Long id,
            @RequestBody Map<String, String> body,
            Authentication auth) {
        try {
            return postService.updatePost(
                    id, body.get("title"), body.get("content"), auth.getName())
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
        } catch (RuntimeException e) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(e.getMessage());
        }
    }

    // DELETE /api/posts/{id} — requires auth (author or admin)
    @DeleteMapping("/{id}")
    public ResponseEntity<?> deletePost(
            @PathVariable Long id,
            Authentication auth) {
        boolean isAdmin = auth.getAuthorities()
            .contains(new SimpleGrantedAuthority("ROLE_ADMIN"));
        try {
            boolean deleted = postService.deletePost(id, auth.getName(), isAdmin);
            if (deleted) return ResponseEntity.ok("Post deleted");
            return ResponseEntity.notFound().build();
        } catch (RuntimeException e) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(e.getMessage());
        }
    }
}
```

---

### `CommentController.java`

```java
package com.example.blog.controller;

import com.example.blog.model.Comment;
import com.example.blog.service.CommentService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/posts/{postId}/comments")
public class CommentController {

    @Autowired
    private CommentService commentService;

    // GET /api/posts/{postId}/comments — public
    @GetMapping
    public ResponseEntity<List<Comment>> getComments(
            @PathVariable Long postId) {
        return ResponseEntity.ok(
            commentService.getCommentsByPostId(postId));
    }

    // POST /api/posts/{postId}/comments — requires auth
    @PostMapping
    public ResponseEntity<?> addComment(
            @PathVariable Long postId,
            @RequestBody Map<String, String> body,
            Authentication auth) {
        String content = body.get("content");
        if (content == null) {
            return ResponseEntity.badRequest().body("Content is required");
        }
        return commentService.addComment(postId, content, auth.getName())
            .map(c -> ResponseEntity.status(HttpStatus.CREATED).body(c))
            .orElse(ResponseEntity.notFound().build());
    }

    // DELETE /api/posts/{postId}/comments/{commentId} — requires auth
    @DeleteMapping("/{commentId}")
    public ResponseEntity<?> deleteComment(
            @PathVariable Long postId,
            @PathVariable Long commentId,
            Authentication auth) {
        boolean isAdmin = auth.getAuthorities()
            .contains(new SimpleGrantedAuthority("ROLE_ADMIN"));
        try {
            boolean deleted = commentService.deleteComment(
                commentId, auth.getName(), isAdmin);
            if (deleted) return ResponseEntity.ok("Comment deleted");
            return ResponseEntity.notFound().build();
        } catch (RuntimeException e) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(e.getMessage());
        }
    }
}
```

---

## API Endpoints Reference

### Auth Endpoints (Public)

| Method | URL                  | Body                                         | Description       |
| ------ | -------------------- | -------------------------------------------- | ----------------- |
| POST   | `/api/auth/login`    | `{"username":"admin","password":"admin123"}` | Get JWT token     |
| POST   | `/api/auth/register` | `{"username":"john","password":"pass123"}`   | Register new user |

### Post Endpoints

| Method | URL                            | Auth            | Description         |
| ------ | ------------------------------ | --------------- | ------------------- |
| GET    | `/api/posts`                   | ❌ Public       | Get all posts       |
| GET    | `/api/posts/{id}`              | ❌ Public       | Get post by ID      |
| GET    | `/api/posts/author/{username}` | ❌ Public       | Get posts by author |
| POST   | `/api/posts`                   | ✅ Required     | Create new post     |
| PUT    | `/api/posts/{id}`              | ✅ Author only  | Update post         |
| DELETE | `/api/posts/{id}`              | ✅ Author/Admin | Delete post         |

### Comment Endpoints

| Method | URL                              | Auth            | Description           |
| ------ | -------------------------------- | --------------- | --------------------- |
| GET    | `/api/posts/{id}/comments`       | ❌ Public       | Get comments for post |
| POST   | `/api/posts/{id}/comments`       | ✅ Required     | Add comment           |
| DELETE | `/api/posts/{id}/comments/{cid}` | ✅ Author/Admin | Delete comment        |

---

## Testing with curl

### 1. Login

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

**Response:**

```json
{
  "token": "eyJhbGciOiJIUzUxMiJ9...",
  "username": "admin",
  "role": "ROLE_ADMIN"
}
```

### 2. Get all posts (public)

```bash
curl http://localhost:8080/api/posts
```

### 3. Create a post (with token)

```bash
curl -X POST http://localhost:8080/api/posts \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"title":"My First Post","content":"Hello World!"}'
```

### 4. Add a comment

```bash
curl -X POST http://localhost:8080/api/posts/1/comments \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"content":"Great post!"}'
```

### 5. Delete a post

```bash
curl -X DELETE http://localhost:8080/api/posts/1 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

## Pre-loaded Data

### Users

| Username | Password   | Role       |
| -------- | ---------- | ---------- |
| `admin`  | `admin123` | ROLE_ADMIN |
| `user1`  | `user123`  | ROLE_USER  |
| `user2`  | `user123`  | ROLE_USER  |

### Posts

| ID  | Title                            | Author |
| --- | -------------------------------- | ------ |
| 1   | Getting Started with Spring Boot | admin  |
| 2   | JWT Authentication Explained     | user1  |
| 3   | In-Memory Databases for Testing  | user2  |

---

## Authorization Rules

| Action         | Who Can Do It           |
| -------------- | ----------------------- |
| Read posts     | Everyone                |
| Create post    | Any logged-in user      |
| Update post    | Only the post author    |
| Delete post    | Post author OR admin    |
| Add comment    | Any logged-in user      |
| Delete comment | Comment author OR admin |

---

## Run the Project

```bash
mvn clean install
mvn spring-boot:run
```

Server starts at `http://localhost:8080`
