# Lamisplus 3.0 - Multi-Tenant Healthcare Platform
## Technical Documentation

## Technical Documentation

---

## Table of Contents

[Document Control](#document-control)
[1. Overview](#1-overview)
[Application Name](#application-name)
[Purpose](#purpose)
[Scope](#scope)
[Audience](#audience)
[2. System Architecture](#2-system-architecture)
[High-Level Architecture](#high-level-architecture)
[Core Components](#core-components)
[1. Frontend (React 18 + Module Federation)](#1-frontend-react-18--module-federation)
[2. Spring Boot Core Application](#2-spring-boot-core-application)
[3. Database Layer (PostgreSQL)](#3-database-layer-postgresql)
Technology Stack](#technology-stack)
[Deployment Models](#deployment-models)
[3. Multi-Tenancy Implementation](#3-multi-tenancy-implementation)
[Schema-per-Tenant Strategy](#schema-per-tenant-strategy)
[Tenant Context Management](#tenant-context-management)
[Database Schema Switching](#database-schema-switching)
[Tenant Isolation Guarantees](#tenant-isolation-guarantees)
[4. Plugin System Architecture](#4-plugin-system-architecture)
[Plugin Structure](#plugin-structure)
[Plugin Lifecycle](#plugin-lifecycle)
[Plugin Loading Process](#plugin-loading-process)
[Plugin Security](#plugin-security)
[Hot-Reloading Plugins](#hot-reloading-plugins)
[5. Security & Authentication](#5-security--authentication)
[JWT Authentication](#jwt-authentication)
[Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
[Permission System](#permission-system)
[Security Filters](#security-filters)
[Audit Logging](#audit-logging)
[6. Configuration & Setup](#6-configuration--setup)
[Application Properties](#application-properties)
[Database Configuration](#database-configuration)
[Security Configuration](#security-configuration)
[Environment Profiles](#environment-profiles)
[7. Code Components](#7-code-components)
[Core Entities](#core-entities)
[Services](#services)
[Repositories](#repositories)
[Controllers](#controllers)
[Filters](#filters)
[8. Data Management](#8-data-management)
[Schema Migrations](#schema-migrations)
[Backup Strategies](#backup-strategies)
[Data Lifecycle](#data-lifecycle)
[9. Deployment & DevOps](#9-deployment--devops)
[Production Deployment](#production-deployment)
[Monitoring](#monitoring)
[Operational Procedures](#operational-procedures)
[10. API Documentation](#10-api-documentation)
[Authentication Endpoints](#authentication-endpoints)
[User Management Endpoints](#user-management-endpoints)
[Error Responses](#error-responses)
[11. Error Handling & Logging](#11-error-handling--logging)
[Global Exception Handler](#global-exception-handler)
[12. Scaling & Performance](#12-scaling--performance)
[Connection Pooling (HikariCP)](#connection-pooling-hikaricp)
[Caching Strategy](#caching-strategy)
[Circuit Breaker (Resilience4j)](#circuit-breaker-resilience4j)
[13. Tenant Lifecycle Management](#13-tenant-lifecycle-management)
[Tenant States](#tenant-states)
14. [Appendices](#14-appendices)
[Appendix A: Glossary](#appendix-a-glossary)
[Appendix B: References](#appendix-b-references)
[Conclusion](#conclusion)

---

## Document Control

| Field | Value |
|-------|-------|
| **Application Name** | Lamisplus 3.0 (CORE 3.0) |
| **Version** | 3.0.0 |
| **Author(s)** | Lamisplus Development Team |
| **Date** | October 25, 2025 |
| **Document Version** | 1.0 |
| **Status** | Production |

---

## 1. Overview

### Application Name
**Lamisplus 3.0** - Multi-Tenant Healthcare Platform with Dynamic Plugin Architecture

### Purpose

Lamisplus 3.0 is an enterprise-grade, multi-tenant healthcare platform designed for hospitals, medical facilities, and healthcare networks. The platform enables multiple independent healthcare organizations (tenants) to securely share a single software instance while maintaining complete data isolation, tenant-specific configurations, and customizable feature sets through a dynamic plugin architecture.

**Key Objectives:**
- **Complete Data Isolation**: Each tenant's data is stored in separate database schemas, ensuring HIPAA compliance and absolute data privacy
- **Plugin-Based Architecture**: Healthcare facilities can install only the modules they need (Lab Management, Pharmacy, Radiology, Patient Management, etc.)
- **Zero-Downtime Plugin Management**: Plugins can be enabled, disabled, or updated without restarting the entire system
- **Scalable Multi-Tenancy**: Designed to efficiently support 1000+ tenants with independent configurations
- **Enterprise Security**: JWT-based authentication, role-based access control (RBAC), granular permissions, and comprehensive audit logging

### Scope

This document provides comprehensive technical documentation covering:

1. **System Architecture** - High-level design, component interactions, and deployment models
2. **Multi-Tenancy Implementation** - Schema-per-tenant strategy, tenant isolation, and context management
3. **Plugin System Architecture** - Dynamic plugin loading, lifecycle management, and security validation
4. **Security & Authentication** - JWT authentication, RBAC, permission system, and audit logging
5. **Configuration & Setup** - Application properties, database setup, and environment profiles
6. **Code Components** - Key classes, services, repositories, and integration patterns
7. **Data Management** - Schema migrations, backups, and data lifecycle
8. **Deployment & DevOps** - Production deployment, monitoring, and operational procedures
9. **API Documentation** - RESTful endpoints, request/response formats, and integration guides
10. **Testing Strategies** - Unit testing, integration testing, and tenant isolation validation

### Audience

This documentation is intended for:

- **Backend Developers** - Understanding architecture and implementing new features
- **DevOps Engineers** - Deploying, configuring, and maintaining infrastructure
- **QA/Test Engineers** - Creating test scenarios and validating multi-tenant functionality
- **System Administrators** - Managing tenant lifecycle and troubleshooting issues
- **Technical Architects** - Evaluating system design and planning enhancements
- **Security Engineers** - Auditing security controls and compliance measures
- **Integration Developers** - Building external integrations and plugins

---

## 2. System Architecture

### High-Level Architecture

Lamisplus 3.0 follows a modern layered architecture with multi-tenancy support at every level:

```
+-------------------------------------------------------------+
|                  Frontend Layer (React 18)                  |
|       Module Federation + Rspack + Plugin Remotes           |
|        - Core Shell: User Management, Authentication        |
|        - Plugin Remotes: Lab, Pharmacy, Patient Modules     |
+-------------------------------------------------------------+
                           |
                           | HTTPS/REST
                           |
                           v
+-------------------------------------------------------------+
|                  API Gateway / Load Balancer                |
|         (Spring Boot Embedded Tomcat / Nginx)               |
|  - SSL/TLS Termination    - Rate Limiting per Tenant        |
|  - Request Routing        - Health Checks                   |
+-------------------------------------------------------------+
                           |
                           |
                           v
+-------------------------------------------------------------+
|              Security & Tenant Resolution Layer             |
|  +------------------------------------------------------+   |
|  |  TenantResolutionFilter (Highest Priority)           |   |
|  |  - Extract X-Tenant-Id from header or JWT            |   |
|  |  - Set TenantContext (ThreadLocal)                   |   |
|  |  - Validate tenant format and permissions            |   |
|  +------------------------------------------------------+   |
|                           |                                 |
|                           v                                 |
|  +------------------------------------------------------+   |
|  |  JwtAuthenticationFilter                             |   |
|  |  - Validate JWT tokens                               |   |
|  |  - Extract user, roles, and tenant from token        |   |
|  |  - Set Spring Security Context                       |   |
|  +------------------------------------------------------+   |
|                           |                                 |
|                           v                                 |
|  +------------------------------------------------------+   |
|  |  PluginAuthorizationFilter                           |   |
|  |  - Verify plugin-specific permissions                |   |
|  |  - Validate tenant has plugin enabled                |   |
|  +------------------------------------------------------+   |
+-------------------------------------------------------------+
                           |
                           |
         +-----------------+------------------+
         |                 |                  |
         v                 v                  v
+-----------------+------------------+---------------------+
|  Core Services  |  Plugin Services | Support Services    |
|                 |                  |                     |
| • AuthService   | • Lab Module     | • HealthMonitoring  |
| • UserService   | • Pharmacy       | • CacheService      |
| • RoleService   | • Radiology      | • CircuitBreaker    |
| • PermService   | • Patient Mgmt   | • AuditLogging      |
| • PluginMgr     | • Setup Module   | • FileStorage       |
| • TenantAdmin   | (Hot-loadable)   | • RateLimiter       |
+-----------------+------------------+---------------------+
         |                 |                  |
         +-----------------+------------------+
                           |
                           v
+-------------------------------------------------------------+
|              Data Layer (PostgreSQL 14+)                    |
|  +------------------------------------------------------+   |
|  |  Schema: _system_ (Global Metadata)                  |   |
|  |  - tenants                                           |   |
|  |  - global_plugins                                    |   |
|  |  - super_admin_audit_log                            |    |
|  +------------------------------------------------------+   |
|  +------------------------------------------------------+   |
|  |  Schema: tenant_clinic_a (Tenant-Specific)          |    |
|  |  - users, roles, permissions                        |    |
|  |  - plugin_configs                                   |    |
|  |  - audit_events                                     |    |
|  |  - refresh_tokens                                   |    |
|  +------------------------------------------------------+   |
|  +------------------------------------------------------+   |
|  |  Schema: tenant_hospital_b (Tenant-Specific)        |    |
|  |  - users, roles, permissions                        |    |
|  |  - plugin_configs                                   |    |
|  |  - audit_events                                     |    |
|  +------------------------------------------------------+   |
+-------------------------------------------------------------+
                           |
                           |
                           v
+-------------------------------------------------------------+
|              Caching Layer (Redis)                          |
|  - User Session Cache (5-minute TTL)                        |
|  - Tenant Configuration Cache                               |
|  - Plugin Metadata Cache                                    |
|  - Rate Limiting Counters                                   |
+-------------------------------------------------------------+
```

### Core Components

#### 1. Frontend (React 18 + Module Federation)

**Technology Stack:**
- React 18 with TypeScript
- Rspack (Webpack alternative for faster builds)
- Module Federation for plugin architecture
- Redux Toolkit for state management
- Tailwind CSS for styling

**Features:**
- **Shell Application**: Core UI, authentication, user management
- **Plugin Remotes**: Dynamically loaded micro-frontends per plugin
- **Route Management**: Dynamic routing based on enabled plugins
- **Shared Components**: Reusable UI components across plugins

#### 2. Spring Boot Core Application

**Technology Stack:**
- Java 21 (LTS with Virtual Threads support)
- Spring Boot 3.3.5
- Spring Security 6.x
- Spring Data JPA / Hibernate
- PostgreSQL 14+ JDBC Driver
- Liquibase for schema migrations
- Resilience4j for circuit breakers and rate limiting

**Core Services:**
- **Authentication Service**: JWT generation, refresh token management, MFA support
- **User Management Service**: CRUD operations for users, roles, and permissions
- **Tenant Administration Service**: Tenant lifecycle, provisioning, and configuration
- **Plugin Manager Service**: Hot-loading plugins, lifecycle management, security validation
- **Audit Service**: Comprehensive logging of all system activities

#### 3. Database Layer (PostgreSQL)

**Architecture**: Schema-per-Tenant

**Structure:**
```sql
-- Global System Schema
CREATE SCHEMA _system_;

-- Tenant Schemas (dynamically created)
CREATE SCHEMA tenant_<tenant_id>;
```

**Key Features:**
- Automatic schema switching based on tenant context
- Connection pooling with HikariCP (max 40 connections)
- Liquibase migrations applied per-tenant schema
- Global plugin registry in system schema

#### 4. Caching Layer (Redis)

**Configuration:**
- Host: localhost
- Port: 6379
- Max connections: 8
- Lettuce client with connection pooling

**Cached Data:**
- User authentication details (5-minute TTL)
- Tenant configurations
- Plugin metadata and health status
- Rate limiting counters

#### 5. External Integrations

- **Email Service**: SMTP for notifications (Gmail configured)
- **File Storage**: Local filesystem or S3-compatible storage
- **Monitoring**: Actuator endpoints for health checks and metrics

### Deployment Architecture

**Container Platform**: Docker containers orchestrated with Docker Compose or Kubernetes

**Production Topology:**
```
Load Balancer (Nginx)
    │
    ├─── Application Instance 1 (Spring Boot)
    ├─── Application Instance 2 (Spring Boot)
    └─── Application Instance 3 (Spring Boot)
         │
         ├─── PostgreSQL Primary
         │    └─── Read Replicas (2)
         │
         └─── Redis Cluster
              ├─── Master Node
              └─── Replica Nodes (2)
```

**Key Deployment Features:**
- Zero-downtime deployments
- Auto-scaling based on CPU/memory usage
- Health check endpoints
- Blue-green deployment support

---

## 3. Multi-Tenancy Design

### Approach

**Selected Model**: **Schema per Tenant**

Lamisplus 3.0 implements a schema-per-tenant multi-tenancy strategy where each tenant's data resides in a completely separate PostgreSQL schema within a shared database instance.

**Database Structure:**
```
PostgreSQL Database: core_database
├── Schema: _system_ (Global metadata)
│   ├── tenants
│   ├── global_plugins
│   ├── super_admin_audit_log
│   └── databasechangelog
│
├── Schema: tenant_clinic_alpha
│   ├── users
│   ├── roles
│   ├── permissions
│   ├── user_roles
│   ├── user_permissions
│   ├── plugin_configs
│   ├── audit_events
│   ├── refresh_tokens
│   └── menu_items
│
├── Schema: tenant_hospital_beta
│   ├── users
│   ├── roles
│   ├── permissions
│   ├── user_roles
│   ├── user_permissions
│   ├── plugin_configs
│   ├── audit_events
│   ├── refresh_tokens
│   └── menu_items
│
└── [Additional tenant schemas...]
```

### Justification and Trade-offs

**Why Schema per Tenant?**

| Criterion | Evaluation | Details |
|-----------|-----------|---------|
| **Data Isolation** | ✅ Excellent | Complete schema-level separation prevents any cross-tenant queries |
| **Security** | ✅ Excellent | Meets HIPAA and healthcare data privacy requirements |
| **Performance** | ✅ Good | No discriminator filtering needed, direct schema access |
| **Scalability** | ⚠️ Moderate | Limited by single database server (mitigated with read replicas) |
| **Operational Complexity** | ✅ Moderate | Easier than separate databases, more isolated than shared schema |
| **Cost Efficiency** | ✅ Good | Shared infrastructure reduces per-tenant costs |
| **Backup/Restore** | ✅ Good | Schema-level backup and restore possible |
| **Migration Management** | ⚠️ Moderate | Migrations must run across all tenant schemas |
| **Resource Utilization** | ✅ Good | Shared connection pooling and query cache |

**Trade-offs Accepted:**
1. **Single Database Bottleneck**: Mitigated through connection pooling, read replicas, and future sharding strategy
2. **Migration Complexity**: Automated through Liquibase with proper rollback mechanisms
3. **Schema Count Limits**: PostgreSQL supports thousands of schemas efficiently

### Tenant Identification

Lamisplus 3.0 uses a multi-source tenant identification strategy with failover logic:

#### 1. HTTP Header (Primary Method)

```http
GET /api/v1/core/users HTTP/1.1
Host: api.lamisplus.com
Authorization: Bearer eyJhbGc...
X-Tenant-Id: clinic_alpha
Content-Type: application/json
```

**Implementation:**
```java
String tenantId = request.getHeader("X-Tenant-Id");
```

**Usage**: API clients, mobile apps, external integrations

#### 2. JWT Claim (Authenticated Requests)

**JWT Payload:**
```json
{
  "sub": "user123",
  "tenantId": "clinic_alpha",
  "isSuperAdmin": false,
  "authorities": ["ROLE_LAB_TECH", "ROLE_VIEWER"],
  "permissions": ["READ_PATIENTS", "WRITE_LAB_TESTS"],
  "iat": 1729900000,
  "exp": 1729901800
}
```

**Extraction:**
```java
Claims claims = Jwts.parser()
    .verifyWith(getSigningKey())
    .build()
    .parseSignedClaims(token)
    .getPayload();

String tenantId = claims.get("tenantId", String.class);
```

**Usage**: All authenticated API requests (single source of truth)

#### 3. Refresh Token Cookie (Token Refresh)

For `/auth/refresh` endpoint:
```java
Cookie refreshTokenCookie = getRefreshTokenCookie(request);
String tenantId = refreshTokenService.extractTenantFromRefreshToken(
    refreshTokenCookie.getValue()
);
```

**Usage**: Automatic token refresh without requiring tenant header

#### 4. Resolution Priority

The `TenantResolutionFilter` applies the following logic:

```java
Priority 1: For authentication endpoints → X-Tenant-Id header required
Priority 2: For refresh endpoint → Extract from refresh token cookie
Priority 3: For authenticated endpoints → Extract from JWT (must match header if provided)
Priority 4: For super admins → System tenant (_system_)
```

**Security Validation:**
- Tenant format validation (alphanumeric, hyphens, underscores only)
- JWT tenant vs. header tenant mismatch detection
- Protected tenant access control (system, global tenants require super admin)

### Tenant Context Propagation

Lamisplus 3.0 uses ThreadLocal storage to maintain tenant context throughout request processing.

#### TenantContext Implementation

**Core Class**: `coreapplication.service.plugin_manager.TenantContext`

```java
public class TenantContext {
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();
    private static final ThreadLocal<Boolean> isSuperAdmin = new ThreadLocal<>();
    private static final ThreadLocal<Boolean> isCrossTenantQuery = new ThreadLocal<>();
    
    public static void setTenantId(String tenantId) {
        if (tenantId == null || tenantId.trim().isEmpty()) {
            throw new IllegalArgumentException("Tenant ID cannot be null or empty");
        }
        
        // Validate protected tenant access
        if (SecurityConstants.PROTECTED_TENANTS.contains(tenantId)) {
            validateProtectedTenantAccess(tenantId);
        }
        
        currentTenant.set(tenantId);
        log.debug("Tenant context set: {}", tenantId);
    }
    
    public static String getTenantId() {
        String tenantId = currentTenant.get();
        
        if (tenantId == null) {
            throw new IllegalStateException(
                "Tenant context not set. All operations require explicit tenant context.");
        }
        
        return tenantId;
    }
    
    public static void clear() {
        currentTenant.remove();
        isSuperAdmin.remove();
        isCrossTenantQuery.remove();
    }
    
    // Auto-closing scope for safe context management
    public static Scope withTenant(String tenantId) {
        String previousTenant = currentTenant.get();
        setTenantId(tenantId);
        return new Scope(previousTenant);
    }
    
    public static class Scope implements AutoCloseable {
        private final String previousTenant;
        
        @Override
        public void close() {
            if (previousTenant != null) {
                currentTenant.set(previousTenant);
            } else {
                clear();
            }
        }
    }
}
```

#### Request Flow with Tenant Context

```
1. Request arrives → TenantResolutionFilter (HIGHEST_PRECEDENCE)
   ↓
2. Extract tenant ID (from header, JWT, or cookie)
   ↓
3. Set TenantContext.setTenantId(tenantId)
   ↓
4. Add X-Current-Tenant response header
   ↓
5. Pass request through filter chain
   ↓
6. JwtAuthenticationFilter validates token
   ↓
7. Service layer uses TenantContext.getTenantId()
   ↓
8. Repository queries use current schema
   ↓
9. Response returned to client
   ↓
10. TenantContext.clear() (in finally block)
```

#### Repository Integration

**TenantAwareRepository Interface:**
```java
@NoRepositoryBean
public interface TenantAwareRepository<T, ID> extends JpaRepository<T, ID> {
    
    default String getTenantSchema() {
        String tenantId = TenantContext.getTenantId();
        return "tenant_" + tenantId;
    }
}
```

**Native Query with Schema:**
```java
@Repository
public interface UserRepository extends TenantAwareRepository<User, Long> {
    
    @Query(value = "SELECT * FROM users WHERE username = :username", 
           nativeQuery = true)
    Optional<User> findByUsername(@Param("username") String username);
}
```

**JPA Configuration:**
```java
@Configuration
@EnableJpaRepositories(basePackages = "coreapplication.repository")
public class DatabaseConfiguration {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource) {
        
        LocalContainerEntityManagerFactoryBean em = 
            new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("coreapplication.domain.entity");
        
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        vendorAdapter.setGenerateDdl(false);
        vendorAdapter.setShowSql(false);
        
        em.setJpaVendorAdapter(vendorAdapter);
        em.setJpaProperties(additionalProperties());
        
        return em;
    }
    
    private Properties additionalProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", 
            "org.hibernate.dialect.PostgreSQLDialect");
        properties.setProperty("hibernate.default_schema", 
            "${tenant.schema:public}");
        return properties;
    }
}
```

#### Async Context Propagation

For asynchronous operations, tenant context must be explicitly propagated:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-task-");
        executor.setTaskDecorator(new TenantAwareTaskDecorator());
        executor.initialize();
        return executor;
    }
}

public class TenantAwareTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        String tenantId = TenantContext.getTenantIdOrNull();
        
        return () -> {
            try {
                if (tenantId != null) {
                    TenantContext.setTenantId(tenantId);
                }
                runnable.run();
            } finally {
                TenantContext.clear();
            }
        };
    }
}
```

### Special Tenant Contexts

#### 1. System Tenant (_system_)

Used for global operations requiring access to system-wide metadata:

```java
// Only super admins can access system tenant
try (TenantContext.Scope scope = TenantContext.withSystemTenant()) {
    // Access global_plugins, tenants tables
    List<GlobalPluginEntity> plugins = globalPluginRepository.findAll();
}
```

#### 2. Super Admin Mode

Bypasses all tenant filters for administrative operations:

```java
public void enableSuperAdminMode() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    
    boolean hasSuperAdminRole = auth.getAuthorities().stream()
        .anyMatch(a -> "ROLE_SUPER_ADMIN".equals(a.getAuthority()));
    
    if (!hasSuperAdminRole) {
        throw new SecurityException("Only super admins can enable super admin mode");
    }
    
    TenantContext.setTenantId(SecurityConstants.SYSTEM_TENANT);
    isSuperAdmin.set(Boolean.TRUE);
}
```

#### 3. Cross-Tenant Query Mode

For specific whitelisted services that need to query across tenants:

```java
// Whitelisted services only
private static final Set<String> CROSS_TENANT_QUERY_WHITELIST = Set.of(
    "coreapplication.service.TenantLookupService",
    "coreapplication.service.PasswordResetService",
    "coreapplication.service.EmailVerificationService"
);

public static void enableCrossTenantQueryMode() {
    String callerClass = getCallerClassName();
    
    if (!CROSS_TENANT_QUERY_WHITELIST.contains(callerClass)) {
        throw new SecurityException(
            "Service '" + callerClass + "' is not authorized for cross-tenant queries");
    }
    
    TenantContext.setTenantId(SecurityConstants.SYSTEM_TENANT);
    isCrossTenantQuery.set(Boolean.TRUE);
}
```

---

## 4. Configuration and Setup

### Application Properties

**Main Configuration** (`application.yml`):

```yaml
spring:
  application:
    name: CoreApplication
  
  # Database Configuration
  datasource:
    url: jdbc:postgresql://localhost:5432/core_database
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:admin}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 40
      minimum-idle: 8
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
      auto-commit: false
      connection-init-sql: "SET SESSION application_name = 'plugin_system'"
  
  # Liquibase Configuration
  liquibase:
    change-log: classpath:/db/changelog/db.changelog-master.xml
    enabled: true
  
  # JPA/Hibernate Configuration
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: update  # Use 'none' in production
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        query.plan_cache_max_size: 2048
  
  # Redis Configuration
  data:
    redis:
      host: localhost
      port: 6379
      database: 0
      lettuce:
        pool:
          enabled: true
          max-active: 8
          max-idle: 8
          min-idle: 0

# Application-specific Configuration
application:
  # Security Configuration
  security:
    user-cache:
      enabled: true
      ttl-minutes: 5
      max-size: 10000
    jwt:
      secret-key: ${JWT_SECRET}
      expiration: 1800000  # 30 minutes
      refresh-token:
        expiration: 604800000  # 7 days
        remember-me-expiration: 2592000000  # 30 days
      cookie:
        name: refreshToken
        http-only: true
        secure: ${COOKIE_SECURE:false}
        same-site: Strict
  
  # CORS Configuration
  cors:
    allowed-origins:
      - http://localhost:3000
      - http://localhost:8080
    allowed-methods:
      - GET
      - POST
      - PUT
      - DELETE
      - OPTIONS
      - PATCH
    allowed-headers:
      - Authorization
      - Content-Type
      - X-Requested-With
      - X-Tenant-Id
    exposed-headers:
      - Authorization
      - X-Tenant-Id
      - X-Current-Tenant
  
  # Multi-Tenant Configuration
  multi-tenant:
    enabled: true
    default-tenant: "default"
    identification-strategy: header
    system-tenant-id: "_system_"
    global-tenant-id: "_global_"
  
  # Plugin Configuration
  plugin:
    dir: ${PLUGIN_DIR:plugins}
    max-size: 104857600  # 100MB
    storage:
      type: filesystem
      filesystem:
        base-path: ./plugin-storage
    marketplace:
      enabled: true
      auto-install-updates: false
      require-approval: true

# Plugin Resource Limits
plugin:
  resource-limits:
    max-memory-mb: 256
    max-cpu-seconds: 300
    max-threads: 20
    auto-restart-enabled: true
    monitoring-interval-seconds: 30

# Rate Limiting Configuration
rate-limiter:
  global:
    enabled: true
    limit-for-period: 10000
    limit-refresh-period: 1m
  tenant:
    enabled: true
    limit-for-period: 1000
    limit-refresh-period: 1m
  plugin:
    enabled: true
    limit-for-period: 100
    limit-refresh-period: 1m

# Circuit Breaker Configuration (Resilience4j)
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
        automatic-transition-from-open-to-half-open-enabled: true

# Server Configuration
server:
  port: ${SERVER_PORT:8080}
  compression:
    enabled: true
    mime-types: text/html,text/xml,application/json
    min-response-size: 1024

# Actuator/Monitoring Configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,circuitbreakers
  endpoint:
    health:
      show-details: when-authorized

# Logging Configuration
logging:
  level:
    root: INFO
    coreapplication: DEBUG
    org.hibernate.SQL: false
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] [TENANT:%X{tenantId}] %-5level %logger{36} - %msg%n"
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 7
```

### Environment Profiles

#### Development Profile (`application-dev.yml`)

```yaml
spring:
  config:
    activate:
      on-profile: dev

  datasource:
    url: jdbc:postgresql://localhost:5432/core_database_dev
    username: dev_user
    password: dev_password

  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update

logging:
  level:
    coreapplication: DEBUG
    org.hibernate.SQL: true
```

#### Production Profile (`application-prod.yml`)

```yaml
spring:
  config:
    activate:
      on-profile: production

  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10

  jpa:
    hibernate:
      ddl-auto: none  # Use Liquibase only
    show-sql: false

application:
  security:
    jwt:
      cookie:
        secure: true
  
  cors:
    allowed-origins:
      - https://yourdomain.com
      - https://www.yourdomain.com
  
  plugin:
    storage:
      type: S3

logging:
  level:
    root: WARN
    coreapplication: INFO
```

### Database Setup

#### Initial Database Creation

```sql
-- Create main database
CREATE DATABASE core_database
    WITH 
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0;

-- Create application user
CREATE USER lamisplus_user WITH PASSWORD 'secure_password_here';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE core_database TO lamisplus_user;
GRANT CREATE ON DATABASE core_database TO lamisplus_user;

-- Connect to the database
\c core_database

-- Create system schema for global metadata
CREATE SCHEMA _system_;
GRANT ALL ON SCHEMA _system_ TO lamisplus_user;

-- Set search path
ALTER USER lamisplus_user SET search_path TO _system_, public;
```

#### System Schema Tables

**Tenants Table:**
```sql
CREATE TABLE _system_.tenants (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    schema_name VARCHAR(100) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_modified_date TIMESTAMP,
    created_by VARCHAR(100),
    last_modified_by VARCHAR(100),
    metadata JSONB,
    CONSTRAINT valid_tenant_id CHECK (tenant_id ~ '^[a-z0-9_-]+$'),
    CONSTRAINT valid_status CHECK (status IN ('ACTIVE', 'SUSPENDED', 'DELETED'))
);

CREATE INDEX idx_tenants_tenant_id ON _system_.tenants(tenant_id);
CREATE INDEX idx_tenants_status ON _system_.tenants(status);
```

**Global Plugins Table:**
```sql
CREATE TABLE _system_.global_plugins (
    id BIGSERIAL PRIMARY KEY,
    plugin_id VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    version VARCHAR(50) NOT NULL,
    description TEXT,
    author VARCHAR(255),
    jar_file_name VARCHAR(255),
    main_class VARCHAR(255),
    status VARCHAR(50) DEFAULT 'AVAILABLE',
    required_permissions TEXT[],
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_modified_date TIMESTAMP,
    metadata JSONB
);
```

#### Tenant Schema Template

When a new tenant is provisioned, the following schema is created:

```sql
-- Create tenant schema
CREATE SCHEMA tenant_<tenant_id>;

-- Set search path
SET search_path TO tenant_<tenant_id>;

-- Users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone_number VARCHAR(20),
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    account_non_locked BOOLEAN NOT NULL DEFAULT TRUE,
    credentials_non_expired BOOLEAN NOT NULL DEFAULT TRUE,
    mfa_enabled BOOLEAN DEFAULT FALSE,
    mfa_secret VARCHAR(255),
    created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_modified_date TIMESTAMP,
    last_login_date TIMESTAMP,
    created_by VARCHAR(100),
    last_modified_by VARCHAR(100)
);

-- Roles table
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_modified_date TIMESTAMP,
    created_by VARCHAR(100),
    last_modified_by VARCHAR(100)
);

-- Permissions table
CREATE TABLE permissions (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    category VARCHAR(100),
    plugin_id VARCHAR(100),
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User-Role mapping
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

-- Role-Permission mapping
CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Plugin configurations
CREATE TABLE plugin_configs (
    id BIGSERIAL PRIMARY KEY,
    plugin_id VARCHAR(100) NOT NULL,
    enabled BOOLEAN DEFAULT FALSE,
    configuration JSONB,
    enabled_date TIMESTAMP,
    disabled_date TIMESTAMP,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_modified_date TIMESTAMP
);

-- Refresh tokens
CREATE TABLE refresh_tokens (
    id BIGSERIAL PRIMARY KEY,
    token VARCHAR(500) UNIQUE NOT NULL,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    expiry_date TIMESTAMP NOT NULL,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    revoked BOOLEAN DEFAULT FALSE,
    revoked_date TIMESTAMP
);

-- Audit events
CREATE TABLE audit_events (
    id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    entity_type VARCHAR(100),
    entity_id VARCHAR(100),
    user_id BIGINT REFERENCES users(id),
    username VARCHAR(100),
    action VARCHAR(50) NOT NULL,
    details JSONB,
    ip_address VARCHAR(50),
    user_agent TEXT,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_enabled ON users(enabled);
CREATE INDEX idx_roles_name ON roles(name);
CREATE INDEX idx_permissions_name ON permissions(name);
CREATE INDEX idx_permissions_plugin ON permissions(plugin_id);
CREATE INDEX idx_plugin_configs_plugin_id ON plugin_configs(plugin_id);
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_expiry ON refresh_tokens(expiry_date);
CREATE INDEX idx_audit_events_created_date ON audit_events(created_date);
CREATE INDEX idx_audit_events_user_id ON audit_events(user_id);
CREATE INDEX idx_audit_events_event_type ON audit_events(event_type);
```

### Migration Management with Liquibase

**Master Changelog** (`db/changelog/db.changelog-master.xml`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- System schema initialization -->
    <include file="classpath:db/changelog/changes/V1_initial_schema_with_tenancy.xml"/>
    
    <!-- Global plugin system -->
    <include file="classpath:db/changelog/changes/global-plugin-system.xml"/>
    
    <!-- Refresh tokens -->
    <include file="classpath:db/changelog/changes/refresh-tokens-table.xml"/>
    
    <!-- Audit events -->
    <include file="classpath:db/changelog/changes/audit_events.sql"/>
    
</databaseChangeLog>
```

**Tenant Schema Migration Service:**

```java
@Service
@Slf4j
public class DatabaseSchemaManager {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private LiquibasePluginHelper liquibaseHelper;
    
    /**
     * Initialize schema for a new tenant
     */
    @Transactional
    public void createTenantSchema(String tenantId) {
        String schemaName = "tenant_" + tenantId;
        
        try {
            // Create schema
            executeSQL("CREATE SCHEMA IF NOT EXISTS " + schemaName);
            
            // Grant privileges
            executeSQL("GRANT ALL ON SCHEMA " + schemaName + " TO lamisplus_user");
            
            // Run Liquibase migrations
            runLiquibaseMigrations(schemaName);
            
            // Initialize default data
            initializeDefaultData(schemaName);
            
            log.info("Tenant schema created successfully: {}", schemaName);
            
        } catch (Exception e) {
            log.error("Failed to create tenant schema: {}", schemaName, e);
            throw new RuntimeException("Schema creation failed", e);
        }
    }
    
    private void runLiquibaseMigrations(String schemaName) throws LiquibaseException {
        Database database = DatabaseFactory.getInstance()
            .findCorrectDatabaseImplementation(
                new JdbcConnection(dataSource.getConnection()));
        
        database.setDefaultSchemaName(schemaName);
        database.setLiquibaseSchemaName(schemaName);
        
        Liquibase liquibase = new Liquibase(
            "db/changelog/tenant/tenant-schema.xml",
            new ClassLoaderResourceAccessor(),
            database
        );
        
        liquibase.update((String) null);
    }
    
    private void executeSQL(String sql) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
        }
    }
    
    private void initializeDefaultData(String schemaName) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            
            stmt.execute("SET search_path TO " + schemaName);
            
            // Insert default admin role
            stmt.execute(
                "INSERT INTO roles (name, description) " +
                "VALUES ('ADMIN', 'Administrator role with full access')"
            );
            
            // Insert default permissions
            stmt.execute(
                "INSERT INTO permissions (name, description, category) VALUES " +
                "('READ_USERS', 'View users', 'USER_MANAGEMENT'), " +
                "('WRITE_USERS', 'Create/edit users', 'USER_MANAGEMENT'), " +
                "('DELETE_USERS', 'Delete users', 'USER_MANAGEMENT')"
            );
        }
    }
}
```

---

## 5. Code Components

### Core Multi-Tenancy Components

| Component | Package | Description |
|-----------|---------|-------------|
| **TenantContext** | `coreapplication.service.plugin_manager` | ThreadLocal tenant context management |
| **TenantResolutionFilter** | `coreapplication.security` | Servlet filter to extract and set tenant context |
| **TenantAwareRepository** | `coreapplication.repository` | Base repository interface with tenant awareness |
| **TenantFilterAspect** | `coreapplication.config` | AOP aspect for tenant filtering on repositories |
| **DatabaseSchemaManager** | `coreapplication.service.database_manager` | Tenant schema creation and migration |
| **SecurityConstants** | `coreapplication.security` | Security-related constants and validation |

### Key Implementation Classes

#### 1. TenantContext (Full Implementation)

**Location**: `coreapplication.service.plugin_manager.TenantContext`

```java
package coreapplication.service.plugin_manager;

import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import java.util.Set;

@Slf4j
public class TenantContext {
    
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();
    private static final ThreadLocal<Boolean> isSuperAdmin = new ThreadLocal<>();
    private static final ThreadLocal<Boolean> isCrossTenantQuery = new ThreadLocal<>();
    
    private static final Set<String> CROSS_TENANT_QUERY_WHITELIST = Set.of(
        "coreapplication.service.TenantLookupService",
        "coreapplication.service.PasswordResetService",
        "coreapplication.service.EmailVerificationService"
    );
    
    private TenantContext() {
        throw new UnsupportedOperationException("Utility class");
    }
    
    public static void setTenantId(String tenantId) {
        if (tenantId == null || tenantId.trim().isEmpty()) {
            throw new IllegalArgumentException("Tenant ID cannot be null or empty");
        }
        
        tenantId = tenantId.trim();
        
        if (SecurityConstants.PROTECTED_TENANTS.contains(tenantId)) {
            validateProtectedTenantAccess(tenantId);
        }
        
        currentTenant.set(tenantId);
        log.debug("Tenant context set: {}", tenantId);
    }
    
    public static void enableSuperAdminMode() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        if (auth == null || !auth.isAuthenticated()) {
            log.debug("Super admin mode enabled during system initialization");
            currentTenant.set(SecurityConstants.SYSTEM_TENANT);
            isSuperAdmin.set(Boolean.TRUE);
            return;
        }
        
        boolean hasSuperAdminRole = auth.getAuthorities().stream()
            .anyMatch(a -> SecurityConstants.ROLE_SUPER_ADMIN.equals(a.getAuthority()));
        
        if (!hasSuperAdminRole) {
            log.error("SECURITY VIOLATION: Attempt to enable super admin mode without authorization. User: {}", 
                auth.getName());
            throw new SecurityException("Only super admins can enable super admin mode");
        }
        
        currentTenant.set(SecurityConstants.SYSTEM_TENANT);
        isSuperAdmin.set(Boolean.TRUE);
        log.warn("SUPER ADMIN MODE ENABLED: User: {}, All tenant filters bypassed", auth.getName());
    }
    
    public static void enableCrossTenantQueryMode() {
        String callerClass = getCallerClassName();
        
        if (!CROSS_TENANT_QUERY_WHITELIST.contains(callerClass)) {
            log.error("SECURITY VIOLATION: Unauthorized cross-tenant query attempt by: {}", callerClass);
            throw new SecurityException(
                "Service '" + callerClass + "' is not authorized for cross-tenant queries");
        }
        
        currentTenant.set(SecurityConstants.SYSTEM_TENANT);
        isCrossTenantQuery.set(Boolean.TRUE);
        log.info("CROSS-TENANT QUERY MODE ENABLED: Service: {}", callerClass);
    }
    
    public static String getTenantId() {
        String tenantId = currentTenant.get();
        
        if (tenantId == null) {
            throw new IllegalStateException(
                "Tenant context not set. All operations require explicit tenant context.");
        }
        
        return tenantId;
    }
    
    public static String getTenantIdOrNull() {
        return currentTenant.get();
    }
    
    public static void clear() {
        currentTenant.remove();
        isSuperAdmin.remove();
        isCrossTenantQuery.remove();
    }
    
    // Auto-closing scope for safe context management
    public static Scope withTenant(String tenantId) {
        String previousTenant = currentTenant.get();
        Boolean previousSuperAdmin = isSuperAdmin.get();
        Boolean previousCrossTenantQuery = isCrossTenantQuery.get();
        
        setTenantId(tenantId);
        
        return new Scope(previousTenant, previousSuperAdmin, previousCrossTenantQuery);
    }
    
    public static class Scope implements AutoCloseable {
        private final String previousTenant;
        private final Boolean previousSuperAdmin;
        private final Boolean previousCrossTenantQuery;
        
        private Scope(String previousTenant, Boolean previousSuperAdmin, 
                     Boolean previousCrossTenantQuery) {
            this.previousTenant = previousTenant;
            this.previousSuperAdmin = previousSuperAdmin;
            this.previousCrossTenantQuery = previousCrossTenantQuery;
        }
        
        @Override
        public void close() {
            if (previousTenant != null) {
                currentTenant.set(previousTenant);
                
                if (Boolean.TRUE.equals(previousSuperAdmin)) {
                    isSuperAdmin.set(previousSuperAdmin);
                } else {
                    isSuperAdmin.remove();
                }
                
                if (Boolean.TRUE.equals(previousCrossTenantQuery)) {
                    isCrossTenantQuery.set(previousCrossTenantQuery);
                } else {
                    isCrossTenantQuery.remove();
                }
            } else {
                clear();
            }
        }
    }
    
    private static void validateProtectedTenantAccess(String tenantId) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        if (auth == null || !auth.isAuthenticated()) {
            log.debug("Protected tenant access during initialization: {}", tenantId);
            return;
        }
        
        boolean hasSuperAdminRole = auth.getAuthorities().stream()
            .anyMatch(a -> SecurityConstants.ROLE_SUPER_ADMIN.equals(a.getAuthority()));
        
        if (!hasSuperAdminRole) {
            log.error("SECURITY VIOLATION: Attempt to access protected tenant '{}' without authorization. User: {}", 
                tenantId, auth.getName());
            throw new SecurityException(
                "Access to protected tenant '" + tenantId + "' requires super admin privileges");
        }
    }
    
    private static String getCallerClassName() {
        StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
        if (stackTrace.length > 4) {
            return stackTrace[4].getClassName();
        }
        return "Unknown";
    }
}
```

#### 2. TenantResolutionFilter (Excerpt - Key Methods)

**Location**: `coreapplication.security.TenantResolutionFilter`

```java
@Slf4j
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TenantResolutionFilter implements Filter {
    
    @Value("${application.security.jwt.secret-key}")
    private String jwtSecretKey;
    
    private final RefreshTokenService refreshTokenService;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String requestURI = httpRequest.getRequestURI();
        
        // Skip tenant resolution for public endpoints
        if (SecurityConstants.isPublicEndpoint(requestURI)) {
            chain.doFilter(request, response);
            return;
        }
        
        try {
            boolean isAuthEndpoint = SecurityConstants.isAuthEndpoint(requestURI);
            String tenantId;
            
            if (isAuthEndpoint) {
                if (requestURI.contains("/refresh")) {
                    tenantId = extractTenantForRefreshEndpoint(httpRequest);
                } else {
                    tenantId = extractTenantForAuthEndpoint(httpRequest);
                }
            } else {
                tenantId = extractTenantForAuthenticatedEndpoint(httpRequest);
            }
            
            // Validate tenant ID
            if (tenantId == null || !SecurityConstants.isValidTenantFormat(tenantId)) {
                httpResponse.sendError(HttpServletResponse.SC_BAD_REQUEST, 
                    "Invalid or missing tenant identifier");
                return;
            }
            
            // Set tenant context with auto-cleanup
            try (TenantContext.Scope ignored = TenantContext.withTenant(tenantId)) {
                httpResponse.setHeader(SecurityConstants.CURRENT_TENANT_HEADER, tenantId);
                chain.doFilter(request, response);
            }
            
        } catch (SecurityException e) {
            httpResponse.sendError(HttpServletResponse.SC_FORBIDDEN, e.getMessage());
        } catch (Exception e) {
            log.error("Error in tenant resolution: {}", e.getMessage(), e);
            httpResponse.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR, 
                "Tenant resolution failed");
        }
    }
    
    private String extractTenantForAuthenticatedEndpoint(HttpServletRequest request) {
        // Extract from JWT (single source of truth)
        String tenantFromJwt = extractTenantFromJwt(request);
        
        if (tenantFromJwt == null) {
            return null;
        }
        
        // Verify header matches JWT if provided
        String headerTenant = request.getHeader(SecurityConstants.TENANT_HEADER);
        
        if (headerTenant != null && !headerTenant.trim().equals(tenantFromJwt)) {
            log.error("SECURITY VIOLATION: Tenant mismatch - JWT: {}, Header: {}", 
                tenantFromJwt, headerTenant.trim());
            return null;
        }
        
        return tenantFromJwt;
    }
    
    private String extractTenantFromJwt(HttpServletRequest request) {
        try {
            String token = extractJwtToken(request);
            if (token == null) return null;
            
            Claims claims = Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
            
            // Check for super admin
            Boolean isSuperAdmin = claims.get(SecurityConstants.IS_SUPER_ADMIN_CLAIM, Boolean.class);
            if (Boolean.TRUE.equals(isSuperAdmin)) {
                return SecurityConstants.SYSTEM_TENANT;
            }
            
            return claims.get(SecurityConstants.TENANT_ID_CLAIM, String.class);
            
        } catch (Exception e) {
            log.debug("Could not extract tenant from JWT: {}", e.getMessage());
            return null;
        }
    }
}
```

#### 3. SecurityConstants

**Location**: `coreapplication.security.SecurityConstants`

```java
public final class SecurityConstants {
    
    // Tenant Headers
    public static final String TENANT_HEADER = "X-Tenant-Id";
    public static final String CURRENT_TENANT_HEADER = "X-Current-Tenant";
    public static final String AUTHORIZATION_HEADER = "Authorization";
    
    // Special Tenants
    public static final String SYSTEM_TENANT = "_system_";
    public static final String GLOBAL_TENANT = "_global_";
    public static final Set<String> PROTECTED_TENANTS = Set.of(SYSTEM_TENANT, GLOBAL_TENANT);
    
    // JWT Claims
    public static final String TENANT_ID_CLAIM = "tenantId";
    public static final String IS_SUPER_ADMIN_CLAIM = "isSuperAdmin";
    public static final String AUTHORITIES_CLAIM = "authorities";
    
    // Roles
    public static final String ROLE_SUPER_ADMIN = "ROLE_SUPER_ADMIN";
    public static final String ROLE_TENANT_ADMIN = "ROLE_TENANT_ADMIN";
    
    // Public Endpoints (no tenant required)
    private static final Set<String> PUBLIC_ENDPOINTS = Set.of(
        "/swagger-ui",
        "/v3/api-docs",
        "/actuator/health",
        "/health"
    );
    
    // Auth Endpoints (tenant required in header)
    private static final Set<String> AUTH_ENDPOINTS = Set.of(
        "/api/v1/core/auth/login",
        "/api/v1/core/auth/register",
        "/api/v1/core/auth/refresh",
        "/api/v1/core/auth/tenant-lookup"
    );
    
    public static boolean isPublicEndpoint(String uri) {
        return PUBLIC_ENDPOINTS.stream().anyMatch(uri::startsWith);
    }
    
    public static boolean isAuthEndpoint(String uri) {
        return AUTH_ENDPOINTS.stream().anyMatch(uri::contains);
    }
    
    public static boolean isValidTenantFormat(String tenantId) {
        return tenantId != null && tenantId.matches("^[a-z0-9_-]+$");
    }
    
    private SecurityConstants() {
        throw new UnsupportedOperationException("Utility class");
    }
}
```

#### 4. User Service with Tenant Context

```java
@Service
@Slf4j
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    /**
     * Find user by username within current tenant context
     */
    public Optional<User> findByUsername(String username) {
        String tenantId = TenantContext.getTenantId();
        log.debug("Finding user '{}' in tenant '{}'", username, tenantId);
        
        return userRepository.findByUsername(username);
    }
    
    /**
     * Create new user in current tenant
     */
    @Transactional
    public User createUser(UserDTO userDTO) {
        String tenantId = TenantContext.getTenantId();
        
        // Verify user doesn't already exist
        if (userRepository.existsByUsername(userDTO.getUsername())) {
            throw new EntityAlreadyExistException("User already exists: " + userDTO.getUsername());
        }
        
        User user = new User();
        user.setUsername(userDTO.getUsername());
        user.setEmail(userDTO.getEmail());
        user.setPassword(passwordEncoder.encode(userDTO.getPassword()));
        user.setFirstName(userDTO.getFirstName());
        user.setLastName(userDTO.getLastName());
        user.setEnabled(true);
        
        User savedUser = userRepository.save(user);
        
        log.info("User created: {} in tenant: {}", savedUser.getUsername(), tenantId);
        
        return savedUser;
    }
    
    /**
     * Update user - tenant context ensures correct tenant
     */
    @Transactional
    public User updateUser(Long userId, UserUpdateDTO updateDTO) {
        String tenantId = TenantContext.getTenantId();
        
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + userId));
        
        user.setFirstName(updateDTO.getFirstName());
        user.setLastName(updateDTO.getLastName());
        user.setEmail(updateDTO.getEmail());
        user.setPhoneNumber(updateDTO.getPhoneNumber());
        
        User updatedUser = userRepository.save(user);
        
        log.info("User updated: {} in tenant: {}", updatedUser.getUsername(), tenantId);
        
        return updatedUser;
    }
}
```

---

## 6. Security and Authentication

### Authentication Architecture

Lamisplus 3.0 implements stateless JWT-based authentication with refresh tokens and multi-factor authentication (MFA) support.

#### Authentication Flow

```
+----------+                                      +----------------+
|  Client  |                                      |  Auth Service  |
+----+-----+                                      +--------+-------+
     |                                                     |
     | 1. POST /auth/login                                 |
     |    {tenant_id, username, password, remember_me}     |
     +---------------------------------------------------->|
     |                                                     |
     |                              2. Validate credentials|
     |                              3. Check tenant access |
     |                              4. Generate JWT        |
     |                              5. Generate refresh token
     |                              6. Store refresh token |
     |                                                     |
     | 7. Return tokens                                    |
     |    {access_token, refresh_token (cookie)}           |
     |<----------------------------------------------------+
     |                                                     |
     | 8. API Request                                      |
     |    Header: Authorization: Bearer <JWT>              |
     |    Header: X-Tenant-Id: clinic_alpha                |
     +---------------------------------------------------->|
     |                                                     |
     |                              9. Validate JWT        |
     |                              10. Extract tenant_id  |
     |                              11. Verify tenant match|
     |                              12. Process request    |
     |                                                     |
     | 13. Response                                        |
     |<----------------------------------------------------+
     |                                                     |
     | 14. POST /auth/refresh (when JWT expires)           |
     |     Cookie: refreshToken=<token>                    |
     +---------------------------------------------------->|
     |                                                     |
     |                              15. Validate refresh token
     |                              16. Generate new JWT   |
     |                                                     |
     | 17. Return new access token                         |
     |<----------------------------------------------------+
```

### JWT Structure

**Header:**
```json
{
  "alg": "HS512",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "sub": "john.doe@clinic-alpha.com",
  "tenantId": "clinic_alpha",
  "userId": "12345",
  "isSuperAdmin": false,
  "authorities": [
    "ROLE_TENANT_ADMIN",
    "ROLE_LAB_MANAGER"
  ],
  "permissions": [
    "READ_USERS",
    "WRITE_USERS",
    "READ_PATIENTS",
    "WRITE_LAB_TESTS",
    "ENABLE_PLUGINS"
  ],
  "iat": 1729900000,
  "exp": 1729901800
}
```

**Key Claims:**
- `sub`: Username (subject)
- `tenantId`: Tenant identifier for data isolation
- `userId`: Unique user ID within tenant
- `isSuperAdmin`: Boolean flag for super admin users
- `authorities`: Spring Security roles
- `permissions`: Granular permission list
- `iat`: Issued at timestamp
- `exp`: Expiration timestamp (30 minutes by default)

### Security Configuration

**SecurityConfig.java:**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;
    
    @Autowired
    private TenantResolutionFilter tenantResolutionFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/actuator/health", "/health").permitAll()
                .requestMatchers("/api/v1/core/auth/**").permitAll()
                
                // Super admin endpoints
                .requestMatchers("/api/v1/core/super-admin/**").hasRole("SUPER_ADMIN")
                
                // Tenant admin endpoints
                .requestMatchers("/api/v1/core/tenant-admin/**").hasRole("TENANT_ADMIN")
                
                // All other requests require authentication
                .anyRequest().authenticated()
            )
            .addFilterBefore(tenantResolutionFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList("http://localhost:3000"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setExposedHeaders(Arrays.asList("Authorization", "X-Current-Tenant"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        
        return source;
    }
}
```

### Authorization

#### Role-Based Access Control (RBAC)

**Predefined Roles:**

| Role | Description | Scope |
|------|-------------|-------|
| **ROLE_SUPER_ADMIN** | System administrator with global access | Cross-tenant |
| **ROLE_TENANT_ADMIN** | Tenant administrator | Single tenant |
| **ROLE_LAB_MANAGER** | Laboratory manager | Single tenant, lab plugin |
| **ROLE_LAB_TECH** | Laboratory technician | Single tenant, lab plugin |
| **ROLE_PHARMACIST** | Pharmacy staff | Single tenant, pharmacy plugin |
| **ROLE_RECEPTIONIST** | Front desk staff | Single tenant |
| **ROLE_VIEWER** | Read-only access | Single tenant |

#### Method-Level Security

```java
@RestController
@RequestMapping("/api/v1/core/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    /**
     * Only users with READ_USERS permission can access
     */
    @GetMapping
    @PreAuthorize("hasAuthority('READ_USERS')")
    public ResponseEntity<Page<UserDTO>> getUsers(Pageable pageable) {
        Page<UserDTO> users = userService.findAll(pageable);
        return ResponseEntity.ok(users);
    }
    
    /**
     * Requires WRITE_USERS permission and tenant admin role
     */
    @PostMapping
    @PreAuthorize("hasAuthority('WRITE_USERS') and hasRole('TENANT_ADMIN')")
    public ResponseEntity<UserDTO> createUser(@RequestBody @Valid UserRequest request) {
        UserDTO user = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
    
    /**
     * Only super admins can delete users
     */
    @DeleteMapping("/{userId}")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long userId) {
        userService.deleteUser(userId);
        return ResponseEntity.noContent().build();
    }
}
```

#### Custom Permission Evaluator

```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {
    
    @Autowired
    private UserPermissionRepository permissionRepository;
    
    @Override
    public boolean hasPermission(Authentication authentication, Object targetDomainObject, 
                                Object permission) {
        if (authentication == null || !(permission instanceof String)) {
            return false;
        }
        
        String permissionName = (String) permission;
        String username = authentication.getName();
        String tenantId = TenantContext.getTenantId();
        
        return permissionRepository.userHasPermission(username, tenantId, permissionName);
    }
    
    @Override
    public boolean hasPermission(Authentication authentication, Serializable targetId, 
                                String targetType, Object permission) {
        return hasPermission(authentication, null, permission);
    }
}
```

### Cross-Tenant Access Prevention

#### Aspect-Based Tenant Filtering

```java
@Aspect
@Component
@Slf4j
public class TenantFilterAspect {
    
    /**
     * Intercept repository methods to ensure tenant context
     */
    @Before("execution(* coreapplication.repository..*.*(..)) && " +
            "!@annotation(coreapplication.repository.TenantAgnostic)")
    public void validateTenantContext(JoinPoint joinPoint) {
        
        if (!TenantContext.isSet()) {
            throw new IllegalStateException(
                "Tenant context not set for repository operation: " + 
                joinPoint.getSignature().getName());
        }
        
        if (TenantContext.isSuperAdminMode()) {
            log.debug("Super admin mode active, bypassing tenant filter");
            return;
        }
        
        String tenantId = TenantContext.getTenantId();
        log.debug("Repository operation with tenant: {}", tenantId);
    }
}
```

#### Service-Level Validation

```java
@Service
public class PatientService {
    
    @Autowired
    private PatientRepository patientRepository;
    
    public Patient findById(Long patientId) {
        String tenantId = TenantContext.getTenantId();
        
        // Repository query automatically scoped to tenant schema
        Optional<Patient> patient = patientRepository.findById(patientId);
        
        if (patient.isEmpty()) {
            log.warn("Patient {} not found in tenant {}", patientId, tenantId);
            throw new ResourceNotFoundException("Patient not found");
        }
        
        return patient.get();
    }
}
```

### Audit Logging

Every security-relevant action is logged to the `audit_events` table:

```java
@Service
@Slf4j
public class AuditService {
    
    @Autowired
    private AuditEventRepository auditRepository;
    
    @Async
    public void logAuditEvent(AuditEvent event) {
        String tenantId = TenantContext.getTenantId();
        
        event.setTenantId(tenantId);
        event.setCreatedDate(LocalDateTime.now());
        
        auditRepository.save(event);
        
        log.info("Audit: [{}] {} by {} in tenant {}", 
            event.getEventType(), event.getAction(), 
            event.getUsername(), tenantId);
    }
    
    public void logLoginAttempt(String username, boolean success, String ipAddress) {
        AuditEvent event = AuditEvent.builder()
            .eventType("AUTHENTICATION")
            .action(success ? "LOGIN_SUCCESS" : "LOGIN_FAILED")
            .username(username)
            .ipAddress(ipAddress)
            .build();
        
        logAuditEvent(event);
    }
    
    public void logUserCreated(User user, String createdBy) {
        AuditEvent event = AuditEvent.builder()
            .eventType("USER_MANAGEMENT")
            .entityType("User")
            .entityId(user.getId().toString())
            .action("CREATE")
            .username(createdBy)
            .details(Map.of("username", user.getUsername(), "email", user.getEmail()))
            .build();
        
        logAuditEvent(event);
    }
}
```

---

## 7. Data Management

### Data Isolation

Lamisplus 3.0 ensures complete data segregation through multiple layers:

**1. Schema-Level Isolation**
- Each tenant's data in separate PostgreSQL schema
- Connection pool automatically routes to correct schema
- SQL injection across schemas prevented

**2. Application-Level Validation**
- TenantContext required for all data operations
- AOP aspects enforce tenant context presence
- Repository layer validates tenant before queries

**3. API-Level Protection**
- JWT token contains tenant ID
- Header validation ensures tenant match
- Cross-tenant requests rejected at filter level

### Tenant Migrations

When deploying a new version with schema changes, migrations must be applied to all tenant schemas.

#### Migration Execution Service

```java
@Service
@Slf4j
public class TenantMigrationService {
    
    @Autowired
    private TenantRepository tenantRepository;
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private LiquibasePluginHelper liquibaseHelper;
    
    /**
     * Apply pending migrations to all active tenants
     */
    public void migrateAllTenants() {
        List<Tenant> activeTenants = tenantRepository.findByStatus("ACTIVE");
        
        log.info("Starting migration for {} active tenants", activeTenants.size());
        
        for (Tenant tenant : activeTenants) {
            try {
                migrateTenant(tenant);
                log.info("✓ Migration successful for tenant: {}", tenant.getTenantId());
            } catch (Exception e) {
                log.error("✗ Migration failed for tenant: {}", tenant.getTenantId(), e);
                // Send alert to administrators
                sendMigrationFailureAlert(tenant, e);
            }
        }
    }
    
    private void migrateTenant(Tenant tenant) throws Exception {
        String schemaName = tenant.getSchemaName();
        
        Database database = DatabaseFactory.getInstance()
            .findCorrectDatabaseImplementation(
                new JdbcConnection(dataSource.getConnection()));
        
        database.setDefaultSchemaName(schemaName);
        database.setLiquibaseSchemaName(schemaName);
        
        Liquibase liquibase = new Liquibase(
            "db/changelog/tenant/tenant-schema.xml",
            new ClassLoaderResourceAccessor(),
            database
        );
        
        // Check for pending changes
        List<ChangeSet> unrunChangeSets = liquibase.listUnrunChangeSets(null);
        
        if (unrunChangeSets.isEmpty()) {
            log.debug("No pending migrations for tenant: {}", tenant.getTenantId());
            return;
        }
        
        log.info("Applying {} migrations to tenant: {}", 
            unrunChangeSets.size(), tenant.getTenantId());
        
        // Execute migrations
        liquibase.update((String) null);
        
        // Record migration
        recordMigrationHistory(tenant, unrunChangeSets);
    }
    
    /**
     * Rollback last migration for a tenant (emergency use)
     */
    public void rollbackTenantMigration(String tenantId, int count) {
        Tenant tenant = tenantRepository.findByTenantId(tenantId)
            .orElseThrow(() -> new TenantNotFoundException(tenantId));
        
        String schemaName = tenant.getSchemaName();
        
        try {
            Database database = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(
                    new JdbcConnection(dataSource.getConnection()));
            
            database.setDefaultSchemaName(schemaName);
            
            Liquibase liquibase = new Liquibase(
                "db/changelog/tenant/tenant-schema.xml",
                new ClassLoaderResourceAccessor(),
                database
            );
            
            liquibase.rollback(count, null);
            
            log.warn("Rolled back {} migrations for tenant: {}", count, tenantId);
            
        } catch (Exception e) {
            log.error("Rollback failed for tenant: {}", tenantId, e);
            throw new RuntimeException("Migration rollback failed", e);
        }
    }
}
```

### Backups and Recovery

#### Automated Backup Strategy

**Backup Schedule:**
- **Full Backup**: Daily at 2 AM (off-peak)
- **Differential Backup**: Every 6 hours
- **WAL Archiving**: Continuous

**PostgreSQL Backup Script** (`tenant-backup.sh`):

```bash
#!/bin/bash
# Tenant-specific schema backup

TENANT_ID=$1
SCHEMA_NAME=$2
BACKUP_DIR="/backups/tenants/${TENANT_ID}"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_DIR}/${SCHEMA_NAME}_${TIMESTAMP}.dump"

# Create backup directory
mkdir -p ${BACKUP_DIR}

# Backup schema
pg_dump \
  --host=${DB_HOST} \
  --port=5432 \
  --username=${DB_USER} \
  --schema=${SCHEMA_NAME} \
  --format=custom \
  --file=${BACKUP_FILE} \
  ${DB_NAME}

# Verify backup
pg_restore --list ${BACKUP_FILE} > /dev/null 2>&1

if [ $? -eq 0 ]; then
    echo "✓ Backup successful: ${BACKUP_FILE}"
    
    # Compress backup
    gzip ${BACKUP_FILE}
    
    # Upload to S3
    aws s3 cp ${BACKUP_FILE}.gz \
        s3://lamisplus-backups/tenants/${TENANT_ID}/ \
        --storage-class STANDARD_IA
    
    # Cleanup old local backups (keep last 7 days)
    find ${BACKUP_DIR} -name "*.dump.gz" -mtime +7 -delete
    
    echo "✓ Backup uploaded to S3"
else
    echo "✗ Backup verification failed for ${TENANT_ID}"
    exit 1
fi
```

**Automated Backup Service:**

```java
@Service
@Slf4j
public class TenantBackupService {
    
    @Autowired
    private TenantRepository tenantRepository;
    
    @Value("${application.backup.directory}")
    private String backupDirectory;
    
    @Scheduled(cron = "0 0 2 * * ?")  // Daily at 2 AM
    public void backupAllTenants() {
        List<Tenant> tenants = tenantRepository.findByStatus("ACTIVE");
        
        log.info("Starting backup for {} tenants", tenants.size());
        
        tenants.parallelStream().forEach(tenant -> {
            try {
                backupTenant(tenant);
            } catch (Exception e) {
                log.error("Backup failed for tenant: {}", tenant.getTenantId(), e);
                sendBackupFailureAlert(tenant, e);
            }
        });
    }
    
    private void backupTenant(Tenant tenant) throws Exception {
        String command = String.format(
            "bash /scripts/tenant-backup.sh %s %s",
            tenant.getTenantId(),
            tenant.getSchemaName()
        );
        
        Process process = Runtime.getRuntime().exec(command);
        int exitCode = process.waitFor();
        
        if (exitCode != 0) {
            throw new BackupException("Backup failed for tenant: " + tenant.getTenantId());
        }
        
        // Record backup metadata
        BackupRecord record = BackupRecord.builder()
            .tenantId(tenant.getTenantId())
            .schemaName(tenant.getSchemaName())
            .backupDate(LocalDateTime.now())
            .status("SUCCESS")
            .build();
        
        backupRecordRepository.save(record);
        
        log.info("✓ Backup completed for tenant: {}", tenant.getTenantId());
    }
}
```

#### Recovery Procedures

**Full Schema Restore:**

```bash
#!/bin/bash
# Restore tenant schema from backup

TENANT_ID=$1
BACKUP_FILE=$2
SCHEMA_NAME="tenant_${TENANT_ID}"

echo "Restoring tenant: ${TENANT_ID} from ${BACKUP_FILE}"

# Download from S3 if needed
if [[ ${BACKUP_FILE} == s3://* ]]; then
    LOCAL_FILE="/tmp/$(basename ${BACKUP_FILE})"
    aws s3 cp ${BACKUP_FILE} ${LOCAL_FILE}
    BACKUP_FILE=${LOCAL_FILE}
fi

# Decompress if needed
if [[ ${BACKUP_FILE} == *.gz ]]; then
    gunzip ${BACKUP_FILE}
    BACKUP_FILE=${BACKUP_FILE%.gz}
fi

# Drop existing schema (DANGEROUS - confirm first!)
read -p "Drop existing schema ${SCHEMA_NAME}? (yes/no): " confirm
if [ "$confirm" == "yes" ]; then
    psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} <<EOF
    DROP SCHEMA IF EXISTS ${SCHEMA_NAME} CASCADE;
    CREATE SCHEMA ${SCHEMA_NAME};
EOF
fi

# Restore from backup
pg_restore \
  --host=${DB_HOST} \
  --port=5432 \
  --username=${DB_USER} \
  --dbname=${DB_NAME} \
  --schema=${SCHEMA_NAME} \
  --clean \
  --if-exists \
  ${BACKUP_FILE}

if [ $? -eq 0 ]; then
    echo "✓ Restore completed successfully"
else
    echo "✗ Restore failed"
    exit 1
fi
```

### Data Retention Policies

Each tenant can configure custom data retention policies:

```java
@Entity
@Table(name = "data_retention_policies", schema = "_system_")
public class DataRetentionPolicy extends BaseEntity {
    
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
    
    @Column(name = "entity_type", nullable = false)
    private String entityType;  // e.g., "AUDIT_EVENTS", "REFRESH_TOKENS"
    
    @Column(name = "retention_days", nullable = false)
    private Integer retentionDays;
    
    @Column(name = "archive_enabled")
    private Boolean archiveEnabled = false;
    
    @Column(name = "archive_location")
    private String archiveLocation;
    
    @Column(name = "purge_enabled")
    private Boolean purgeEnabled = true;
}
```

**Data Purge Job:**

```java
@Service
@Slf4j
public class DataPurgeService {
    
    @Scheduled(cron = "0 0 3 * * ?")  // Daily at 3 AM
    public void purgeExpiredData() {
        List<DataRetentionPolicy> policies = retentionPolicyRepository.findAll();
        
        for (DataRetentionPolicy policy : policies) {
            try {
                try (TenantContext.Scope scope = TenantContext.withTenant(policy.getTenantId())) {
                    purgeDataForPolicy(policy);
                }
            } catch (Exception e) {
                log.error("Purge failed for policy: {}", policy.getId(), e);
            }
        }
    }
    
    private void purgeDataForPolicy(DataRetentionPolicy policy) {
        LocalDateTime cutoffDate = LocalDateTime.now()
            .minusDays(policy.getRetentionDays());
        
        switch (policy.getEntityType()) {
            case "AUDIT_EVENTS":
                purgeAuditEvents(cutoffDate, policy.getArchiveEnabled());
                break;
            case "REFRESH_TOKENS":
                purgeRefreshTokens(cutoffDate);
                break;
            default:
                log.warn("Unknown entity type for purge: {}", policy.getEntityType());
        }
    }
    
    private void purgeAuditEvents(LocalDateTime cutoffDate, boolean archive) {
        if (archive) {
            // Archive before deleting
            List<AuditEvent> events = auditRepository.findByCreatedDateBefore(cutoffDate);
            archiveService.archiveAuditEvents(events);
        }
        
        int deleted = auditRepository.deleteByCreatedDateBefore(cutoffDate);
        log.info("Purged {} audit events older than {}", deleted, cutoffDate);
    }
}
```

---

## 8. Deployment & DevOps

### CI/CD Pipeline

Lamisplus 3.0 uses GitHub Actions for automated building, testing, and deployment.

**GitHub Actions Workflow** (`.github/workflows/build.yml`):

```yaml
name: Lamisplus CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: lamisplus/core

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_DB: core_database_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run unit tests
        run: cd lamisplus-core/backend && mvn clean test
      
      - name: Run integration tests
        run: cd lamisplus-core/backend && mvn verify -P integration-tests
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: core_database_test
          DB_USERNAME: test_user
          DB_PASSWORD: test_password
          REDIS_HOST: localhost
          REDIS_PORT: 6379
      
      - name: Build application
        run: cd lamisplus-core/backend && mvn clean package -DskipTests
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: lamisplus-jar
          path: lamisplus-core/backend/target/*.jar
  
  build-docker-image:
    needs: build-and-test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: lamisplus-jar
          path: lamisplus-core/backend/target/
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.DOCKER_REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: ./lamisplus-core/backend
          push: true
          tags: |
            ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=registry,ref=${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
  
  deploy-staging:
    needs: build-docker-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Deploy to Staging
        run: |
          echo "Deploying to staging environment..."
          # Add deployment commands here
  
  deploy-production:
    needs: build-docker-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Deploy to Production
        run: |
          echo "Deploying to production environment..."
          # Add deployment commands here
```

### Docker Configuration

**Dockerfile:**

```dockerfile
FROM eclipse-temurin:21-jre-alpine

# Add application user
RUN addgroup -S lamisplus && adduser -S lamisplus -G lamisplus

# Set working directory
WORKDIR /app

# Copy application JAR
COPY target/core-application-*.jar app.jar

# Create directories for plugins and logs
RUN mkdir -p /app/plugins /app/logs /app/plugin-storage && \
    chown -R lamisplus:lamisplus /app

# Switch to application user
USER lamisplus

# Expose application port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Run application
ENTRYPOINT ["java", \
            "-XX:+UseContainerSupport", \
            "-XX:MaxRAMPercentage=75.0", \
            "-Djava.security.egd=file:/dev/./urandom", \
            "-jar", "app.jar"]
```

**Docker Compose** (`docker-compose.yml`):

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: core_database
      POSTGRES_USER: lamisplus_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lamisplus_user"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - lamisplus-network
  
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - lamisplus-network
  
  lamisplus-core:
    build:
      context: ./lamisplus-core/backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-production}
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: core_database
      DB_USERNAME: lamisplus_user
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      JWT_SECRET: ${JWT_SECRET}
      COOKIE_SECURE: true
    volumes:
      - ./plugins:/app/plugins
      - ./logs:/app/logs
      - ./plugin-storage:/app/plugin-storage
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - lamisplus-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  lamisplus-network:
    driver: bridge
```

### Tenant Provisioning

#### Automated Tenant Creation API

**TenantAdminController:**

```java
@RestController
@RequestMapping("/api/v1/core/super-admin/tenants")
@PreAuthorize("hasRole('SUPER_ADMIN')")
@Slf4j
public class SuperAdminController {
    
    @Autowired
    private SuperAdminService superAdminService;
    
    @PostMapping
    public ResponseEntity<TenantAdminCreationResponse> createTenant(
            @RequestBody @Valid CreateTenantAdminRequest request) {
        
        log.info("Creating new tenant: {}", request.getTenantId());
        
        TenantAdminCreationResponse response = superAdminService.createTenantWithAdmin(request);
        
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    @PutMapping("/{tenantId}/suspend")
    public ResponseEntity<Void> suspendTenant(@PathVariable String tenantId) {
        superAdminService.suspendTenant(tenantId);
        return ResponseEntity.noContent().build();
    }
    
    @PutMapping("/{tenantId}/activate")
    public ResponseEntity<Void> activateTenant(@PathVariable String tenantId) {
        superAdminService.activateTenant(tenantId);
        return ResponseEntity.noContent().build();
    }
}
```

**Request/Response:**

```json
// POST /api/v1/core/super-admin/tenants
{
  "tenantId": "clinic_delta",
  "tenantName": "Delta Medical Clinic",
  "adminUsername": "admin",
  "adminEmail": "admin@delta-clinic.com",
  "adminPassword": "SecurePassword123!",
  "organizationName": "Delta Healthcare Group",
  "contactEmail": "info@delta-clinic.com",
  "contactPhone": "+1-555-0100"
}

// Response
{
  "tenantId": "clinic_delta",
  "schemaName": "tenant_clinic_delta",
  "adminUserId": 1,
  "adminUsername": "admin",
  "temporaryPassword": "SecurePassword123!",
  "status": "ACTIVE",
  "createdDate": "2025-10-25T10:30:00Z"
}
```

### Monitoring and Logging

#### Actuator Endpoints

**Exposed Endpoints:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,circuitbreakers,ratelimiters
```

**Health Check Response:**

```http
GET /actuator/health HTTP/1.1
Authorization: Bearer <admin-token>

{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.12"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000000,
        "free": 350000000000,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

#### Centralized Logging

**Logging Pattern with Tenant Context:**

```yaml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] [TENANT:%X{tenantId}] %-5level %logger{36} - %msg%n"
```

**Example Log Output:**

```
2025-10-25 10:45:30 [http-nio-8080-exec-1] [TENANT:clinic_alpha] INFO  UserService - User created: john.doe
2025-10-25 10:45:31 [http-nio-8080-exec-2] [TENANT:clinic_alpha] DEBUG TenantContext - Tenant context set: clinic_alpha
2025-10-25 10:45:32 [http-nio-8080-exec-3] [TENANT:hospital_beta] INFO  AuthService - User logged in: admin
```

---

## 9. Testing Strategy

### Unit Testing

**Tenant Context Testing:**

```java
@SpringBootTest
@ActiveProfiles("test")
class TenantContextTest {
    
    @Test
    void testTenantContextIsolation() {
        // Set tenant A
        TenantContext.setTenantId("tenant_a");
        assertEquals("tenant_a", TenantContext.getTenantId());
        
        // Clear context
        TenantContext.clear();
        assertThrows(IllegalStateException.class, TenantContext::getTenantId);
        
        // Set tenant B
        TenantContext.setTenantId("tenant_b");
        assertEquals("tenant_b", TenantContext.getTenantId());
    }
    
    @Test
    void testScopeAutoClose() {
        TenantContext.setTenantId("original_tenant");
        
        try (TenantContext.Scope scope = TenantContext.withTenant("temp_tenant")) {
            assertEquals("temp_tenant", TenantContext.getTenantId());
        }
        
        // Context should be restored
        assertEquals("original_tenant", TenantContext.getTenantId());
    }
}
```

### Integration Testing

**Multi-Tenant Data Isolation Test:**

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Sql(scripts = "/test-data/setup-test-tenants.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/test-data/cleanup-test-tenants.sql", executionPhase = AFTER_TEST_METHOD)
class TenantIsolationIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testDataIsolationBetweenTenants() {
        // Create user in Tenant A
        HttpHeaders headersA = new HttpHeaders();
        headersA.set("X-Tenant-Id", "tenant_a");
        headersA.set("Authorization", "Bearer " + getTokenForTenant("tenant_a"));
        
        UserRequest userRequest = new UserRequest();
        userRequest.setUsername("testuser");
        userRequest.setEmail("test@tenant-a.com");
        userRequest.setPassword("Password123!");
        
        HttpEntity<UserRequest> requestA = new HttpEntity<>(userRequest, headersA);
        
        ResponseEntity<UserDTO> responseA = restTemplate.postForEntity(
            "/api/v1/core/users",
            requestA,
            UserDTO.class
        );
        
        assertEquals(HttpStatus.CREATED, responseA.getStatusCode());
        UserDTO userA = responseA.getBody();
        assertNotNull(userA.getId());
        
        // Try to access user from Tenant B
        HttpHeaders headersB = new HttpHeaders();
        headersB.set("X-Tenant-Id", "tenant_b");
        headersB.set("Authorization", "Bearer " + getTokenForTenant("tenant_b"));
        
        HttpEntity<Void> requestB = new HttpEntity<>(headersB);
        
        ResponseEntity<UserDTO> responseB = restTemplate.exchange(
            "/api/v1/core/users/" + userA.getId(),
            HttpMethod.GET,
            requestB,
            UserDTO.class
        );
        
        // Should return 404 as user doesn't exist in Tenant B's schema
        assertEquals(HttpStatus.NOT_FOUND, responseB.getStatusCode());
    }
}
```

---

## 10. API Documentation

### Authentication Endpoints

#### POST /api/v1/core/auth/login

**Request:**
```http
POST /api/v1/core/auth/login HTTP/1.1
Host: api.lamisplus.com
Content-Type: application/json
X-Tenant-Id: clinic_alpha

{
  "username": "john.doe@clinic-alpha.com",
  "password": "SecurePassword123!",
  "rememberMe": false
}
```

**Response (200 OK):**
```json
{
  "accessToken": "eyJhbGciOiJIUzUxMiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 1800,
  "user": {
    "id": 12345,
    "username": "john.doe@clinic-alpha.com",
    "email": "john.doe@clinic-alpha.com",
    "firstName": "John",
    "lastName": "Doe",
    "roles": ["ROLE_LAB_TECH"],
    "permissions": ["READ_PATIENTS", "WRITE_LAB_TESTS"],
    "tenantId": "clinic_alpha"
  }
}
```

### User Management Endpoints

#### GET /api/v1/core/users

**Request:**
```http
GET /api/v1/core/users?page=0&size=20&sort=lastName,asc HTTP/1.1
Host: api.lamisplus.com
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
X-Tenant-Id: clinic_alpha
```

**Response (200 OK):**
```json
{
  "content": [
    {
      "id": 1,
      "username": "john.doe",
      "email": "john.doe@clinic-alpha.com",
      "firstName": "John",
      "lastName": "Doe",
      "enabled": true,
      "roles": ["ROLE_LAB_TECH"]
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": {
      "sorted": true
    }
  },
  "totalElements": 150,
  "totalPages": 8
}
```

### Error Responses

**Standard Error Format:**

```json
{
  "timestamp": "2025-10-25T10:45:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Invalid tenant identifier",
  "tenantId": "invalid_tenant",
  "path": "/api/v1/core/users"
}
```

---

## 11. Error Handling & Logging

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ExceptionResponse> handleResourceNotFound(
            ResourceNotFoundException ex,
            WebRequest request) {
        
        ExceptionResponse error = ExceptionResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message(ex.getMessage())
            .tenantId(TenantContext.getTenantIdOrNull())
            .path(getRequestPath(request))
            .build();
        
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    @ExceptionHandler(SecurityException.class)
    public ResponseEntity<ExceptionResponse> handleSecurityException(
            SecurityException ex,
            WebRequest request) {
        
        log.error("Security exception: {}", ex.getMessage());
        
        ExceptionResponse error = ExceptionResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.FORBIDDEN.value())
            .error("Forbidden")
            .message("Access denied")
            .tenantId(TenantContext.getTenantIdOrNull())
            .path(getRequestPath(request))
            .build();
        
        return new ResponseEntity<>(error, HttpStatus.FORBIDDEN);
    }
}
```

---

## 12. Scaling & Performance

### Connection Pooling (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 40
      minimum-idle: 8
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

### Caching Strategy

**Redis Configuration:**

```java
@Configuration
@EnableCaching
public class RedisConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

### Circuit Breaker (Resilience4j)

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
```

---

## 13. Tenant Lifecycle Management

### Tenant States

| State | Description | Actions Allowed |
|-------|-------------|-----------------|
| **PROVISIONING** | Schema being created | System operations only |
| **ACTIVE** | Fully operational | All operations |
| **SUSPENDED** | Temporarily disabled | Read-only for admins |
| **DELETED** | Permanently removed | None |

---

## 14. Appendices

### Appendix A: Glossary

| Term | Definition |
|------|------------|
| **Tenant** | An independent healthcare organization using the system |
| **Schema** | PostgreSQL schema containing tenant-specific tables |
| **Plugin** | Hot-loadable module providing specific functionality |
| **JWT** | JSON Web Token for stateless authentication |
| **RBAC** | Role-Based Access Control for authorization |

### Appendix B: References

- Spring Boot Documentation: https://spring.io/projects/spring-boot
- PostgreSQL Multi-Schema: https://www.postgresql.org/docs/current/ddl-schemas.html
- Liquibase: https://docs.liquibase.com/
- Resilience4j: https://resilience4j.readme.io/

---

## Conclusion

This technical documentation provides comprehensive coverage of the Lamisplus 3.0 multi-tenant healthcare platform architecture, implementation details, and operational procedures. The system is designed to scale efficiently while maintaining strict data isolation between tenants, ensuring security, compliance, and optimal performance.

For questions or support, please contact the Lamisplus development team.

**Document Version**: 1.0  
**Last Updated**: October 25, 2025
