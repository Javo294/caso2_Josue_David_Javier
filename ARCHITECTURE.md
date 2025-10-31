# Arquitectura del sistema — Entregable 2

**Resumen:** Este documento describe la arquitectura global, decisiones, patrones, integración con proveedor cloud (Azure), uso de Pub/Sub, capas, seguridad, y vinculación con artefactos de código. Cubre también integración opcional con Supabase/Vercel.

---

## 1.  Diagrama general de todos los sistemas o subempresas [TODO]
> *Este punto corresponde a:* “Crear un diagrama general de todos los sistemas o subempresas, o diagramas individuales y luego un diagrama de alto nivel que integre todo.”

Aquí se muestra un diagrama de arquitectura donde se integran todas las subempresas o dominios principales: **PromptContent**, **PromptAds**, **PromptCRM** y **PromptSales**. Cada uno se representa como un microservicio independiente (siguiendo la relación 1:1 entre dominio y microservicio).

**Ver imagen del diagrama principal:**  
`architecture/diagrams/high-level-architecture.png`  

Este diagrama incluye:
- Capas (Presentation, Application, Domain, Infrastructure)
- Componentes cloud (Azure DB, Event Hubs, Redis, KeyVault, etc.)
- Flujos principales entre los sistemas
- API Gateway y conexiones entre microservicios

---

## 2. Patrones de arquitectura usados
> *Corresponde al ítem:* “Documentar patrones de arquitectura usados, cómo y dónde se aplican.”

Los patrones principales usados son:

| Patrón | Dónde se aplica | Propósito |
|--------|------------------|------------|
| **Domain-Driven Design (DDD)** | En la organización del código de cada microservicio | Mantener el dominio limpio y separado |
| **Event-Driven Design (Pub/Sub)** | En los eventos `content.generated`, `campaign.executed`, `notification.sent` | Desacoplar procesos asíncronos |
| **API Gateway / Facade** | En el enrutamiento principal | Centraliza autenticación y control de versiones |
| **Anti-Corruption Layer (ACL)** | Entre dominios diferentes | Evita contaminación de modelos |
| **Dependency Injection** | En los controladores y servicios | Facilita testing e intercambiar implementaciones |
| **Saga / Orchestration** | En campañas publicitarias | Manejo de flujos complejos con compensaciones |

---

## 3. Comunicación entre servicios
> *Corresponde al ítem:* “Definir si se usará REST o GraphQL, serverless o servidores dedicados, lenguajes de programación y microservicios.”

- Se usará **REST** como contrato principal (OpenAPI 3.0).  
- **GraphQL** solo se aplicará en el frontend como *BFF opcional* para optimizar consultas.
- El despliegue será **en contenedores dentro de Azure AKS** (no serverless completo, aunque algunos jobs pequeños sí).  
- Lenguajes principales: **Node.js (backend)** y **Python (workers/eventos)**.  
- Arquitectura de **microservicios independientes**, no monolito.

---

## 4. Convivencia entre microservicios y DDD
> *Corresponde al ítem:* “Justificar cómo conviven microservicios en caso de que se usen y domain driven design.”

Cada dominio principal se convierte en un microservicio. Por ejemplo:
- PromptContent → Generación de contenido AI.
- PromptAds → Gestión de campañas publicitarias.
- PromptCRM → Gestión de leads e interacción con clientes.

Los microservicios se comunican mediante:
- **REST (HTTP)** para llamadas sincrónicas.  
- **Event Hubs (Kafka)** para comunicación asíncrona.  

Los *bounded contexts* definidos en el DDD del entregable anterior se respetan y cada uno mantiene su propio modelo de datos.

---

## 5. Uso de Pub/Sub o Event Driven Design
> *Corresponde al ítem:* “Indicar uso de pub/sub o event driven design.”

Se usa **Azure Event Hubs** (compatible con Kafka) para manejar los eventos clave del sistema.

Eventos principales:
- `content.generated`
- `campaign.executed`
- `notification.sent`

Esto permite que los microservicios se comuniquen de forma asíncrona y desacoplada.  
Cada evento tiene un esquema (ejemplo en `architecture/events_content_generated.avro`).

---

## 6. Decisión: Monolithic o Microservicios
> *Corresponde al ítem:* “Definir si se usará monolithic.”

Se eligió una **arquitectura basada en microservicios**, no monolítica, para aprovechar el aislamiento entre dominios que permite el DDD y tener una escalabilidad mas independiente.

---

## 7. Uso de API Gateway
> *Corresponde al ítem:* “Definir si se va usar algún API gateway.”

Se usara [**Azure Application Gateway**](https://learn.microsoft.com/en-us/azure/application-gateway/) como punto de entrada principal.  
Se encargará de:
- Autenticación y autorización (JWT, OAuth2)
- Filtrado de tráfico y WAF
- Enrutamiento a cada microservicio

---


## 8. Layers de la arquitectura [TODO]
> *Corresponde al ítem:* “Incluir todos los layers de la arquitectura.”

| Capa | Propósito | Carpeta en el código |
|------|------------|---------------------|
| **Presentation** | Manejo de peticiones HTTP / validación | `/src/presentation/` |
| **Application** | Casos de uso / coordinación de servicios | `/src/application/` |
| **Domain** | Entidades, reglas de negocio | `/src/domain/` |
| **Infrastructure** | Conexión con DB, APIs externas | `/src/infrastructure/` |
| **Common / Cross-cutting** | Seguridad, logging, configuración | `/src/common/` |

El diagrama visual también muestra esta separación por capas.

Ver: `architecture/diagrams/high-level-architecture.svg`

---

## 9. Cloud Provider Seleccionado
> *Corresponde al ítem:* “Seleccionar cloud provider entre gRPC, AWS o Azure.”

Se escogió **Azure**, por su buena integración con contenedores, bases de datos administradas y servicios gestionados.  

Componentes principales de Azure usados:
- AKS (Kubernetes)
- Event Hubs (Kafka compatible)
- Azure DB (para PostgreSQL)
- Azure Cache (para Redis)
- Azure Key Vault
- Application Gateway
- Azure Monitor / Log Analytics

---

## 10. Uso de Supabase o Vercel
> *Corresponde al ítem:* “Indicar el uso de Supabase y Vercel si fuese requerido y el cómo se va usar e integrar dentro de su arquitectura con el proveedor de cloud.”

- **Vercel** se usa solo como *frontend hosting* para la aplicación web (Next.js).  
  Se conecta con el API Gateway mediante HTTPS seguro.  
- **Supabase** no se usa directamente, ya que el backend usa PostgreSQL administrado en Azure.

---

## 11. Detalle de patrones, layers, configuración y componentes del cloud [TODO]
> *Corresponde al ítem:* “Para el cloud provider seleccionado, detallar patrones, layers, configuración de componentes…”

| Servicio Azure | Rol | Qué evita programar manualmente |
|----------------|-----|----------------------------------|
| AKS | Orquestación de microservicios | scaling, balanceo, HA |
| Event Hubs | Pub/Sub asíncrono | colas personalizadas |
| DB for PostgreSQL | Persistencia | mantenimiento manual y backups |
| Redis Cache | Cache distribuido | sesiones en memoria |
| Key Vault | Secretos | archivos .env inseguros |
| Application Gateway | API pública y routing | nginx/WAF propio |
| Monitor | Logs y métricas | configuración manual de Prometheus |

Los archivos de configuración estarán en `architecture/k8s/` y `architecture/configs/`.

---

## 12. Guía de programación y seguridad del backend
> *Corresponde al ítem:* “Dar guía de programación de todos los layers y validación de seguridad de métodos del backend.”

Cada layer tendrá plantillas base con ejemplos de uso y comentarios dentro del código.  

Ejemplos en `architecture/templates/`:
- `repository.template.ts`: cómo crear repositorios
- `jwt_middleware.ts`: validación de tokens
- `http_client.ts`: cliente HTTP con retry y circuit breaker

Validaciones de seguridad aplicadas:
- Tokens JWT y OAuth2
- Scopes por ruta
- Sanitización de datos
- Validación de DTOs

---

## 13. Diagramas de clases de puntos críticos [TODO]
> *Corresponde al ítem:* “Incluir diagramas de clases de los puntos críticos…”

Diagramas incluidos:
- `architecture/class-diagrams/content-domain-classes.svg`  
  → Muestra inyección de dependencias y patrón Pub/Sub.  
- `architecture/class-diagrams/campaign-domain-classes.svg`  
  → Muestra patrón Ambassador y orquestación de campañas.

---

## 14. Relación entre diagramas y diseño por dominios
> *Corresponde al ítem:* “El diagrama de arquitectura a nivel de layers y estructura debe hacer match con el diseño por dominios.”

El diseño de dominios del entregable anterior coincide con los microservicios representados aquí:  
cada bounded context tiene su propio servicio y base de datos.  
Las relaciones entre dominios se hacen por ACLs o eventos.

---

## 15. Tecnologías y su integración
> *Corresponde al ítem:* “Numerar todas las tecnologías a usar y explicar integración en cada sistema.”

| # | Tecnología | Uso | Integración |
|--:|-------------|------|-------------|
| 1 | Node.js | Backend | En microservicios principales |
| 2 | Python | Workers/eventos | En consumidores de Event Hub |
| 3 | PostgreSQL | Base de datos | Azure DB for PostgreSQL |
| 4 | Redis | Cache | Azure Cache for Redis |
| 5 | Kubernetes | Orquestador | Azure AKS |
| 6 | Terraform | Infraestructura como código | Definiciones en `/architecture/terraform/` |
| 7 | OpenAPI | Contratos de API | `/architecture/openapi/` |
| 8 | Auth0 / Azure AD | Autenticación | Configuración en Gateway |

---

## 16. Configuraciones especiales
> *Corresponde al ítem:* “Indicar configuraciones especiales si aplica.”

- **Multi-version API routing** (`/api/v1`, `/api/v2`).  
- **CI/CD automático** con GitHub Actions.  

---

## 17. Versiones y control de versiones
> *Corresponde al ítem:* “Especificar versiones a usar y cómo se controlarán las versiones.”

- Las APIs estarán versionadas por ruta (`/api/v1/...`).  
- Dependencias controladas con Dependabot.  
- GitFlow: ramas `main`, `develop`, `feature/`, `release/`, `hotfix/`.  

Cada API puede tener versiones activas simultáneamente (v1, v2) para evitar bloqueos entre equipos. (Ver readme seccion de Maintainability)

---

## 18. Vinculación entre patrones, layers y código real
> *Corresponde al ítem:* “Patrones arquitectónicos, layers de servicios o componentes de cloud y patrones de diseño orientados a objetos deben estar vinculados a código.”

Ejemplos de vínculos:
- Los patrones DI y Repository tienen plantillas en `architecture/templates/`  
- Event Hubs y Pub/Sub se demuestran en `architecture/events_content_generated.avro`  
- Cada layer está reflejado en la estructura `/src/...` de cada microservicio.

---

## Checklist final por revisar
- [] Diagrama de arquitectura general con layers  
- [] Patrones documentados y ubicados en código  
- [] Comunicación y microservicios definidos  
- [] Pub/Sub con eventos críticos incluidos  
- [] Cloud provider seleccionado y justificado  
- [] Guía de programación y seguridad  
- [] Diagramas de clases críticos  
- [] Tecnologías numeradas y versionadas  
- [] Vinculación con el código  

