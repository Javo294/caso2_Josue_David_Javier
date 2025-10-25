
# Caso 2

Integrantes: 
- Josué Salazar
- Javier Rodríguezz
- Jose David Chavez

Curso: Diseño de Software

---

# Métricas de los requerimientos no funcionales

### Performance

#### **Redis Cache - Consultas Cacheadas**
- **Tiempo de respuesta target**: ≤ 300 milisegundos
- **Tiempo de respuesta máximo**: 400 milisegundos (requerimiento original)
- **Operaciones simples (GET)**: ≤ 100 milisegundos
- **Latencia sub-millisecond**: Para operaciones en memoria caliente
- **Tecnología**: Redis Cloud con clustering, configuración de persistent connections
- **Justificación**: Benchmarks de Redis demuestran latencias sub-milisegundo para operaciones en memoria. El objetivo de 300ms incluye overhead de red y serialización[Link](https://redis.io/blog/redisjson-public-preview-performance-benchmarking/)

#### **Cálculo de Throughput**
- **Peak hours (7am-7pm)**: 100,000 operaciones/minuto = **1,666.67 ops/segundo**
- **Off-peak**: 300 procesos background/minuto = **5 ops/segundo**
- **Transacciones diarias estimadas**: 72,000,000 durante 12 horas pico

### Scalability
**Justificación metodológica**: Se aplica la fórmula de Kubernetes HPA para calcular el autoescalado basado en métricas de CPU y memoria, garantizando que el sistema soporte incrementos de 10x en la carga.[](https://cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler)

**Autoescalado Horizontal con Kubernetes HPA**

```
# Configuración base HPA 
minReplicas: 3 
maxReplicas: 30 
targetCPUUtilizationPercentage: 70 
targetMemoryUtilizationPercentage: 80
```

**Fórmula de escalado**: `desiredReplicas = ceil[currentReplicas × (currentMetric / targetMetric)]`[Link](https://www.devzero.io/blog/kubernetes-hpa)

​**Ejemplo de cálculo**: Con 3 replicas iniciales y CPU al 85% durante pico de carga:
- Replicas deseadas = ceil[3 × (85/70)] = ceil[3.64] = **4 replicas**

**Capacidades del sistema**:
- **Carga base**: 500 campañas activas, 30 usuarios concurrentes
- **Carga máxima**: 5,000 campañas activas, 300 usuarios concurrentes (incremento 10x)
- **Throughput pico**: 1,666.67 operaciones/segundo
- **Procesos background**: 5 operaciones/segundo fuera de horario

**Configuración K8s por microservicio**:

```
resources:
	requests:
	    cpu: "500m"
	    memory: "512Mi"
	limits:
	    cpu: "2000m"
	    memory: "2Gi"
        
	behavior:   
		scaleDown:    
			policies:    
				- type: Pods      
				  value: 2      
				  periodSeconds: 60    
				- type: Percent      
				  value: 10      
				  periodSeconds: 60  
		scaleUp:    
			policies:    
				- type: Pods      
				  value: 4      
				  periodSeconds: 30    
				- type: Percent      
				  value: 50      
				  periodSeconds: 30`
```

**Justificación**: La configuración de scale-down conservadora (10% por minuto) previene el "thrashing", mientras que el scale-up agresivo (50% por 30 segundos) responde rápidamente a picos de demanda.[Link](https://cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler)

### Reliability
**Justificación metodológica**: La tasa de errores se establece basándose en estándares de confiabilidad para sistemas transaccionales empresariales.

**Tasa de Errores Máxima Permitida**: 0.1% de transacciones por día
- **Transacciones diarias estimadas**: 72,000,000
- **Errores máximos permitidos**: 72,000 errores/día
- **Errores por hora (peak)**: 6,000 errores/hora

**Monitoreo y Alertas**:
- **Tecnología**: Prometheus + Grafana + AlertManager
- **Métricas clave**:
    - `pg_stat_statements` para PostgreSQL
    - Custom metrics en aplicación usando OpenTelemetry
    - Error tracking con Sentry

**Thresholds de alertas**:
- **Warning**: Error rate > 0.05% (3,600 errores/hora)
- **Critical**: Error rate > 0.1% (6,000 errores/hora)
- **Emergency**: Error rate > 0.5% o servicio completamente caído

**Sistema de notificaciones**:
- PagerDuty para alertas críticas (on-call rotation)
- Slack para warnings
- Email digest para métricas diarias

### Availability
**Justificación metodológica**: El cálculo de disponibilidad se basa en la fórmula estándar: `Availability% = (Uptime / Total Time) × 100`[](https://sderay.com/designing-highly-available-system-achieving-99-999-uptime/)

**Disponibilidad Mínima**: 99.9% mensual

**Downtime permitido**:[Link](https://dev.to/raza_shaikh_eb0dd7d1ca772/kubernetes-high-availability-strategies-for-resilient-production-grade-infrastructure-37fb)
- **Por año**: 525.60 minutos (8.76 horas)
- **Por mes**: 43.20 minutos
- **Por semana**: 10.08 minutos
- **Por día**: 1.44 minutos (86 segundos)

**Configuración de Alta Disponibilidad**:

**1. Load Balancing**
- **Tecnología**: AWS Application Load Balancer (ALB) o Azure Load Balancer
- **Configuración**:
    - Health checks cada 10 segundos
    - Unhealthy threshold: 3 fallos consecutivos
    - Timeout: 5 segundos
    - Cross-zone load balancing habilitado

**2. Failover Automático**
- **Tiempo de detección de falla**: 30 segundos
- **Tiempo de switchover**: 20 segundos
- **Tiempo total de failover**: 50 segundos[Link](https://dev.to/raza_shaikh_eb0dd7d1ca772/kubernetes-high-availability-strategies-for-resilient-production-grade-infrastructure-37fb)    
- **Failovers permitidos por mes**: 52 (sin exceder downtime mensual)

**3. Replicación de Bases de Datos**
- **PostgreSQL**: Replicación síncrona con 2 réplicas (1 sync, 1 async)
- **Redis**: Redis Sentinel con 3 nodos (1 master, 2 replicas)
- **RPO (Recovery Point Objective)**: 0 segundos (replicación síncrona)
- **RTO (Recovery Time Objective)**: 50 segundos

**4. Kubernetes Multi-AZ Deployment**
```
# Node affinity para distribución multi-AZ 
affinity:   
	podAntiAffinity:    
		requiredDuringSchedulingIgnoredDuringExecution:    
			- labelSelector:        
				  matchExpressions:        
					  - key: app          
					    operator: In          
					    values:          
					    - promptsales-api      
			topologyKey: topology.kubernetes.io/zone
```

**Cálculo de disponibilidad en serie** (componentes dependientes):[Link](https://mollysheets.com/2025/03/12/calculating-uptime-for-a-platform-in-k8snaas-and-k8scaas-business-models/)

Si cada componente tiene 99.9% disponibilidad:
- Sistema con 3 componentes en serie: 0.999³ = 0.997 = **99.7%**
- Para mantener 99.9% del sistema completo, cada componente necesita ≥ 99.97% disponibilidad

### Security
**Autenticación y Autorización**
- **OAuth 2.0**: Para autenticación de usuarios y aplicaciones
- **OpenID Connect (OIDC)**: Para single sign-on (SSO)
- **JWT**: Tokens con expiración de 15 minutos, refresh tokens de 7 días
- **Tecnología**: Auth0, Azure AD B2C, o Keycloak

**Cifrado en Tránsito**
- **TLS 1.3**: Todas las comunicaciones entre servicios​
- **mTLS**: Para comunicación entre microservicios en Kubernetes
- **Certificate rotation**: Automática cada 90 días usando cert-manager

**Cifrado en Reposo**
- **AES-256**: Para datos en bases de datos y almacenamiento    
- **Transparent Data Encryption (TDE)**: En PostgreSQL
- **Encrypted EBS volumes**: En AWS

**Auditoría y Logging**
- **Retención**: 90 días mínimo​
- **Stack**: ELK (Elasticsearch, Logstash, Kibana) o Datadog
- **SIEM**: Para detección de amenazas
- **Compliance**: GDPR, CCPA

**Políticas de Acceso**
- **Principle of Least Privilege**: Roles y permisos granulares
- **RBAC en Kubernetes**: Role-Based Access Control
- **Network Policies**: Segmentación de red en K8s

### Maintainability
**Modularidad y Separación de Dominios**
- **Arquitectura**: Domain-Driven Design con bounded contexts claramente definidos[Link](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
- **Microservicios**: Cada bounded context implementado como microservicio independiente
- **Código**: Clean Architecture con separación de layers (domain, application, infrastructure, presentation)

**Documentación**
- **Código**: Comentarios en puntos críticos, JSDoc/TSDoc para funciones públicas
- **API**: OpenAPI 3.0 specification para todos los endpoints REST
- **Arquitectura**: ADRs (Architecture Decision Records) en repositorio
- **README.md**: Completo con setup, deployment, y troubleshooting

**Testing Coverage**
- **Mínimo**: 80% code coverage
- **Unit tests**: Para lógica de dominio
- **Integration tests**: Para APIs y bases de datos
- **E2E tests**: Para flujos críticos de usuario

### Interoperability
**APIs REST**
- **Estándar**: REST Level 3 (HATEOAS cuando sea apropiado) 
- **Formato**: JSON para request/response
- **Versionado**: URL-based (e.g., `/api/v1/campaigns`)
- **Rate limiting**: 1000 requests/minuto por API key

**MCP Servers**
- **Protocolo**: Model Context Protocol para comunicación entre IA y sistemas​
- **Uso**: Integración entre PromptContent, PromptAds y PromptCrm
- **Seguridad**: mTLS para autenticación entre servidores

**Integraciones Externas**
- **Google Ads API**, **Meta Ads API**, **TikTok for Business**
- **HubSpot**, **Salesforce**, **Zendesk**
- **OpenAI API**, **Anthropic API**
- **Patrón**: API Gateway para centralizar y monitorear integraciones

### Compliance
**GDPR (General Data Protection Regulation)**
- Right to access, rectification, erasure (right to be forgotten)
- Data portability
- Consent management
- Data breach notification (72 horas)

**CCPA (California Consumer Privacy Act)**
- Consumer rights to know, delete, opt-out
- Non-discrimination for exercising rights

**Implementación**
- Data retention policies automatizadas
- Anonymization de datos en ambientes no productivos
- Privacy by design en desarrollo
- 
### Extensibility
**Arquitectura Modular**
- **Plugin architecture**: Para agregar nuevos canales de marketing
- **Event-driven**: Pub/Sub con NATS o Kafka para desacoplar componentes
- **Bounded contexts independientes**: Pueden evolucionar sin afectar otros dominios

**Versionado de APIs**
- Múltiples versiones soportadas simultáneamente
- Deprecation policy: 6 meses de notice antes de remover versión

# Domain driven desing

## Identificación de Dominios Principales

### **Dominios Globales (Compartidos)**:
1. **Identity & Access Management (IAM)**
    - Autenticación de usuarios
    - Autorización y permisos
    - Gestión de roles
    - Audit logs de seguridad
    - **Entidades**: User, Role, Permission, Session, AuditLog
        
2. **Billing & Subscriptions**    
    - Gestión de planes y tiers
    - Facturación y pagos 
    - Tracking de uso 
    - **Entidades**: Subscription, Plan, Invoice, Payment, UsageMetric        
3. **Notifications**
    - Email, SMS, push notifications
    - Alertas del sistema
    - **Entidades**: Notification, NotificationTemplate, NotificationLog
        
4. **Analytics & Reporting**
    - Dashboards centralizados
    - Métricas consolidadas de las 3 subempresas
    - Data warehouse
    - **Entidades**: Report, Dashboard, Metric, DataPipeline

### **Bounded Context: PromptContent** [Link](https://semaphore.io/blog/domain-driven-design-microservices)

**Subdominio Core**:
- **Content Generation**: Creación automática de contenido con IA
- **Content Management**: Versionado, aprobaciones, derechos de uso

**Entidades y Agregados**:
- **Content Aggregate Root**: Content
    - ContentId (Value Object)
    - ContentType (enum: text, image, video)
    - Status (enum: draft, review, approved, published)
    - Versions (Collection)
    - ApprovalWorkflow
    - AIGenerationMetadata
        
- **Campaign Content** (Entity)
- **Content Template** (Entity)
- **Media Asset** (Entity con storage reference)
- **AI Prompt** (Value Object)

**Domain Services**:
- `ContentGenerationService`: Integración con OpenAI API, Adobe, Canva
- `ContentApprovalService`: Workflow de aprobación
- `ContentVersioningService`: Control de versiones

**Integraciones Externas**:
- OpenAI API, Anthropic API
- Canva API, Adobe Creative Cloud API
- Meta Business Suite
- Storage (S3, Azure Blob Storage)

### **Bounded Context: PromptAds** [Link](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)

**Subdominio Core**:
- **Campaign Management**: Diseño, segmentación y ejecución de campañas
- **Ad Performance**: Análisis en tiempo real

**Entidades y Agregados**:
- **Campaign Aggregate Root**: Campaign
    - CampaignId (Value Object)
    - TargetAudience (Value Object)
    - Budget (Value Object: amount, currency, spend)
    - Schedule (Value Object: start, end, timezone)
    - Channels (Collection: Google Ads, Meta, TikTok, Email)
    - PerformanceMetrics (Value Object)
        
- **Ad Creative** (Entity)
- **Audience Segment** (Entity)
- **Budget Allocation** (Entity)
- **Performance Report** (Entity)

**Domain Services**:
- `CampaignExecutionService`: Publicación en plataformas
- `AudienceTargetingService`: Segmentación con IA
- `BudgetOptimizationService`: Ajuste automático de presupuesto
- `PerformanceAnalyticsService`: Análisis en tiempo real

**Integraciones Externas**:
- Google Ads API
- Meta Ads API
- TikTok for Business API
- Mailchimp API
- LinkedIn Campaign Manager API

### **Bounded Context: PromptCrm** [Link](https://semaphore.io/blog/domain-driven-design-microservices)

**Subdominio Core**:
- **Lead Management**: Captura, clasificación y seguimiento
- **Customer Engagement**: Chatbots, voicebots, automation

**Entidades y Agregados**:
- **Lead Aggregate Root**: Lead
    - LeadId (Value Object)
    - ContactInfo (Value Object)
    - Source (enum: website, social, referral)
    - Score (Value Object: calculado con IA)
    - Status (enum: new, qualified, nurturing, converted)
    - InteractionHistory (Collection)
    - PurchaseIntent (Value Object)

- **Customer** (Entity)
- **Interaction** (Entity: email, call, chat, meeting)
- **Deal** (Entity)
- **Conversation** (Entity: chatbot/voicebot)

**Domain Services**:
- `LeadScoringService`: Predicción de intención con IA
- `ConversationService`: Chatbot/voicebot automation
- `LeadNurturingService`: Flujos automatizados
- `IntegrationSyncService`: Sincronización con CRMs externos

**Integraciones Externas**:
- HubSpot API
- Salesforce API
- Zendesk API
- WhatsApp Business API
- Twilio API (SMS, Voice)

### **Bounded Context: PromptSales (Portal Unificado)** [Link](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
**Subdominio Core**:
- **Strategy Design**: Diseño de estrategias de mercadeo multicanal
- **Orchestration**: Coordinación entre las 3 subempresas
- **Consolidated Analytics**: Reportería unificada

**Entidades y Agregados**:
- **Marketing Strategy Aggregate Root**: MarketingStrategy
    - StrategyId (Value Object)
    - Client (Value Object)
    - Objectives (Collection)
    - Timeline (Value Object)
    - Budget (Value Object)
    - ContentPlan (referencia a PromptContent)
    - CampaignPlan (referencia a PromptAds)
    - LeadFlows (referencia a PromptCrm)
    - ApprovalStatus
        
- **Client** (Entity)
- **Objective** (Value Object: KPI, target, deadline)
- **Task Schedule** (Entity: agenda inteligente)
- **Consolidated Report** (Entity)

**Domain Services**:
- `StrategyOrchestrationService`: Coordinación entre bounded contexts
- `AIRecommendationService`: Sugerencias automáticas
- `ConsolidatedAnalyticsService`: Agregación de métricas
- `ApprovalWorkflowService`: Revisión y aprobación

---
## Contratos entre Dominios mediante Interfaces

### **Patrón: Domain Model Facade as API** [Link](https://eprints.cs.univie.ac.at/6948)

​**1. Facade: ContentServiceFacade**

typescript
```
// Interface pública para PromptSales → PromptContent
interface ContentServiceFacade {   
	generateContent(request: ContentGenerationRequest): Promise<ContentResponse>;
	getContentById(contentId: string): Promise<ContentDTO>;
	approveContent(contentId: string, approver: string): Promise<void>;
	listContentByCampaign(campaignId: string): Promise<ContentDTO[]>;
} 

// Value Objects (Anti-Corruption Layer) 
interface ContentGenerationRequest {   
	campaignId: string;   
	contentType: 'text' | 'image' | 'video';   
	audience: AudienceProfile;   prompt: string;   
	constraints?: ContentConstraints; 
} 

interface ContentDTO {
	id: string;   
	type: string;   
	status: string;   
	url?: string;   
	metadata: Record<string, any>;   
	createdAt: Date; 
}
```

**2. Facade: CampaignServiceFacade**

typescript
```
// Interface pública para PromptSales → PromptAds 

interface CampaignServiceFacade {   
	createCampaign(request: CampaignCreationRequest): Promise<CampaignDTO>;
	executeCampaign(campaignId: string): Promise<ExecutionResult>;   
	getPerformanceMetrics(campaignId: string): Promise<PerformanceDTO>;   
	pauseCampaign(campaignId: string): Promise<void>;
} 

interface CampaignCreationRequest {   
	strategyId: string;   
	budget: BudgetAllocation;   
	audience: AudienceSegment;   
	schedule: ScheduleConfig;   
	channels: Channel[]; 
} 

interface PerformanceDTO {   
	campaignId: string;   
	impressions: number;   
	clicks: number;   
	conversions: number;   
	spend: number;   
	roi: number;   
	updatedAt: Date; 
}
```

**3. Facade: LeadServiceFacade**

typescript
```
// Interface pública para PromptSales → PromptCrm 

interface LeadServiceFacade {   
	createLead(source: string, data: LeadData): Promise<LeadDTO>;   
	scoreLeads(leads: string[]): Promise<ScoredLeadDTO[]>;   
	getLeadsByStrategy(strategyId: string): Promise<LeadDTO[]>;   
	startAutomatedNurturing(leadId: string, flow: string): Promise<void>; 
} 

interface LeadDTO {   
	id: string;   
	contactInfo: ContactInfo;   
	score: number;   
	status: string;
	source: string;   
	lastInteraction?: Date; 
} 

interface ScoredLeadDTO extends LeadDTO {   
	purchaseIntent: number; // 0-100   
	nextBestAction: string;   
	predictedConversionDate?: Date;
}
```

**4. API REST Contracts (OpenAPI)** [Link](https://www.microservice-api-patterns.org/ZIO-FromDDDToMAPIsQS2020v10p.pdf)

​Todos los facades se exponen también como APIs REST siguiendo el patrón **Aggregate Roots as API Endpoints**:[Link](https://eprints.cs.univie.ac.at/6948)

text
```
# OpenAPI 3.0 Contract Example 
/api/v1/content:   
	post:    
		summary: Generate content    
		operationId: generateContent    
		security:      
			- OAuth2: [content:write]    
			requestBody:      
				$ref: '#/components/schemas/ContentGenerationRequest'   
			responses:      
				201:        
					$ref: '#/components/schemas/ContentResponse'      
				400:        
					$ref: '#/components/schemas/ValidationError'
```

## Diagrama de Dominios

<img width="811" height="791" alt="image" src="https://github.com/user-attachments/assets/fbf91f9e-3677-41ce-b5b9-0521937dfd4e" />

El sistema se estructura en capas con los siguientes bounded contexts:[link](https://martinfowler.com/bliki/BoundedContext.html)

**Relaciones entre Bounded Contexts**:[Link](https://www.infoq.com/articles/ddd-contextmapping/)

1. **PromptSales → PromptContent**: Customer/Supplier (Open Host Service)
2. **PromptSales → PromptAds**: Customer/Supplier (Open Host Service)
3. **PromptSales → PromptCrm**: Customer/Supplier (Open Host Service)
4. **Todos → Shared Kernel**: Shared (IAM, Billing, Notifications)
5. **Todos → External Systems**: Anti-Corruption Layer (ACL)
## Independencia entre Dominios

**Principios de Independencia**:[](https://blog.bitsrc.io/developing-a-ddd-oriented-microservices-1b65bd45e2a8)

1. **Desacoplamiento de Datos**: Cada bounded context tiene su propia base de datos
2. **Comunicación Asíncrona**: Event-driven con NATS/Kafka para operaciones no críticas
3. **Comunicación Síncrona**: REST APIs con circuit breakers (Resilience4j) para operaciones críticas
4. **Versionado**: Cada API mantiene versiones independientes
5. **Deployment**: Cada bounded context se despliega independientemente en Kubernetes

**Pruebas por Dominio**:[Link](https://blog.bitsrc.io/developing-a-ddd-oriented-microservices-1b65bd45e2a8)

1. **Unit Tests**: Lógica de dominio pura (sin dependencias externas)
2. **Integration Tests**: Facades y repositories con bases de datos de test
3. **Contract Tests**: Validación de contratos entre bounded contexts (Pact)
4. **E2E Tests**: Flujos completos que cruzan múltiples dominios

