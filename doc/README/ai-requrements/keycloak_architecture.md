# Архитектура аутентификации с использованием Keycloak

## 1. Обзор решения

### 1.1. Цель
Интеграция Keycloak как центрального сервиса управления идентификацией и доступом (IAM) для обеспечения единой точки аутентификации, авторизации и управления пользователями.

### 1.2. Ключевые преимущества
- **Единый источник истины** для пользовательских данных и ролей
- **Поддержка стандартов** OAuth 2.0, OpenID Connect, SAML 2.0
- **Готовые функции**: многофакторная аутентификация, социальные логины, self-service
- **Централизованное управление** клиентами, ролями, политиками
- **Масштабируемость** через кластеризацию и репликацию

### 1.3. Область применения
- Аутентификация пользователей через веб и мобильные приложения
- Авторизация API на основе ролей (RBAC)
- Федерация идентификации с внешними провайдерами (Google, GitHub, LDAP)
- Управление сессиями и single sign-on (SSO)
- Аудит событий безопасности

## 2. Архитектурные решения

### 2.1. Выбор Keycloak vs кастомное решение
| Критерий | Keycloak | Кастомное решение |
|----------|----------|-------------------|
| Время разработки | Несколько дней | Несколько месяцев |
| Поддержка стандартов | Полная (OIDC, OAuth2, SAML) | Требует реализации |
| Безопасность | Протестировано сообществом | Риск уязвимостей |
| Масштабируемость | Готовая кластеризация | Требует разработки |
| Тотальная стоимость владения | Низкая (open source) | Высокая (разработка+поддержка) |

**Решение:** Использовать Keycloak как готовое решение, сфокусировавшись на интеграции, а не реализации базовых функций IAM.

### 2.2. Модель развертывания
- **Контейнеризация**: Keycloak в Docker с официальным образом `quay.io/keycloak/keycloak`
- **Оркестрация**: Kubernetes StatefulSet для stateful компонентов
- **База данных**: PostgreSQL с репликацией (внешняя от Keycloak)
- **Кэширование**: Infinispan embedded для распределенного кэша сессий
- **Балансировка нагрузки**: Ingress с TLS termination

### 2.3. Модель данных Keycloak
Keycloak использует собственную схему данных с сущностями:
- **Realms**: Изолированные пространства для клиентов (например, "production", "staging")
- **Clients**: Приложения, которые используют аутентификацию (SPA, мобильное приложение, backend API)
- **Users**: Пользователи с атрибутами, учетными данными и ролями
- **Roles**: Глобальные и client-specific роли
- **Groups**: Иерархические группы пользователей
- **Identity Providers**: Внешние провайдеры (Google, LDAP, SAML)

## 3. Диаграмма контейнеров (C4 Level 2)

Создам PlantUML диаграмму контейнеров.

<write_to_file>
<path>c4_Level_2_containers_diagram_KeycloakAuth_v1.plantuml</path>
<content>
@startuml
!include <C4/C4_Container>

Title Container Diagram - Authentication with Keycloak

Person(user, "End User", "Uses web or mobile application")
Person(admin, "Administrator", "Manages users and roles via admin UI")

System_Boundary(auth_system, "Authentication System") {
    Container(spa, "Single Page Application", "JavaScript", "React/Vue/Angular application")
    Container(mobile_app, "Mobile Application", "iOS/Android", "Native mobile app")
    Container(backend_api, "Backend API", "Java/Node.js/Go", "Business logic microservices")
    
    Container(keycloak, "Keycloak Server", "Java", "Identity and Access Management")
    ContainerDb(keycloak_db, "Keycloak Database", "PostgreSQL", "Stores realms, users, clients, roles")
    Container(keycloak_cache, "Session Cache", "Infinispan", "Distributed session storage")
    
    Container(ldap, "LDAP/Active Directory", "External Directory", "Corporate user directory")
}

System_Ext(google_auth, "Google OAuth", "External identity provider")
System_Ext(email_service, "Email Service", "Sends verification/password reset emails")

Rel(user, spa, "Uses", "HTTPS")
Rel(user, mobile_app, "Uses", "HTTPS")
Rel(spa, keycloak, "Redirects for authentication", "OpenID Connect")
Rel(mobile_app, keycloak, "Authenticates via", "OAuth 2.0 Authorization Code PKCE")
Rel(backend_api, keycloak, "Validates tokens", "JWT validation via introspection")
Rel(keycloak, keycloak_db, "Reads/writes user data", "JDBC")
Rel(keycloak, keycloak_cache, "Stores sessions", "Hot Rod protocol")
Rel(keycloak, ldap, "Synchronizes users", "LDAP protocol")
Rel(keycloak, google_auth, "Federates authentication", "OAuth 2.0")
Rel(keycloak, email_service, "Sends emails", "SMTP")
Rel(admin, keycloak, "Manages via", "Admin Console/API")

SHOW_LEGEND()

@enduml