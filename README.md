# Caso2_Josue_David_Javier

## 
```bash
root/
├── apps/
│   ├── web/                        # Next.js Frontend
│   │   └── src/
│   │       ├── app/                # Pages / routes
│   │       ├── components/         # UI components
│   │       ├── campaigns/          # Campaigns dashboard
│   │       ├── analytics/          # Data visualization
│   │       └── lib/                # API clients, utils
│   └── admin-dashboard/            # Management interface
│
├── packages/
│   ├──domain/
│   │   ├── shared/             
│   │   │   ├── value_objects/
│   │   │   │   ├── email.py
│   │   │   │   └── money.py
│   │   │   └── domain_events/
│   │   │       ├── event_bus.py
│   │   │       └── domain_event.py
│   │   ├── global/
│   │   │   ├── entities/
│   │   │   │   ├── user.py
│   │   │   │   ├── organization.py
│   │   │   │   └── tenant.py
│   │   │   ├── aggregates/
│   │   │   │   └── user_organization.py
│   │   │   ├── services/
│   │   │   │   └── authentication_service.py
│   │   │   └── repositories/
│   │   │       ├── user_repository.py
│   │   │       └── organization_repository.py
│   │   │
│   │   ├── promptcontent/
│   │   │   ├── entities/
│   │   │   │   ├── content.py
│   │   │   │   └── approval_request.py
│   │   │   ├── aggregates/
│   │   │   │   └── content_workflow.py
│   │   │   ├── domain_services/
│   │   │   │   └── content_generator.py
│   │   │   ├── value_objects/
│   │   │   │   ├── content_status.py
│   │   │   │   └── content_spec.py
│   │   │   └── events/
│   │   │       ├── content_approved.py
│   │   │       └── content_rejected.py
│   │   │
│   │   ├── promptads/ 
│   │   │   ├── entities/
│   │   │   │   ├── ad_campaign.py
│   │   │   │   └── ad_creative.py
│   │   │   ├── aggregates/
│   │   │   │   └── campaign_management.py
│   │   │   ├── domain_services/
│   │   │   │   └── budget_allocator.py
│   │   │   └── value_objects/
│   │   │       ├── budget.py
│   │   │       └── targeting_criteria.py
│   │   │
│   │   └── promptcrm/
│   │       ├── entities/
│   │       │   ├── lead.py
│   │       │   └── conversation.py
│   │       ├── aggregates/
│   │       │   └── lead_conversation.py
│   │       ├── domain_services/
│   │       │   └── lead_scorer.py
│   │       └── value_objects/
│   │           ├── lead_score.py
│   │           └── contact_info.py
│   │
│   ├── infrastructure/             # Infra layer (implementations)
│   │   ├── database/               # SQLAlchemy repositories
│   │   ├── external_api/           # API clients (Google, Meta, OpenAI)
│   │   ├── cache/                  # Redis cache adapters
│   │   ├── logging/                # Observability tools
│   │   └── auth/                   # JWT / OAuth2 provider
│   │
│   ├── application/                # Flask application layer
│   │   ├── api/                    # Blueprints per bounded context
│   │   ├── schemas/                # Request/response models (Pydantic)
│   │   ├── facades/                # Coordination between services
│   │   └── config/                 # Environment and DI container
│   │
│   └── UI/                         # Shared  UI utilities 
├── infra/                          # Infrastructure deployment
│   ├── docker/                     # Dockerfiles
│   ├── k8s/                        # Kubernetes manifests
│   ├── ci-cd/                      # GitHub Actions workflows
│   └── terraform/                  # IaC for Azure
│
└── docs/
    ├── domain-diagrams/            # Domain diagrams + dependencies
    └── api-specs/                  # OpenAPI files
```

## Diargama DDD

```mermaid

```

## Stack de tecnologias
| **Área/Capa** | **Tecnología** | **Rol**|
| --- | --- | --- |
|**Backend (App layer)** | Python + Flask | API REST, inyección de dependencias, routing |
|**Domain Layer**| Python puro | Entidades, value objects, reglas de negocio |
|**Infra Layer** | SQLAlchemy, Redis, Requests, JWT | Repositorios, integraciones, cache, seguridad|
|**Frontend**| Next.js, TailwindCSS, ShadCN |Interfaz, dashboards, componentes UI|
| **Base de datos**| PostgreSQL| Datos relacionales|
| **Cache / Queue**|Redis| Cache y procesamiento asíncrono|
| **Infraestructura**|Docker, Kubernetes| Contenerización y despliegue|
| **CI/CD**| GitHub Actions| Automatización de pruebas y despliegues|
| **Observabilidad**|Prometheus, Grafana| Métricas, trazabilidad y monitoreo|
| **Seguridad**|OAuth2, GDPR| Autenticación, autorización y cifrado de datos |

## Métricas de los requerimientos no funcionales

### Maintainability
**Objetivo:** asegurar que el código sea modular, entendible, testeable y fácil de evolucionar, especialmente bajo un diseño DDD con separación de dominios.

|Métrica|Valor objetivo|Parámetro|Herramientas|Ejemplo/justificación|
|---------|---------|---------|---------|---------|
|Maintainability Index| ≥ 80 (sobre 0-100) | MI = MAX(0, (171 − 5.2·ln(Halstead Volume) − 0.23·CyclomaticComplexity − 16.2·ln(LOC)) ·100/171)| SonarQube & radon (Python) | Si un módulo de un bounded context obtiene MI = 85 → buen nivel de mantenibilidad |
|Complejidad ciclomática promedio por función | < 10 | (Sum de CyclomaticComplexity funciones) / número de funciones | SonarQube & radon (Python) | Si módulo PromptAds tiene promedio 8 → cumple |
|Porcentaje de funciones con complejidad > 50 | 0 % | (# funciones con CC>50) / (total funciones)|SonarQube| Si 0 de 500 funciones > 50 → cumple|
|Cobertura de pruebas unitarias en dominio| ≥ 75 % líneas cubiertas | líneas ejecutadas en tests / líneas totales del domain|pytest| Si un dominio tiene 80% cobertura → cumple |

### Interoperability
**Objetivo:** asegurar que los servicios REST/APIs entre subempresas y externos se integren fácilmente, sean documentados, versionados y operados sin fricción.

|Métrica|Valor objetivo|Parámetro|Herramientas|Ejemplo/justificación|
|---------|---------|---------|---------|---------|
| Cobertura de documentación API (OpenAPI) | ≥ 100 % endpoints públicos | (# endpoints con spec OpenAPI) / (total endpoints públicos) | Spectral, linters OpenAPI | Si 50/50 tienen definición → cumple |
| Latencia de integración | < 500 ms directas; < 400 ms cacheadas | medir trazas de llamadas entre servicios | OpenTelemetry, Prometheus | Si gateway → PromptAds tarda 450 ms → dentro del límite |

### Compliance
**Objetivo:** medir el grado de cumplimiento de regulaciones de protección de datos (como General Data Protection Regulation / GDPR), gestión de consentimiento, auditoría, acceso a datos personales, retención, etc.

|Métrica|Valor objetivo|Parámetro|Herramientas|Ejemplo/justificación|
|---------|---------|---------|---------|---------|
| % de registros con consentimiento válido | ≥ 99.5 % | (# registros con consentimiento válido) / (total registros que requieren consentimiento) | consultas DB + dashboard | Si 99.7% leads con consentimiento → cumple |
| MTTD (Mean Time to Detect incidentes de datos) | < 4 horas | promedio detección de incidente desde ocurrencia | SIEM + alertas | Si detectan acceso no autorizado en 3 h → cumple |
| MTTR (Mean Time to Remediate incidentes) | ≤ 72 horas | promedio tiempo para contener + mitigar + notificar | registros de incidentes | Si promedio 48 h → cumple |
| Audit trail completeness para PII | 100 % eventos críticos (create/update/delete) retenidos ≥ 90 días | (# eventos auditados) / (total eventos críticos) | logging centralizado | Si todo cambio PII queda registrado → cumple |
| Retención de datos conforme política | 100 % de datos sensibles archivados o eliminados según política (ej. 90 días) | verificar job automático + datos reales | jobs de purga + sistema | Si datos >90 días son purgados → cumple |
| Cifrado en reposo / tránsito | 100 % datos sensibles cifrados (AES-256 en reposo, TLS 1.3 en tránsito) | revisión de configuración DB/infra y certificados | auditoría de cifrado | Si DB usa AES-256 y APIs TLS 1.3 → cumple |

### Extensibility
**Objetivo:** permitir agregar nuevas subempresas o módulos al sistema sin necesidad de modificar los sistemas existentes o romperlos.

|Métrica|Valor objetivo|Parámetro|Herramientas|Ejemplo/justificación|
|---------|---------|---------|---------|---------|
| Tiempo para añadir nuevo módulo/servicio | ≤ 5 días | tiempo desde requerimiento hasta integración + test + despliegue | histórico de commits | Si agregas “PromptVideoAds” en 4 días → cumple |
| Número de cambios en módulos existentes al añadir nuevo | ≤ 1 cambio por módulo existente | contar modificaciones en módulos ajenos al nuevo | análisis de Git diff | Si sólo 1 módulo compartido cambió → cumple |
| Puntos de extensión (hooks/plugins) por módulo clave | ≥ 3 hooks/plugins documentados | contar interfaces/hook disponibles | revisión de documentación | módulo Ads con 3 hooks documentados → cumple |
| Porcentaje de módulos desplegables de forma independiente | ≥ 90 % | (# módulos desplegables sin afectar otros) / (total módulos) | logs CI/CD | Si 9 de 10 módulos independientes → 90% → cumple |

--- 

### Performance [TODO]
Tiempo máximo permitido para una consulta estándar: 1.5 segundos.
Tiempo máximo para resultados cacheados: 200 ms usando Redis.
Tecnología: PostgreSQL, Redis; aquí pueden ser métricas por las técnologías clave por separado.

### Scalability [TODO]
Debe soportar incremento de carga de hasta 10x sin degradación.
Kubernetes configurado con autoescalado horizontal por CPU y memoria. Justificarlo con la configuracion de K8s.

### Reliability [TODO]
Tasa de errores máxima permitida: 0.1% de transacciones por día.
Monitoreo con pg_stat_statements y logs centralizados.
Es importante determinar como se monitorea y como se notifican alertas.

### Availability [TODO]
Disponibilidad mínima: 99.9% mensual. En el diseño de infraestructura debe lograr verse como se logra esto, podría ir en el diagrama de arquitectura, pero sería mejor uno de infraestructura.
Redis y bases de datos con failover y replicación.
Considere load balancers.

### Security [TODO]
Autenticación: OAuth 2.0 y/o OpenID Connect.
Cifrado TLS 1.3 en comunicación y AES-256 en reposo.

### Tecnologias para medir RNF

- [Radon](https://pypi.org/project/radon/): Radon is a Python tool that computes various metrics from the source code. Radon can compute cyclomatic complexity, raw metrics (these include SLOC, comment lines, blank lines, &c.), Halstead metrics and Maintainability Index.
- [SonarQube](https://github.com/SonarSource/sonarqube):an open-source, self-hosted platform for continuous inspection of code quality and security, designed to integrate into development environments and CI/CD pipelines.
- [Pytest](https://pypi.org/project/pytest/): framework makes it easy to write small tests, yet scales to support complex functional testing for applications and libraries.
- [OpenTelemetry](https://opentelemetry.io/): collection of APIs, SDKs, and tools. Use it to instrument, generate, collect, and export telemetry data (metrics, logs, and traces) to help analyze software’s performance and behavior.
- [Prometheus](https://prometheus.io/): an open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
- [Spectral](https://stoplight.io/open-source/spectral): an open-source API style guide enforcer and linter.