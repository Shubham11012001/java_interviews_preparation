# Spring Security: The Complete Interview Guide

A Medium-style tutorial for Java Backend Engineers and Technical Architects | Authentication · Authorization · JWT · OAuth 2.0 · OpenID Connect

If you are preparing for Senior Software Engineer, Lead Engineer, or Technical Architect interviews, Spring Security is one of those topics where interviewers expect more than annotations such as `@PreAuthorize` and `@WithMockUser`.

They want to know:

* What happens internally when a user logs in?

* How does Spring Security identify a user on every request?

* What is the difference between authentication and authorization?

* How do JWT, OAuth 2.0, and OpenID Connect work together?

* How do you secure REST APIs and microservices?

* How do you prevent token theft, privilege escalation, CSRF, and other attacks?

* How would you design security for a production system handling millions of requests?

We'll build the understanding from zero, then move toward production architecture and senior-level interview questions.

# 1. What is Spring Security?

Spring Security is a security framework for Java applications that helps protect APIs and web applications by controlling who can access them and what they can do.

Imagine you work in a bank.

* A customer can view their own bank balance.

* A bank employee can view customer details according to their permissions.

* An administrator can manage certain system settings.

* An unauthenticated visitor cannot access a customer's private information.

Spring Security helps enforce these rules.

Without security, your application might expose an API like this:

http

```
GET /api/accounts/12345/balance
```

Anyone who knows the URL might access the endpoint if your application does not enforce access controls.

With Spring Security, the application checks:

1. Is the request allowed to enter the application?

2. Who is making the request?

3. Is the identity valid?

4. Does that identity have permission to access this resource?

5. Is the user allowed to access this particular account?

## Why do we need it?

Spring Security helps with:

* Authentication — verifying identity.

* Authorization — checking permissions.

* Password hashing and credential verification.

* Session management.

* JWT-based authentication.

* OAuth 2.0 and OpenID Connect integration.

* Protection against common web attacks.

* Method-level security.

* Security integration with Spring MVC and Spring Boot.

Architect's perspective: Spring Security is not merely a login library. It is a security enforcement layer that sits across your application's request-processing flow and can also protect individual methods and business operations.

# 2. The four concepts you must never confuse

These four terms are related but have different meanings.

Authentication — Who are you?

Example: Verify a customer's username and password.

Authorization — What are you allowed to do?

Example: A customer can view their account but cannot approve a loan.

JWT — A token format

A compact, digitally protected container for claims about an identity or other subject.

OAuth 2.0 — A delegated authorization framework

Allows a client to obtain limited access to a protected resource without receiving the user's password.

One more important concept:

OpenID Connect (OIDC) adds an identity layer on top of OAuth 2.0. It defines how a client can verify a user's authentication and obtain identity information.

A quick example:

You sign in to an application using your company's identity provider.

* OIDC helps the application establish who signed in.

* OAuth 2.0 governs the application's delegated access to protected resources.

* The identity provider may issue a JWT access token and an ID token.

* Spring Security validates the appropriate credentials and enforces your application's authorization rules.

Remember: JWT and OAuth 2.0 are not competing technologies. JWT is a token format; OAuth 2.0 is a protocol framework.

# 3. How Spring Security works internally

Before studying JWT or OAuth, understand the request flow. This is one of the most important interview topics.

## 3.1 The big-picture architecture

1. Client

Browser, mobile app, or another microservice

2. Servlet container / servlet filters

The HTTP request enters the application

3. DelegatingFilterProxy

Connects the servlet filter mechanism to Spring-managed security

4. FilterChainProxy

Selects the matching SecurityFilterChain

5. Security filters

Authentication, authorization, CSRF, exception handling, etc.

6. DispatcherServlet → Controller → Service → Repository

Reached only if the security checks allow the request to proceed

The exact filters depend on your configuration and authentication mechanism. The diagram illustrates the general architecture, not a fixed filter sequence for every application.

## 3.2 What is a filter?

A servlet filter is code that can inspect or modify an HTTP request or response before it reaches the controller, or after downstream processing.

Think of a filter as a security guard at a building entrance.

The guard may:

* Check an identity card.

* Reject someone with invalid credentials.

* Attach information about the visitor.

* Allow the visitor to proceed to the next checkpoint.

Spring Security uses a chain of filters to implement its security behavior.

For example:

```
HTTP request
    |
    v
SecurityContext-related processing
    |
    v
Authentication mechanism
    |
    v
Exception translation
    |
    v
Authorization
    |
    v
Controller
```

This is a conceptual view. The actual filter order varies by configuration.

## 3.3 Important Spring Security components

| Component                  | Responsibility                                                                  |
| -------------------------- | ------------------------------------------------------------------------------- |
| `SecurityFilterChain`      | Defines security filters and request-matching rules                             |
| `FilterChainProxy`         | Selects and invokes the matching security filter chain                          |
| `Authentication`           | Represents authentication information and authorities                           |
| `SecurityContext`          | Holds the current authentication                                                |
| `SecurityContextHolder`    | Provides access to the current security context                                 |
| `AuthenticationManager`    | Coordinates authentication                                                      |
| `AuthenticationProvider`   | Performs a particular type of authentication                                    |
| `UserDetailsService`       | Loads user information, commonly for username/password login                    |
| `PasswordEncoder`          | Verifies passwords and hashes new passwords                                     |
| `AuthorizationManager`     | Makes authorization decisions                                                   |
| `AccessDeniedHandler`      | Handles access-denied responses in the filter chain                             |
| `AuthenticationEntryPoint` | Initiates authentication or returns an unauthenticated response, often HTTP 401 |

### The most important flow to remember

```
Request
   |
SecurityFilterChain
   |
Authentication mechanism
   |
AuthenticationManager
   |
AuthenticationProvider
   |
Validate credentials
   |
Create authenticated Authentication
   |
SecurityContext
   |
Authorization checks
   |
Controller
```

This is a simplified flow. For example, JWT resource-server authentication uses a bearer-token authentication provider rather than the username/password provider.

### Interview question: Is Spring Security based on interceptors or filters?

Answer: Its primary HTTP security enforcement mechanism in a servlet-based Spring MVC application is the servlet filter chain. Spring MVC interceptors operate later in the request lifecycle and are not a replacement for Spring Security's filter-based protection.

# 4. Authentication in Spring Security

Authentication answers one question:

> Can the application trust that this request represents the claimed identity?

## 4.1 Username and password authentication

Suppose a user submits:

http

```
POST /login
Content-Type: application/json

{
  "username": "alice",
  "password": "user-entered-password"
}
```

A typical authentication process is:

1. The application receives the credentials over HTTPS.

2. An authentication mechanism extracts them.

3. `AuthenticationManager` delegates to an appropriate `AuthenticationProvider`.

4. The provider loads user information, commonly through `UserDetailsService`.

5. `PasswordEncoder` verifies the submitted password against the stored password hash.

6. If valid, the provider returns an authenticated `Authentication` object.

7. Depending on the application's design, the authenticated identity is stored in a session or used to establish a token-based authentication flow.

Important: the raw password should never be stored in the database or logged.

## 4.2 What is `Authentication`?

In Spring Security, `Authentication` represents a security identity and its authentication state.

It can contain:

* The principal — who the user is.

* Credentials — information used to authenticate, depending on the mechanism.

* Authorities — permissions or roles.

* An authenticated flag.

Example:

Java

```
Authentication authentication =
        SecurityContextHolder.getContext()
                .getAuthentication();

String username = authentication.getName();
```

The exact principal type depends on your authentication mechanism. It may be a `UserDetails`, a JWT-backed principal, or another object.

Do not assume that the principal is always a `UserDetails`.

## 4.3 What is `SecurityContext`?

The `SecurityContext` holds the current `Authentication`.

Think of it as a small container containing the answer to:

* Who is making this request?

* What authorities do they have?

* Has this identity been authenticated?

Java

```
Authentication authentication =
        SecurityContextHolder.getContext()
                .getAuthentication();
```

In a traditional servlet application, `SecurityContextHolder` commonly uses a thread-local strategy. Spring Security also provides mechanisms for propagating context where appropriate.

Architect warning: Do not assume a thread-local security context automatically follows work submitted to an arbitrary executor, a new thread, or an asynchronous task. Context propagation must be handled deliberately.

## 4.4 What is `UserDetailsService`?

`UserDetailsService` loads user information by username.

Java

```
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username);
}
```

In a real application, you might implement it to load a user from a database.

Conceptually:

```
Username
   |
   v
UserDetailsService
   |
   v
User repository / database
   |
   v
UserDetails
```

It typically supplies the stored password hash, username, and authorities needed for password authentication.

It does not itself verify the password. That is commonly handled by the authentication provider using a `PasswordEncoder`.

Also, `UserDetailsService` is not required for every authentication style. A JWT resource server can authenticate tokens using a configured decoder and validators without loading a username/password record for every request.

## 4.5 What is `PasswordEncoder`?

Passwords must be stored using a suitable password-hashing algorithm, not plain text and not reversible encryption.

Spring Security supports password encoders such as:

* BCrypt.

* Argon2.

* PBKDF2.

* SCrypt.

Example:

Java

```
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

When registering a user:

Java

```
String encodedPassword =
        passwordEncoder.encode(rawPassword);
```

When authenticating, the framework verifies the submitted password against the stored hash.

You generally do not encode the submitted password and compare the resulting strings because secure password-hashing schemes use salts.

### Why not plain SHA-256?

A fast general-purpose hash is not ideal for password storage because attackers can test enormous numbers of guesses quickly.

Use a dedicated password-hashing algorithm with an appropriate work factor. Choose the algorithm and cost based on current guidance and production performance testing.

# 5. Authentication vs Authorization — the most common interview question

Let's use a bank application.

Alice successfully logs in. The system now knows who Alice is. But should Alice be allowed to approve a loan?

Not necessarily.

Authentication establishes Alice's identity. Authorization determines what Alice may do.

| Feature              | Authentication                              | Authorization                                             |
| -------------------- | ------------------------------------------- | --------------------------------------------------------- |
| Main question        | Who are you?                                | What can you do?                                          |
| When does it happen? | When credentials or tokens are validated    | When access to a resource or operation is evaluated       |
| Input                | Password, session, token, certificate, etc. | Identity, roles, permissions, resource attributes, policy |
| Success              | Identity is established                     | Requested action is permitted                             |
| Failure              | Often HTTP 401 for an API                   | Often HTTP 403 for an API                                 |
| Example              | Verify Alice's JWT                          | Check whether Alice can approve a loan                    |

## 5.1 HTTP 401 vs HTTP 403

This question comes up constantly in backend interviews.

* 401 Unauthorized: The request lacks acceptable authentication credentials. Despite its name, it means the caller has not successfully authenticated for the protected resource.

* 403 Forbidden: The caller is authenticated, but access is denied.

Example:

```
GET /api/admin/reports

No valid token
    → 401 Unauthorized

Valid token, but no required authority
    → 403 Forbidden
```

There are nuances: an application may deliberately return 404 to hide the existence of a resource. But 401 and 403 are the standard distinction to know.

## 5.2 What is RBAC?

RBAC means Role-Based Access Control.

Permissions are grouped into roles, and users are assigned those roles.

```
Alice → CUSTOMER
Bob   → LOAN_OFFICER
Carol → ADMIN

CUSTOMER     → VIEW_OWN_ACCOUNT
LOAN_OFFICER → REVIEW_LOAN
ADMIN        → MANAGE_USERS
```

Spring Security supports role-based checks.

Java

```
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long userId) {
    // Delete user
}
```

By convention, `hasRole("ADMIN")` checks for the authority `ROLE_ADMIN`. The `ROLE_` prefix is configurable.

For a fine-grained permission:

Java

```
@PreAuthorize("hasAuthority('LOAN_APPROVE')")
public void approveLoan(Long loanId) {
    // Approve loan
}
```

`hasAuthority` checks the exact authority string.

### Role vs authority

A role often represents a broader job function. An authority usually represents a specific permission. Spring Security represents both using `GrantedAuthority`.

For example:

```
ROLE_MANAGER
    ├── REPORT_VIEW
    ├── EMPLOYEE_VIEW
    └── EMPLOYEE_UPDATE
```

A role hierarchy can express inherited permissions, but do not assume that assigning a role automatically grants every related permission unless you configure the mapping or hierarchy.

## 5.3 Authentication is not enough: resource ownership

Consider:

http

```
GET /api/accounts/1001
```

Alice is authenticated and has `ROLE_CUSTOMER`.

She changes the URL:

http

```
GET /api/accounts/2002
```

Account 2002 belongs to Bob.

If your application checks only `ROLE_CUSTOMER`, Alice may still access Bob's account.

This is a serious authorization vulnerability commonly associated with broken object-level authorization (BOLA).

The correct approach is to verify both the required permission and the resource relationship.

Java

```
@PreAuthorize("@accountSecurity.canRead(authentication, #accountId)")
public Account getAccount(Long accountId) {
    return accountRepository.findById(accountId)
            .orElseThrow();
}
```

The `accountSecurity` bean would implement the ownership or policy check. For more complex operations, enforce the same policy in the service or data-access flow where appropriate.

Architect rule: Role checks answer “Is this type of user allowed?” Resource-level checks answer “Is this user allowed to access this specific object?”

You frequently need both.

# 6. Configure Spring Security in Spring Boot

Now let's implement a basic configuration.

Modern Spring Security applications generally use a `SecurityFilterChain` bean rather than the old `WebSecurityConfigurerAdapter` approach.

## 6.1 A basic security configuration

The following example uses a simple in-memory user store for demonstration, not production identity management.

Java

```
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(
            HttpSecurity http) throws Exception {

        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**")
                    .hasRole("ADMIN")
                .requestMatchers("/api/**")
                    .authenticated()
                .anyRequest().denyAll()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    UserDetailsService userDetailsService(
            PasswordEncoder passwordEncoder) {

        UserDetails user = User.withUsername("alice")
            .password(passwordEncoder.encode("demo-password"))
            .roles("USER")
            .build();

        return new InMemoryUserDetailsManager(user);
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Imports are omitted for readability.

### What does this configuration mean?

* `/public/**` is publicly accessible.

* `/admin/**` requires `ROLE_ADMIN`.

* `/api/**` requires authentication.

* Any other request is denied by default.

* HTTP Basic is enabled for this demonstration.

* Passwords are hashed using BCrypt.

* Method-level security is enabled.

`httpBasic()` is not a recommendation for every production API. Production applications need an authentication mechanism appropriate to the client and deployment architecture.

Also, a real deployment should not hardcode credentials or use an in-memory user store as its primary identity system.

## 6.2 Why does request-rule ordering matter?

Suppose you write:

Java

```
.requestMatchers("/api/**").authenticated()
.requestMatchers("/api/admin/**").hasRole("ADMIN")
```

The broad rule may match first, making the more specific rule unreachable for those requests.

Put more specific rules before broad rules:

Java

```
.requestMatchers("/api/admin/**").hasRole("ADMIN")
.requestMatchers("/api/**").authenticated()
```

This is a general rule for ordered request matchers within the same security chain.

## 6.3 What does `@EnableMethodSecurity` do?

It enables method-level security annotations such as:

Java

```
@PreAuthorize("hasRole('ADMIN')")
@PostAuthorize("returnObject.owner == authentication.name")
@PreFilter("filterObject.owner == authentication.name")
@PostFilter("filterObject.owner == authentication.name")
```

Use these annotations thoughtfully.

* `@PreAuthorize` checks before method execution.

* `@PostAuthorize` checks after method execution.

* `@PreFilter` filters applicable method arguments.

* `@PostFilter` filters returned collections.

`@PostAuthorize` does not undo database writes or external side effects that already happened. Do not use it as the only protection for sensitive mutations.

Method security is usually implemented using Spring AOP proxies, so self-invocation and proxy boundaries matter. Calling an annotated method from another method on the same instance may bypass the proxy-based interception.

## 6.4 What happens if I add `spring-boot-starter-security`?

Spring Boot applies security auto-configuration when its conditions are met.

For a typical servlet web application, adding the starter enables default web security and authentication behavior. It does not automatically give you your intended production access policies.

Once you define your own `SecurityFilterChain`, you control the relevant HTTP security rules.

Interview takeaway: Adding the dependency is not the same as designing the security model.

# 7. Session-based authentication vs token-based authentication

This is an important architecture decision.

## 7.1 Session-based authentication

A typical session-based flow:

```
User submits username + password
             |
             v
Server authenticates user
             |
             v
Server creates session
             |
             v
Client receives session cookie
             |
             v
Client sends cookie on later requests
             |
             v
Server looks up session and identity
```

The browser generally holds a session identifier. The server holds the associated session state, though the session may live in a distributed store rather than in local memory.

Example cookie:

http

```
Set-Cookie: JSESSIONID=...; Secure; HttpOnly; SameSite=Lax
```

Cookie attributes must be chosen for the application's requirements. The example is illustrative, not a universal configuration.

### Advantages

* Straightforward for traditional server-rendered web applications.

* Server can invalidate a session.

* The browser does not need to store an access token in JavaScript-accessible storage.

* Session state can support logout and session expiration.

### Disadvantages

* Session storage and replication may be needed across instances.

* Session expiration and distributed invalidation need design.

* Browser cookies introduce CSRF considerations for state-changing requests.

Sessions can scale well with proper architecture. They are not automatically a bad choice for microservices.

## 7.2 Token-based authentication

A common token-based architecture uses bearer access tokens.

```
Client authenticates
       |
       v
Authorization server issues access token
       |
       v
Client sends token to API
       |
       v
API validates token
       |
       v
API authorizes request
```

Example:

http

```
GET /api/accounts/1001
Authorization: Bearer eyJhbGciOi...
```

The token is a credential. Anyone who obtains a valid bearer token may be able to use it until it expires or is otherwise rejected.

### Advantages

* APIs can validate tokens without maintaining a server-side session for every request.

* Resource servers can scale independently.

* Tokens can carry scoped claims or authorities.

* Different services can validate tokens issued by a trusted identity provider.

### Disadvantages

* Revocation is harder for self-contained tokens.

* Token theft can allow an attacker to impersonate the bearer.

* Key rotation, issuer validation, audience validation, and expiration must be handled correctly.

* Authorization decisions based on stale claims may remain stale until token expiry or another revocation mechanism intervenes.

## 7.3 Comparison table

| Feature                | Session                                     | Self-contained JWT access token                            |
| ---------------------- | ------------------------------------------- | ---------------------------------------------------------- |
| Typical state          | Server-side session                         | Claims inside token                                        |
| Lookup on each request | Often session lookup                        | Often local cryptographic validation                       |
| Immediate invalidation | Relatively straightforward                  | Requires additional controls or online checks              |
| Horizontal scaling     | Shared/distributed session may be needed    | Can validate independently with shared trust configuration |
| Payload                | Stored on server                            | Claims are readable by the token holder                    |
| Browser CSRF risk      | Present when cookies are sent automatically | Depends on how the token is transported                    |
| Common use             | Traditional web apps                        | APIs and distributed systems                               |

Neither approach wins in every situation.

Architect answer: Choose based on clients, threat model, revocation requirements, deployment topology, and operational complexity—not because tokens are supposedly always more scalable.

# 8. JWT explained from zero

JWT stands for JSON Web Token.

A JWT is a compact way to represent claims as a JSON object, protected by a digital signature or message authentication code, depending on the algorithm.

For interview purposes, understand its three parts:

```
header.payload.signature
```

A typical JWT looks like:

```
xxxxx.yyyyy.zzzzz
```

These are Base64URL-encoded components.

## 8.1 The three parts of a JWT

Part 1

Header

Describes token metadata, commonly the signing algorithm and token type.

JSON

```
{
  "alg": "RS256",
  "typ": "JWT"
}
```

Part 2

Payload

Contains claims about the subject and token.

JSON

```
{
  "sub": "alice",
  "iss": "https://auth.example.com",
  "aud": "accounts-api",
  "exp": 1791655200,
  "scope": "accounts.read"
}
```

Part 3

Signature

Protects the signed content against undetected modification. The exact signing or verification process depends on the configured algorithm.

The claims above are illustrative. A real application must validate their values and use the appropriate expected issuer and audience.

## 8.2 Is a JWT encrypted?

Usually, no.

A signed JWT is generally readable by anyone who possesses it. Base64URL encoding is not encryption.

Therefore, never put sensitive information such as passwords, secret keys, or confidential personal data in a JWT merely because it has a signature.

There are also encrypted JWT formats, but ordinary signed access tokens should be treated as readable.

## 8.3 What does the JWT signature do?

Imagine Alice receives a signed token containing:

JSON

```
{
  "sub": "alice",
  "role": "USER"
}
```

An attacker modifies it:

JSON

```
{
  "sub": "alice",
  "role": "ADMIN"
}
```

If the API validates the signature correctly, the altered token will fail verification.

A signature provides integrity and issuer-authenticity assurance under the applicable key and trust assumptions. It does not conceal the payload.

## 8.4 Important JWT claims

| Claim   | Meaning            | Why it matters                                |
| ------- | ------------------ | --------------------------------------------- |
| `iss`   | Issuer             | Who issued the token?                         |
| `sub`   | Subject            | Who or what is the token about?               |
| `aud`   | Audience           | Which recipient is the token intended for?    |
| `exp`   | Expiration time    | When does the token stop being valid?         |
| `nbf`   | Not before         | When does the token become valid?             |
| `iat`   | Issued at          | When was the token issued?                    |
| `jti`   | Token identifier   | Can help identify or track a particular token |
| `scope` | Common scope claim | What delegated access is granted?             |

Not every JWT contains every claim, and not every token uses the same claim naming conventions.

### A crucial interview point

A valid signature does not automatically mean a token should be accepted.

A resource server must also apply the required checks, including:

* Trusted signing key and allowed algorithm.

* Expected issuer.

* Expected audience.

* Expiration and applicable time constraints.

* Appropriate token type and intended use.

* Required scopes or authorities.

A token signed by a trusted issuer but intended for another API should not automatically be accepted by your API.

# 9. How JWT authentication works in Spring Boot

Let's build the most common production scenario: a Spring Boot REST API secured with JWT access tokens.

## 9.1 The complete flow

Step 1: Client logs in

Credentials are sent to the identity provider over HTTPS.

Step 2: Identity provider authenticates the user

Credentials, MFA, or another supported mechanism is verified.

Step 3: Access token is issued

The token contains claims and has a defined lifetime and intended audience.

Step 4: Client calls the resource server

Authorization: Bearer access_token

Step 5: Spring Security validates the token

Signature, issuer, audience, expiry and other configured checks.

Step 6: Authorization is evaluated

Scopes, roles, authorities and resource-level policies.

Step 7: Controller executes

Only if the applicable security checks pass.

A key distinction: the resource server usually does not need to contact the identity provider for every JWT request. It can validate the token locally using trusted verification keys and configured validators.

That reduces network dependencies, but it also means a revoked token may continue to work until expiration unless you implement additional revocation controls.

## 9.2 Access token vs refresh token

These are not the same thing.

Access token

* Sent to a resource server.

* Grants access according to its claims and the server's policies.

* Usually has a relatively short lifetime.

Refresh token

* Used with an authorization server to obtain new access tokens.

* Usually more sensitive and potentially longer-lived.

* Should be sent only to the appropriate token endpoint, not to ordinary resource APIs.

A refresh token is not simply another access token.

For public clients such as mobile apps, refresh-token rotation and reuse detection are important security measures where supported. The exact policy depends on the identity provider and protocol flow.

## 9.3 What happens when a JWT expires?

Suppose the token expires at 10:30 AM and the user sends a request at 10:31 AM.

The resource server should reject it if its expiry validator is correctly configured.

Typical flow:

```
Expired access token
       |
       v
Resource server rejects request
       |
       v
Client attempts refresh, if supported
       |
       +---- Refresh accepted → New access token
       |
       +---- Refresh rejected → User must authenticate again
```

Do not solve token expiration by making access tokens valid indefinitely.

## 9.4 How do you implement JWT validation in Spring Boot?

The recommended standard approach for a Spring Security API is generally to configure it as an OAuth 2.0 Resource Server rather than writing a custom JWT filter from scratch.

Dependency:

XML

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

Example configuration:

Java

```
@Configuration
@EnableWebSecurity
public class ApiSecurityConfig {

    @Bean
    SecurityFilterChain apiSecurityFilterChain(
            HttpSecurity http) throws Exception {

        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(
                    SessionCreationPolicy.STATELESS
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/api/admin/**")
                    .hasRole("ADMIN")
                .requestMatchers("/api/**")
                    .authenticated()
                .anyRequest().denyAll()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            )
            .build();
    }
}
```

Imports are omitted.

This is a starting point for a bearer-token API that does not use cookies for authentication. Disabling CSRF is appropriate only when the API's authentication arrangement does not rely on automatically attached browser credentials, such as session cookies. Do not copy that line blindly into a cookie-authenticated application.

Configure a trusted issuer in your application properties:

YAML

```
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

The issuer URI must match your actual identity provider. This configuration enables Spring Security's issuer-based JWT validation and discovery behavior, subject to the provider's metadata and key availability.

If your API requires a specific audience, configure and test audience validation as well; do not assume that issuer validation alone enforces the intended audience.

### What does `STATELESS` mean?

Java

```
SessionCreationPolicy.STATELESS
```

This tells Spring Security not to create an HTTP session to persist the security context between requests.

Each request must supply its own acceptable authentication credentials.

It does not mean the entire application has no state. Your application can still use databases, Redis, caches, or other forms of application state.

## 9.5 What happens internally during JWT authentication?

A simplified Spring Security resource-server flow is:

```
HTTP request
    |
    v
BearerTokenAuthenticationFilter
    |
    v
BearerTokenAuthenticationToken
    |
    v
AuthenticationManager
    |
    v
JwtAuthenticationProvider
    |
    v
JwtDecoder
    |
    +--> Verify signature
    +--> Validate configured claims
    +--> Validate time constraints
    |
    v
Convert JWT claims to authorities
    |
    v
Create authenticated Authentication
    |
    v
Store it in SecurityContext
    |
    v
Authorize the request
```

Spring Security uses a `JwtAuthenticationToken` for the authenticated JWT principal in the typical resource-server configuration.

The token's claims are converted into authorities according to configured conventions and converters. For example, OAuth scopes commonly become authorities with the prefix `SCOPE_`.

Java

```
@PreAuthorize("hasAuthority('SCOPE_accounts.read')")
public Account getAccount(Long accountId) {
    // Fetch account
}
```

This only works if the token's scopes and the resource-server authority mapping are configured as expected.

# 10. JWT security mistakes that senior engineers must recognize

## Mistake 1: Trusting the token because it is well-formed

A token can have three valid-looking segments and still be malicious or expired.

Correct approach: Cryptographically validate it and apply the appropriate issuer, audience, expiry, and token-use checks.

## Mistake 2: Using `alg: none` or trusting arbitrary algorithms

Never let an untrusted token dictate an unrestricted verification policy.

Correct approach: Configure trusted algorithms and key sources; use a well-maintained library and standard resource-server support.

## Mistake 3: Storing sensitive information in JWT claims

Signed JWT payloads are readable.

Correct approach: Put only the claims required for the token's purpose in it. Keep secrets and sensitive business information server-side.

## Mistake 4: Giving tokens excessively long lifetimes

A stolen bearer token remains useful for longer.

Correct approach: Use an appropriate access-token lifetime, secure storage, refresh-token controls, and a revocation strategy matching your risk.

## Mistake 5: Assuming logout instantly revokes a JWT

Deleting a token from the client does not automatically invalidate a copy held by an attacker.

Correct approach: Use short-lived access tokens and consider token revocation, denylisting, token-version checks, or other centralized controls when immediate invalidation is required.

## Mistake 6: Putting too much authorization logic inside the token

Imagine a token says Alice is an administrator. An administrator revokes her access, but her token remains valid.

The token may still carry the old claim.

Correct approach: Decide which permissions can safely remain valid until token expiry and which require a fresh server-side authorization decision.

## Mistake 7: Storing bearer tokens in browser local storage without considering XSS

JavaScript-accessible storage can expose tokens to injected scripts if an XSS vulnerability occurs.

Correct approach: Evaluate the threat model carefully. For browser applications, a backend-for-frontend (BFF) using secure cookies can reduce exposure of OAuth tokens to browser JavaScript. Cookie-based approaches must address CSRF.

# 11. OAuth 2.0 explained simply

OAuth 2.0 is a framework for delegated authorization.

Imagine Alice uses a finance application that wants to access her transactions from another service.

The finance application should not ask Alice to hand over her bank password. Instead, it should obtain a limited authorization grant and an access token from a trusted authorization server.

The token can grant access to specific resources and scopes.

## 11.1 The four main OAuth roles

| Role                 | Meaning                                               | Example           |
| -------------------- | ----------------------------------------------------- | ----------------- |
| Resource Owner       | Entity that can authorize access                      | Alice             |
| Client               | Application requesting access                         | Finance dashboard |
| Authorization Server | Issues tokens after processing an authorization grant | Identity platform |
| Resource Server      | Hosts the protected API                               | Transactions API  |

An authorization server and a resource server may be operated by the same organization, but they have distinct logical responsibilities.

## 11.2 How OAuth 2.0 works

```
Alice
  |
  v
Finance application
  |
  | Requests authorization
  v
Authorization server
  |
  | Alice authenticates and grants access
  v
Authorization server issues access token
  |
  v
Finance application calls transactions API
  |
  v
Resource server validates token and permissions
  |
  v
Allowed transactions are returned
```

OAuth does not require every access token to be a JWT. Access tokens can be opaque, with the resource server validating them through an introspection endpoint or another provider-specific mechanism.

## 11.3 What is a scope?

A scope represents a requested or granted category of access.

Examples:

```
accounts.read
accounts.write
transactions.read
transactions.export
```

A client with `transactions.read` should not automatically receive `transactions.export`.

However, the resource server must enforce the scope. Merely including a scope in a token does not cause an API to enforce it automatically.

### Scope vs role

* A role often describes a user's application-level responsibility.

* A scope commonly describes access granted to a client or token under an OAuth authorization flow.

They can overlap in implementation, but they are not identical concepts.

# 12. OAuth 2.0 grant types you must know

OAuth 2.0 uses different authorization grants for different client types and situations.

## 12.1 Authorization Code Flow with PKCE

This is the preferred general pattern for modern user-facing applications, including public clients such as mobile apps and single-page applications.

PKCE stands for Proof Key for Code Exchange.

Why is it needed?

A stolen authorization code should not be sufficient to redeem the code when the client has correctly implemented PKCE.

Simplified flow:

```
1. Client creates code verifier and code challenge.
2. Client redirects user to authorization server.
3. User authenticates and approves requested access.
4. Authorization server redirects back with an authorization code.
5. Client redeems code using the code verifier.
6. Authorization server validates PKCE and issues tokens.
```

Important details:

* The client creates a secret random code verifier for the transaction.

* It sends a derived challenge during authorization.

* During token exchange, it sends the verifier.

* The authorization server checks that the verifier matches the challenge.

PKCE is not a replacement for HTTPS, redirect URI validation, or normal OAuth security checks.

## 12.2 Client Credentials Flow

Use this for machine-to-machine access when a client acts on its own behalf rather than signing in a human user.

Example:

```
Billing service
     |
     v
Authorization server
     |
     v
Access token for Billing service
     |
     v
Invoice API
```

A service can authenticate using an appropriate client credential, such as a secret or private-key-based mechanism, and obtain a token for permitted scopes.

Prefer stronger workload identity mechanisms where available rather than distributing long-lived client secrets everywhere.

## 12.3 Resource Owner Password Credentials grant

This older grant allows a client to collect a user's password and exchange it for a token.

Do not use this for new systems. Modern OAuth security guidance says this grant must not be used because it exposes user credentials to the client and does not support modern authentication patterns well.

## 12.4 Implicit grant

The old implicit flow issued tokens directly through the browser authorization response.

Do not use it for new applications. Authorization Code with PKCE is the modern alternative for public clients.

## 12.5 Refresh token grant

A client can present an eligible refresh token to obtain a new access token.

The authorization server checks that the refresh token remains valid and permitted. Refresh tokens are not intended for resource APIs.

## 12.6 Which flow should you choose?

| Scenario                                              | Common choice                             |
| ----------------------------------------------------- | ----------------------------------------- |
| Web application acting for a signed-in user           | Authorization Code; PKCE is recommended   |
| Mobile application                                    | Authorization Code with PKCE              |
| Single-page application                               | Authorization Code with PKCE              |
| Backend service calling another service as itself     | Client Credentials                        |
| Old design that collects the user's password directly | Avoid Resource Owner Password Credentials |
| New browser or mobile application using implicit flow | Prefer Authorization Code with PKCE       |

The exact design depends on the client type, identity provider, and whether the backend acts on behalf of a user or as its own workload.

# 13. OpenID Connect (OIDC): OAuth's identity layer

This is a favorite follow-up question:

If OAuth 2.0 is about authorization, how does “Login with Google” work?

The answer is OpenID Connect.

OIDC adds a standardized authentication layer on top of OAuth 2.0.

It introduces the ID token, which communicates information about the authentication event and the authenticated subject to the client.

## 13.1 ID token vs access token

| Feature                                     | ID token                                                           | Access token                                              |
| ------------------------------------------- | ------------------------------------------------------------------ | --------------------------------------------------------- |
| Purpose                                     | Communicates authentication and identity information to the client | Grants access to a resource                               |
| Intended recipient                          | OIDC client                                                        | Resource server or API                                    |
| Typical contents                            | Subject, issuer, audience, authentication-related claims           | Scopes, audience, subject, or other access-related claims |
| Should an API accept it as an access token? | No, not by default                                                 | Yes, if valid for that API                                |
| Format                                      | JWT in standard OIDC flows                                         | JWT or opaque token                                       |

Imagine a user signs in to a web application through an identity provider.

* The ID token helps the client establish who authenticated.

* The access token is used to call the protected API.

* The API validates the access token according to its own trust and authorization policy.

An ID token and access token may both be JWTs, but they have different purposes. Do not interchange them.

## 13.2 What is the difference between OAuth and OIDC?

Interview answer:

OAuth 2.0 provides delegated authorization to access protected resources. OpenID Connect builds on OAuth 2.0 to standardize authentication and identity information, including the ID token and user information mechanisms.

# 14. Spring Security as an OAuth 2.0 client

So far, we've discussed securing an API that receives access tokens. What if your Spring Boot application needs to sign users in through Google, Microsoft Entra ID, Okta, or another identity provider?

Then your application can act as an OAuth 2.0 client using Spring Security.

For example, add:

XML

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

Configure an OIDC provider in `application.yml`:

YAML

```
spring:
  security:
    oauth2:
      client:
        registration:
          company:
            provider: company-idp
            client-id: ${OAUTH_CLIENT_ID}
            client-secret: ${OAUTH_CLIENT_SECRET}
            scope:
              - openid
              - profile
              - email
        provider:
          company-idp:
            issuer-uri: https://identity.example.com
```

This is illustrative configuration. Actual registration and provider settings depend on the identity provider, supported endpoints, client type, and redirect URI.

The `openid` scope signals an OIDC authentication request.

A simple browser-login configuration:

Java

```
@Bean
SecurityFilterChain webSecurity(
        HttpSecurity http) throws Exception {

    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(Customizer.withDefaults())
        .build();
}
```

Spring Security manages the OAuth login flow, subject to your configuration.

For a normal browser login, the resulting application often uses an HTTP session to maintain the authenticated user state. OAuth login does not automatically mean your application becomes stateless.

## 14.1 OAuth Client vs Resource Server

This distinction matters in microservice architectures.

| Spring Security capability    | What it does                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------ |
| OAuth 2.0 Client              | Obtains or uses tokens to access protected resources                           |
| OAuth 2.0 Resource Server     | Protects APIs by validating incoming access tokens                             |
| OAuth 2.0 Login / OIDC Client | Signs users in through an external identity provider                           |
| Authorization Server          | Issues tokens and manages authorization grants; this is a separate server role |

An application can be both an OAuth client and a resource server, but each responsibility must be configured appropriately.

If you need to build your own authorization server, consider a dedicated implementation such as Spring Authorization Server rather than attempting to turn ordinary JWT validation into an authorization server.

# 15. How to secure microservices using Spring Security

Now let's think like a Technical Architect.

Suppose you are designing a banking platform:

* API Gateway

* Customer Service

* Account Service

* Payment Service

* Notification Service

* Identity Provider

How should security work?

## 15.1 Recommended high-level architecture

Web / Mobile Client

Identity Provider

User authentication, MFA, token issuance

API Gateway

Routing, rate limits, edge controls, optional token validation

Backend microservices

Customer Service

Account Service

Payment Service

Notification Service

Each service enforces its own required security and business authorization rules.

## 15.2 Should the API Gateway validate JWTs?

Often, yes. The gateway can reject invalid requests early, reducing unnecessary downstream traffic.

But should only the gateway validate the token?

Not necessarily.

If backend services can be reached directly, or if you need independent security boundaries, each resource server should validate the appropriate incoming token or receive authenticated identity through a rigorously protected, trusted architecture.

A gateway is not a substitute for downstream authorization.

A common defense-in-depth design is:

1. Gateway performs edge validation and traffic controls.

2. Each backend service validates its own trust boundary.

3. Each service enforces its own scopes and business permissions.

4. Network policies prevent unauthorized direct access.

5. Service-to-service communication uses authenticated workload identities and encrypted transport where required.

Avoid blindly forwarding a user's access token to every downstream service. Tokens should have appropriate audiences and scopes, and token exchange or a separate service identity may be necessary.

## 15.3 User token vs service token

Suppose the Account Service calls the Notification Service.

There are two distinct situations.

Situation A: A user-triggered action

The downstream operation may need to retain the user's identity and authorization context. Depending on the design, the original token, a suitably scoped token, or a token obtained through token exchange may be appropriate.

Situation B: A background service operation

A scheduled job sends notifications without an active user request. The workload should typically authenticate using its own service identity and receive only the permissions it requires.

Do not treat a service identity as an administrator just because it is internal.

## 15.4 What is the principle of least privilege?

Every user, application, service account, and token should have only the permissions necessary to perform its job.

For example:

```
Notification Service
    ├── Can publish notification events
    ├── Can read notification templates
    └── Cannot approve payments
```

A good architecture avoids broad shared credentials and unrestricted service-to-service access.

## 15.5 How do you handle JWT key rotation?

For asymmetric signing algorithms, an identity provider may sign tokens with a private key and publish corresponding public keys through a trusted JWKS endpoint.

A sensible rotation process is:

1. Publish the new public key.

2. Begin signing new tokens with the corresponding private key.

3. Allow validators to discover and cache the updated keys.

4. Retain the old public key while tokens signed with it may still be valid.

5. Remove the old key when it is safe to do so.

Exact timing depends on token lifetimes, cache behavior, and provider capabilities.

Spring Security's resource-server infrastructure supports public-key discovery and key rotation patterns. Your operational design still needs to account for caching, availability, and key compromise.

If a private signing key is compromised, normal graceful rotation may not be sufficient. You may need an emergency trust change and token-rejection strategy.

# 16. CSRF and CORS: two concepts interviewers love to mix up

They are different problems.

## 16.1 What is CSRF?

CSRF stands for Cross-Site Request Forgery.

Imagine Alice is logged in to her bank website. Her browser holds a session cookie.

She visits a malicious website in another tab. That website attempts to make the browser submit a request to the bank.

If the browser automatically attaches the bank's cookie and the bank does not have appropriate CSRF protection, an unwanted action may succeed.

```
Alice logs in to bank
        |
        v
Browser stores session cookie
        |
        v
Alice visits malicious site
        |
        v
Browser is tricked into sending request
        |
        v
Bank receives cookie automatically
```

The attacker may not be able to read the bank's response, but that does not necessarily prevent the attacker from causing an unwanted state change.

### When is CSRF protection important?

It is particularly important when a browser automatically sends credentials, such as session cookies or authentication cookies.

For cookie-authenticated applications, Spring Security's CSRF protection should generally remain enabled for state-changing requests, with the token handled correctly by the client.

For a backend API that exclusively authenticates using an explicitly supplied bearer token in the `Authorization` header and does not rely on browser-automatically attached authentication credentials, CSRF risk is materially different. Disabling CSRF may be appropriate for that API configuration.

Do not disable CSRF simply because an application is called a REST API.

## 16.2 What is CORS?

CORS stands for Cross-Origin Resource Sharing.

Suppose your frontend runs at:

```
https://app.example.com
```

Your backend API runs at:

```
https://api.example.com
```

These are different origins because the hostnames differ.

The browser enforces the same-origin policy. CORS allows a server to declare which cross-origin browser requests are permitted.

Example:

Java

```
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();

    config.setAllowedOrigins(
        List.of("https://app.example.com")
    );

    config.setAllowedMethods(
        List.of("GET", "POST", "PUT", "DELETE")
    );

    config.setAllowedHeaders(
        List.of("Authorization", "Content-Type")
    );

    UrlBasedCorsConfigurationSource source =
        new UrlBasedCorsConfigurationSource();

    source.registerCorsConfiguration("/**", config);

    return source;
}
```

Then enable CORS in the Spring Security chain:

Java

```
http.cors(Customizer.withDefaults());
```

Imports are omitted.

This example is a starting point. Production CORS configuration must match the actual frontend, HTTP methods, headers, credential requirements, and deployment setup.

### Important CORS rules

* Do not allow every origin in a sensitive production application without a clear reason.

* Credentialed CORS requests require carefully configured origins; `Access-Control-Allow-Origin: *` cannot be combined with credentialed browser access.

* CORS is not authentication.

* CORS is not authorization.

* CORS does not prevent non-browser clients from calling your API.

* CORS does not replace CSRF protection.

## 16.3 CSRF vs CORS

| Feature                        | CSRF                                                | CORS                                                    |
| ------------------------------ | --------------------------------------------------- | ------------------------------------------------------- |
| Main problem                   | Unwanted authenticated actions                      | Browser restrictions on cross-origin access             |
| Typical mechanism              | CSRF token and related defenses                     | Server response headers and browser enforcement         |
| Relevant to cookies?           | Yes, especially automatically attached auth cookies | Yes, when cross-origin browser requests use credentials |
| Does it authenticate the user? | No                                                  | No                                                      |
| Does it grant API permissions? | No                                                  | No                                                      |

# 17. Spring Security filter chain: deeper interview questions

At an architect level, interviewers may ask how authentication failures are handled and how security configuration is selected.

## 17.1 What is `SecurityFilterChain`?

A `SecurityFilterChain` defines a set of security filters and the requests for which they apply.

You can define multiple chains for different security boundaries.

For example:

* One chain for stateless `/api/**` endpoints.

* Another chain for browser login and session-based pages.

* A separate chain for specialized endpoints with distinct requirements.

Spring Security selects the first matching chain in its configured order. If multiple chains exist, use `@Order` and `securityMatcher(...)` deliberately.

Example:

Java

```
@Bean
@Order(1)
SecurityFilterChain apiChain(HttpSecurity http)
        throws Exception {

    return http
        .securityMatcher("/api/**")
        .sessionManagement(session -> session
            .sessionCreationPolicy(
                SessionCreationPolicy.STATELESS
            )
        )
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(Customizer.withDefaults())
        )
        .build();
}
```

A second chain can handle browser endpoints, but it must be configured separately. Do not assume that a rule in one chain automatically applies to another.

If no chain matches a request, the request may not pass through Spring Security's configured chains. Design the matchers to avoid accidentally leaving endpoints unprotected.

## 17.2 What is the difference between `AuthenticationManager` and `AuthenticationProvider`?

Think of the manager as a coordinator and the provider as a specialist.

* `AuthenticationManager` is the main abstraction for authenticating an `Authentication` request.

* `ProviderManager` is a common implementation that delegates to configured providers.

* `AuthenticationProvider` handles a particular authentication mechanism.

Examples of authentication mechanisms include username/password authentication and JWT bearer-token authentication.

You do not need to create a custom provider for every application. Prefer built-in implementations where possible.

## 17.3 What is `AuthenticationEntryPoint`?

It handles the start of an authentication process when the request needs authentication, often by returning a 401 response for an API.

For example:

http

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
```

The precise challenge header depends on the authentication mechanism.

## 17.4 What is `AccessDeniedHandler`?

It handles an access-denied situation within the filter-based security flow, typically returning a 403 response.

Example:

```
Authenticated user
       |
       v
Attempts ADMIN endpoint
       |
       v
Authorization check fails
       |
       v
AccessDeniedHandler
       |
       v
403 Forbidden
```

## 17.5 What is `ExceptionTranslationFilter`?

It bridges certain security exceptions raised downstream in the filter chain into appropriate HTTP security behavior.

Broadly:

* An unauthenticated request that requires authentication may invoke the `AuthenticationEntryPoint`.

* An authenticated request denied access may invoke the `AccessDeniedHandler`.

It does not handle every exception in the entire application, and controller exceptions may be handled by Spring MVC's exception-resolution mechanisms instead.

## 17.6 Why might a controller not be reached?

Possible reasons include:

* A filter rejects the request.

* The token is invalid or expired.

* The URL is not permitted.

* A required role or scope is missing.

* CSRF validation fails.

* A request matcher selects a different security chain than expected.

* A custom filter terminates processing.

When debugging, inspect the security logs and the selected filter chain before assuming the controller has a problem.

# 18. Advanced authorization: beyond roles

Basic role checks are useful, but production authorization often needs more context.

## 18.1 ABAC — Attribute-Based Access Control

ABAC makes decisions using attributes about the subject, resource, action, or environment.

Example:

A loan officer may approve a loan only when:

* The loan is within their approval limit.

* The customer belongs to their assigned region.

* The loan is in an approvable state.

* The request satisfies relevant business policy.

  Subject attributes
  +
  Resource attributes
  +
  Action
  +
  Environment
  |
  v
  Authorization decision

Spring Security's authorization mechanisms can be integrated with application-specific policy checks, although a complex enterprise policy engine may be appropriate for some systems.

## 18.2 Where should authorization logic live?

A common mistake is to put every security check in the controller.

For example:

Java

```
@GetMapping("/accounts/{id}")
public Account getAccount(@PathVariable Long id) {
    // Controller
}
```

The controller should not be the only place enforcing sensitive business access rules.

A stronger design combines:

* HTTP-level request authorization.

* Method-level authorization where appropriate.

* Domain-level ownership and business-rule checks.

* Data-access constraints that limit access to authorized records.

For example, instead of loading any account and checking ownership afterward, you can query using both the account ID and the user's permitted customer ID:

Java

```
return accountRepository
    .findByIdAndCustomerId(accountId, customerId)
    .orElseThrow(AccountNotFoundException::new);
```

The `customerId` must come from a trusted, authenticated identity-to-customer mapping, not blindly from an arbitrary request parameter.

This approach can reduce accidental data exposure and enforce the ownership boundary closer to the data.

## 18.3 What is the confused deputy problem?

A service has permission to perform an operation. A caller tricks it into using that permission on the caller's behalf without the proper authorization.

Example:

* The Payment Service can initiate payments.

* A low-privilege user can call an endpoint that indirectly invokes it.

* The service trusts the internal request without checking the user's permitted actions.

The solution is not merely to secure network traffic. The service must validate the authorization context and enforce the relevant business rules.

## 18.4 Multi-tenant authorization

Suppose your SaaS platform serves 500 companies.

A user from Company A must never access Company B's data.

Checking `ROLE_USER` is not enough.

You must establish and enforce the tenant boundary throughout the request:

```
Authenticated identity
       |
       v
Trusted tenant mapping
       |
       v
Tenant-aware authorization
       |
       v
Tenant-scoped database query
```

Use tenant-aware repository queries, and consider database-level protections such as row-level security where appropriate.

Architect rule: Tenant isolation must not depend on developers remembering to filter every query manually without a systematic enforcement mechanism.

# 19. Other security mechanisms you should know

Spring Security is not just JWT and OAuth.

## 19.1 Password storage

Use a dedicated password-hashing algorithm such as BCrypt or Argon2, with parameters chosen for your environment.

Never store plaintext passwords or log submitted credentials.

When migrating from an older password-hashing scheme, support a safe migration path and consider rehashing after successful authentication.

## 19.2 Multi-factor authentication (MFA)

MFA requires additional evidence beyond a password.

Examples include an authenticator-generated code, a hardware security key, or a platform passkey.

In many architectures, the identity provider manages MFA. Spring Security then consumes the resulting authenticated identity and claims. If the application has special step-up requirements, it must enforce those requirements using trustworthy authentication-context information.

## 19.3 Brute-force protection

An attacker may repeatedly guess passwords.

Controls can include:

* Rate limiting.

* Temporary throttling or lockout policies.

* Credential-stuffing detection.

* MFA.

* Monitoring and alerting.

Be careful with permanent account lockouts: an attacker might deliberately lock other users out of their accounts.

## 19.4 Security headers

Useful browser-facing defenses include:

* Content Security Policy (CSP).

* `X-Content-Type-Options: nosniff`.

* Frame restrictions using CSP `frame-ancestors` or suitable alternatives.

* Appropriate HSTS configuration when HTTPS is correctly deployed.

Spring Security provides security-header support, but headers must be configured to match your deployment and application behavior.

## 19.5 TLS / HTTPS

HTTPS protects credentials and tokens in transit.

For a production system, enforce HTTPS at the appropriate edge and ensure proxy or load-balancer arrangements do not cause the application to make unsafe assumptions about the original request scheme.

HTTPS does not prevent authorization bugs, token theft from compromised clients, or all forms of application attack.

## 19.6 Audit logging

For sensitive actions, record enough information to investigate what happened:

* Authenticated subject or service identity.

* Action performed.

* Target resource.

* Timestamp.

* Outcome.

* Correlation or request ID.

Do not log passwords, full bearer tokens, refresh tokens, or other secrets. Sanitize untrusted log values and apply appropriate access and retention controls to audit data.

# 20. Common production scenarios and architect-level decisions

Let's apply these concepts to real system-design situations.

## Scenario 1: A banking API must serve millions of requests

Question: Would you call the identity provider for every request?

Not necessarily.

With JWT access tokens, a resource server can validate signatures locally using trusted public keys and enforce claims and permissions without an identity-provider call on every request.

Architect considerations:

* Cache and refresh signing keys safely.

* Validate issuer, audience, expiry, and required scopes.

* Define a revocation strategy.

* Avoid expensive remote authorization calls on the critical path unless required.

* Measure cryptographic validation and authorization latency.

* Apply rate limiting and request-size limits.

* Avoid putting sensitive or rapidly changing permissions into long-lived tokens.

A JWT design does not eliminate all state or all network calls. It eliminates the need for a per-request identity-provider call in the common local-validation pattern.

## Scenario 2: An employee is terminated

Question: The employee's JWT expires in 30 minutes. How do you prevent further access immediately?

First, recognize the problem: local validation alone may continue accepting the token until expiry.

Possible approaches:

1. Short access-token lifetimes.

2. Centralized token revocation or introspection where appropriate.

3. A denylist keyed by token identifier or a suitable token fingerprint.

4. User or session version checks against trusted server-side state.

5. Immediate key or issuer trust changes in an emergency, if warranted.

Each approach has costs. A denylist adds state and potentially a lookup; a per-request user-version lookup introduces a dependency. Choose based on the actual revocation requirement.

## Scenario 3: Payment API requires strong protection

The design should include more than `hasRole("USER")`.

Consider:

* User authentication with appropriate assurance.

* Fine-grained payment permissions.

* Ownership and account-level authorization.

* Payment limits and state-transition validation.

* Idempotency controls to avoid duplicate processing.

* Strong service-to-service authentication.

* Audit trails and alerting.

* Step-up authentication for high-risk actions, if required.

Security must be enforced in the business operation, not just the URL.

## Scenario 4: A frontend and API are deployed on different domains

Check:

* CORS allowed origins and methods.

* Whether credentials are cookies or bearer tokens.

* CSRF defenses for cookie-authenticated operations.

* Secure cookie attributes.

* HTTPS.

* Token storage and XSS exposure.

* The behavior of browser preflight requests.

Do not “fix CORS” by allowing every origin or “fix CSRF” by disabling all protection without understanding the authentication design.

## Scenario 5: The identity provider is temporarily unavailable

The answer depends on what the API needs to do.

With a locally validated JWT and cached signing keys, existing tokens may continue to validate even if the provider is unavailable.

However:

* New logins may fail.

* Refresh-token exchanges may fail.

* Key discovery or refresh may be affected.

* Introspection-based validation may fail.

Design explicit failure behavior. Never accept invalid or unverifiable tokens merely because the identity provider is down.

## Scenario 6: One microservice needs to call another securely

Use workload identity and appropriately scoped credentials. Depending on the environment, use OAuth client credentials, workload identity federation, mTLS, or a combination.

Verify the receiving service's identity and permissions. Avoid shared static credentials across every service.

## Scenario 7: A user has the right role but accesses another tenant's data

This is a resource-level authorization failure.

Use tenant-aware authorization, ownership checks, scoped repository access, and appropriate isolation controls. Add tests that attempt cross-tenant access explicitly.

## Scenario 8: A token has the correct signature but is intended for a different API

Reject it if the audience is not valid for the receiving API.

Signature validation establishes integrity under a trusted key. It does not establish that the token is intended for every resource server.

# 21. How to test Spring Security properly

A security implementation is not complete just because the application starts successfully.

You must test both allowed and denied requests.

## 21.1 Essential test cases

For an endpoint requiring an administrator role, test:

| Test case                             | Expected result                               |
| ------------------------------------- | --------------------------------------------- |
| No token                              | 401 or configured authentication challenge    |
| Malformed token                       | Rejected                                      |
| Expired token                         | Rejected                                      |
| Invalid signature                     | Rejected                                      |
| Wrong issuer                          | Rejected                                      |
| Wrong audience                        | Rejected when audience validation is required |
| Valid token, insufficient authority   | 403                                           |
| Valid token with required authority   | Success, assuming other business checks pass  |
| User accesses another user's record   | Denied                                        |
| User accesses another tenant's record | Denied                                        |

The exact response can vary based on endpoint policy and exception handling. Test the security behavior, not only the status code.

## 21.2 Spring Security test support

Spring Security provides testing support through `spring-security-test`.

Add the test dependency:

XML

```
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

You can test authorization using MockMvc:

Java

```
@Test
@WithMockUser(roles = "ADMIN")
void adminCanAccessAdminEndpoint() throws Exception {
    mockMvc.perform(get("/api/admin/reports"))
        .andExpect(status().isOk());
}
```

Imports and test setup are omitted.

For a user with insufficient permissions:

Java

```
@Test
@WithMockUser(roles = "USER")
void userCannotAccessAdminEndpoint() throws Exception {
    mockMvc.perform(get("/api/admin/reports"))
        .andExpect(status().isForbidden());
}
```

These tests simulate an authenticated identity. They do not prove that JWT signature validation, issuer validation, or real OAuth login works correctly.

For JWT resource-server tests, you can use Spring Security's JWT request post-processor to simulate a JWT-authenticated request. Separately, add integration tests that exercise actual decoder validation and relevant token-validation failures.

Architect rule: Test authentication, authorization, and business-level resource access as separate concerns.

# 22. Common Spring Security mistakes

These are the mistakes worth remembering before an interview.

| Mistake                                        | Why it is dangerous                                          | Better approach                                      |
| ---------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| Using `permitAll()` on sensitive endpoints     | Removes the intended access restriction                      | Permit only deliberately public endpoints            |
| Disabling CSRF everywhere                      | Can expose cookie-authenticated operations to CSRF           | Configure CSRF based on credential transport         |
| Trusting any valid JWT                         | A token may be intended for another API                      | Validate issuer, audience and required claims        |
| Treating a JWT as encrypted                    | Claims may expose sensitive data                             | Never place secrets in readable token payloads       |
| Checking only roles                            | Users may access other users' resources                      | Enforce ownership and tenant-level access            |
| Trusting the API gateway alone                 | Backend access paths may bypass gateway controls             | Protect downstream trust boundaries                  |
| Using long-lived access tokens                 | Increases exposure after token theft                         | Use appropriate expiry and revocation controls       |
| Storing raw passwords                          | A breach exposes user credentials                            | Use a dedicated password-hashing algorithm           |
| Writing custom JWT filters unnecessarily       | Can introduce validation and ordering bugs                   | Prefer Spring Security Resource Server               |
| Returning stack traces in API errors           | Leaks implementation details                                 | Return safe error responses and log details securely |
| Allowing every CORS origin                     | Expands browser access unexpectedly                          | Configure explicit trusted origins                   |
| Assuming `@PreAuthorize` always executes       | Proxy boundaries and self-invocation can bypass interception | Understand AOP and test method-security behavior     |
| Giving every service administrator permissions | Increases the impact of compromise                           | Apply least privilege and workload identity          |
| Logging bearer tokens                          | Logs become a source of credential theft                     | Redact tokens and sensitive data                     |

# 23. Spring Security interview questions and answers

Use this section as your final revision sheet.

## Basic questions

### Q1. What is Spring Security?

A security framework for Spring applications that provides authentication, authorization, and protection against common security threats.

### Q2. What is authentication?

Verifying the identity of a user, service, or other principal.

### Q3. What is authorization?

Deciding whether an authenticated or otherwise identified principal is allowed to perform an operation.

### Q4. What is the difference between 401 and 403?

401 means acceptable authentication credentials are missing or invalid. 403 means access is denied. Applications may intentionally use other statuses, such as 404, to conceal resource existence.

### Q5. What is `SecurityContextHolder`?

A mechanism for accessing the current `SecurityContext`, which holds the current `Authentication`.

### Q6. What is `UserDetailsService`?

An interface commonly used to load user details by username for username/password authentication.

### Q7. What is `PasswordEncoder`?

An abstraction for encoding passwords for storage and verifying submitted passwords against stored password hashes.

### Q8. What is `SecurityFilterChain`?

A configured chain of security filters applied to matching HTTP requests.

### Q9. What is `@EnableMethodSecurity`?

It enables method-level security annotations such as `@PreAuthorize`.

### Q10. Why was `WebSecurityConfigurerAdapter` removed from modern configuration patterns?

Modern Spring Security uses component-based configuration with beans such as `SecurityFilterChain`, which offers a more flexible and composable approach.

## JWT questions

### Q11. What is JWT?

A compact token format used to carry claims in a structured form, often protected by a signature.

### Q12. Is JWT encrypted?

Not necessarily. Common signed JWTs are readable; their signatures protect integrity, not confidentiality.

### Q13. What happens if someone changes the JWT payload?

Correct signature verification should fail because the signed content has changed.

### Q14. Why use JWT in microservices?

It can allow resource servers to validate access tokens locally without requiring an identity-provider call for every request.

### Q15. What are the disadvantages of JWT?

Revocation, stale claims, key management, token theft, validation complexity, and token-size overhead.

### Q16. How do you invalidate a JWT before expiry?

Use an appropriate revocation or rejection mechanism, such as a denylist, introspection, a user/session version check, or sufficiently short token lifetimes, depending on requirements.

### Q17. What is a JWT audience?

The intended recipient of a token. A resource server should validate that the token is intended for it when audience validation is required.

### Q18. What is the difference between access and refresh tokens?

Access tokens grant access to protected resources. Refresh tokens are used with the authorization server to obtain new access tokens.

### Q19. What is JWKS?

JSON Web Key Set: a JSON structure containing public keys or other key representations used to verify tokens. In common asymmetric JWT setups, the resource server uses the appropriate public key.

### Q20. Why should we avoid custom JWT filters?

They can duplicate framework functionality and introduce bugs in signature verification, exception handling, filter ordering, and security-context management.

## OAuth and OIDC questions

### Q21. Is OAuth 2.0 an authentication protocol?

OAuth 2.0 is primarily an authorization framework. OIDC adds standardized authentication and identity information.

### Q22. What is an authorization server?

The server responsible for issuing tokens and processing supported authorization grants.

### Q23. What is a resource server?

A server hosting protected resources that accepts and validates suitable access tokens.

### Q24. What is a client in OAuth?

An application that requests authorization and obtains tokens to access protected resources.

### Q25. What is Authorization Code with PKCE?

An authorization flow in which the client exchanges an authorization code using a transaction-specific verifier, reducing the risk of code interception and misuse.

### Q26. What is Client Credentials Flow?

A flow used by a client to obtain a token for its own machine-to-machine access, rather than acting on behalf of a signed-in human user.

### Q27. What is the difference between an ID token and an access token?

An ID token communicates authentication information to the OIDC client. An access token grants access to a resource server.

### Q28. Does OAuth require JWT?

No. OAuth access tokens can be JWTs or opaque tokens.

### Q29. What is the difference between OAuth Client and Resource Server in Spring Security?

The client obtains or uses tokens to access protected resources. The resource server validates incoming access tokens to protect its own APIs.

### Q30. Why is the password grant discouraged?

It requires the client to handle the user's password directly and does not fit modern authentication patterns well. It should not be used for new systems.

## Architect-level questions

### Q31. How would you secure 20 microservices?

Use a trusted identity provider, well-defined token audiences and scopes, resource-server validation, least-privilege service identities, secure inter-service communication, and resource-level authorization.

### Q32. Should the gateway be the only component validating tokens?

Not necessarily. Each protected backend should enforce its own trust boundary, particularly if it can be reached directly or has distinct authorization requirements.

### Q33. How do you handle immediate user revocation?

Determine the required revocation time, then choose appropriate token lifetimes and revocation controls. Local JWT validation alone does not guarantee immediate invalidation.

### Q34. How would you prevent one tenant from reading another tenant's data?

Use tenant-aware authorization, trusted tenant mapping, scoped data access, and systematic cross-tenant tests. Consider database-level isolation where appropriate.

### Q35. How do you secure service-to-service calls?

Use workload identities, appropriate OAuth client credentials or other trusted machine authentication, encrypted transport, and narrowly scoped permissions.

### Q36. What happens if the identity provider goes down?

New login or token issuance may fail. Locally verifiable, unexpired JWTs may continue to work, but key refresh and other identity-provider-dependent operations can be affected. Define explicit failure behavior.

### Q37. How do you manage signing-key rotation?

Publish new verification keys, begin using the new signing key, allow validators to refresh their trusted keys, and retire the old key after the relevant token validity window. Handle compromised keys as an emergency.

### Q38. How do you protect a high-value payment API?

Use strong identity verification, fine-grained authorization, account ownership checks, business limits, state validation, audit logging, secure service identities, and appropriate step-up authentication.

### Q39. Where should authorization be enforced?

At the HTTP boundary and at the appropriate business-operation or data-access boundaries. Sensitive ownership, tenant, and domain rules must not depend solely on controller annotations.

### Q40. How would you debug unexpected 403 errors?

Check the selected filter chain, request matchers, authentication principal, granted authorities, role prefixes, scope-to-authority mapping, method-security expressions, and resource-level policy. Use security logs without exposing credentials.

# 24. The ultimate Spring Security mental model

If you remember only one flow, remember this:

1. Receive the request

HTTP request enters the application

2. Authenticate

Validate a session, password, token, or other credential

3. Establish identity

Build the Authentication and SecurityContext

4. Authorize

Check roles, authorities, scopes and business access policies

5. Execute the business operation

Only when the required checks pass

Keep these distinctions clear:

* Spring Security is the security framework.

* Authentication establishes identity.

* Authorization decides access.

* JWT is a token format.

* OAuth 2.0 is a delegated authorization framework.

* OIDC adds an authentication and identity layer to OAuth 2.0.

* A session is one way to maintain authenticated state.

* A resource server validates incoming access tokens.

* An authorization server issues tokens.

* CSRF and CORS address different problems.

# 25. Your Technical Architect checklist

Before an interview, see whether you can explain each of these without looking at the guide.

Interview readiness

## 0/15

Explain the Spring Security filter chain and SecurityContext.

Trace username/password authentication through AuthenticationManager and AuthenticationProvider.

Explain UserDetailsService and PasswordEncoder.

Distinguish 401, 403, roles, authorities, and scopes.

Configure SecurityFilterChain and method-level security.

Explain JWT headers, claims, signatures, validation, expiry, and revocation.

Implement a JWT resource server using Spring Boot configuration.

Explain OAuth roles, Authorization Code with PKCE, and Client Credentials.

Distinguish ID tokens from access tokens.

Explain session authentication versus stateless bearer-token authentication.

Distinguish CSRF from CORS and know when CSRF can safely be disabled.

Design API Gateway and downstream microservice security.

Explain key rotation, token theft, least privilege, and identity-provider outages.

Prevent BOLA, cross-tenant access, and confused-deputy vulnerabilities.

Test unauthorized requests, invalid tokens, insufficient permissions, and resource ownership.

Reset checklistCopy checklist

## Final advice

At the Technical Architect level, do not stop at explaining how to configure Spring Security. Explain why you chose the security model, what threats it addresses, what happens when a dependency fails, and what trade-offs you are accepting.

For example, instead of saying, “We use JWT because it is stateless,” explain:

“We use short-lived access tokens that our resource servers validate locally to reduce authentication-service dependencies on the request path. We validate issuer, audience, signature, expiry, and required authorities. Because local validation does not provide immediate revocation, we use a separate revocation strategy for operations where rapid access removal is essential.”

That answer demonstrates framework knowledge, security awareness, and architecture-level decision-making — exactly the combination to aim for in senior and architect interviews.
