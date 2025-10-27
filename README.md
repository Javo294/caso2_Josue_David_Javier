
# Caso 2

Integrantes: 
- Josué Salazar
- Javier Rodríguezz
- Jose David Chavez

Curso: Diseño de Software

---

# Métricas de los requerimientos no funcionales

### Performance
- Redis + Postgresql + Python

#### Redis
[Benchmark de Redis](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)
<img width="764" height="233" alt="image" src="https://github.com/user-attachments/assets/118526ba-9f70-429b-b9e4-326a337cc8c2" />

Tiempo máximo para resultados cacheados: 500ms

(Pendiente calculo propio siguiendo la guía del link)

#### Postgresql
[Benchmark de Postgresql](https://www.tigerdata.com/blog/benchmarking-postgresql-batch-ingest#the-results)
<img width="844" height="688" alt="image" src="https://github.com/user-attachments/assets/153af99c-af00-4928-bc94-e734c4ece141" />

(Pendiente calculo propio siguiendo la guia del link)

### Scalability
**Autoescalado Horizontal con Kubernetes HPA**

```
# Configuración base HPA 
minReplicas: 3 
maxReplicas: 30 
targetCPUUtilizationPercentage: 70 
targetMemoryUtilizationPercentage: 80
```

**Fórmula de escalado**: 
Usando escalado automático horizontal (HPA), se establece que si el uso del CPU supera el 70%, el HPA crea más pods hasta un máximo de 30, siendo inferior al 70% disminuye los pods hasta un mínimo de 3

**Capacidades del sistema**:
- **Carga base**: 500 campañas activas, 30 usuarios concurrentes
- **Carga máxima**: 5,000 campañas activas, 300 usuarios concurrentes (debe soportar un incremento x10)
- **Throughput pico**: 1,500 operaciones/segundo
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

### Reliability
**Tasa de Errores Máxima Permitida**: 0.1% de transacciones por día
- **Transacciones diarias estimadas**: 64,800,000 (Sea un promedio de 50% de las transacciones por segundo en hora pico al día, son (1500/2)x86400)
- **Errores máximos permitidos**: 64,800 errores/día
- **Errores por hora (peak)**: 5,000 errores/hora

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
**Disponibilidad Mínima**: 99.9% mensual

**Downtime permitido**:
- **Por año**: 525.60 minutos (8.76 horas)
- **Por mes**: 43.20 minutos
- **Por semana**: 10.08 minutos
- **Por día**: 1.44 minutos (86 segundos)

**Configuración de Alta Disponibilidad**:

**1. Load Balancing**
- **Tecnología**: Azure 
- **Configuración**:
    - Health checks cada 10 segundos
    - Unhealthy threshold: 3 fallos consecutivos
    - Timeout: 5 segundos
    - Cross-zone load balancing habilitado

**2. Failover Automático**
- **Tiempo de detección de falla**: 30 segundos
- **Tiempo de switchover**: 20 segundos
- **Tiempo total de failover**: 50 segundos 
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

**Cálculo de disponibilidad en serie** (componentes dependientes):

Si cada componente tiene 99.9% disponibilidad:
- Sistema con 3 componentes en serie: 0.999³ = 0.997 = **99.7%**
- Para mantener 99.9% del sistema completo, cada componente necesita ≥ 99.97% disponibilidad

### Security
**Autenticación y Autorización**
- Para la autenticación de los usuarios usamos OpenID Connect (OIDC). 
- Para los servicios utilizamos OAuth2 Client Credentials.
- Auth0 se encarga de documentar el flujo de los servicios y de validarlos por JWKS (RS256)
- Tecnologías usadas: Auth0

**Bases de datos**
-PostgreSQL
-MariaDB
-MongoDB

**Cifrado**
- **En transito**: Utilizamos TLS 1.3 para todas las comunicaciones entre servicios​
- **Para las bases de datos y almacenamiento**: Utilizamos AES-256 con la excepción de PostgreSQL     
- **En PostgreSQL**: Transparent Data Encryption (TDE)

**Auditoría y Logging**
- **Retención**: 90 días mínimo​
- **Stack**: ELK (Elasticsearch, Logstash, Kibana)
- **Compliance**: GDPR, CCPA

### Maintainability

#### Durante el desarrollo

**Sistema para la gestión del código**

**GitFlow:** Implementamos GitFlow con ramas principales `main` (producción) y `develop` (integración). Para mas infromacion en [GitFlow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) 

- **Branching strategy:**
	1. `main` → Producto terminado y funcionando 
	2. `develop`→ Donde se arma el próximo producto
	3. `feature/*` → Branches de trabajo para nuevas funcionalidades
	4. `release/*` → Control de calidad antes de enviar a producción/main
	5. `hotfix/*` → Reparaciones urgentes del sistema en uso

**Pull Requests:** Requeridos para todo merge a `develop` o `main`, con revisión de al menos otro desarrollador (NO el que realizo el codigo) para `develop` y de los otros miembros del equipo para el `main`

**Sistema de Tickets**

Seguimiento de issues con templates estandarizados. Etiquetado por tipo (bug, feature, improvement) y prioridad. **Herramientas:** Trello y GitHub Issues

#### Soporte post desarrollo

Niveles de Soporte
- **L1:** Manuales de usuario y videos tutoriales para problemas comunes
- **L2:** Soporte por email con tiempo máximo de respuesta de 48 horas
- **L3:** Sistema de ticketing para issues técnicos con escalamiento al equipo de desarrollo

#### Arquitectura de Mantenibilidad

- **Arquitectura**: Domain-Driven Design con bounded contexts claramente definidos
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

### Interoperability

#### APIs REST

Se va a implementar APIs REST con JSON como formato estándar. Versionado mediante el fromato (`/api/v<Numero de version>/`)

Documentación OpenAPI 3.0 para todos los endpoints

#### MCP Servers

Utilizamos Model Context Protocol para comunicación entre sistemas IA. Facilita integración entre módulos (PromptContent, PromptAds, PromptCrm)

#### Integraciones Externas

- APIs de plataformas de ads (Google Ads, Meta Ads, TikTok for Business)
- CRMs (Customer relationship management)(HubSpot, Salesforce, Zendesk)
- APIs de IA (OpenAI, Anthropic)
- Centralizadas mediante API Gateway para monitoreo y gestión

### Compliance

#### Procesamiento de Pagos y Transacciones

Todas las transacciones financieras se realizan mediante servicios terceros. Ejemplos de servicios: Stripe, PayPal, Plaid para procesamiento de pagos.

**Nota:** No se busca almacenar datos sensibles de tarjetas de crédito o información bancaria

#### Estándares de Seguridad

**OWASP Web Security:** Cumplimiento completo de estándares OWASP para aplicaciones web. Siguiendo el testing oficial de [Web Application Security Testing](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/)

**OWASP Backend Security:** Implementación de mejores prácticas OWASP para backend

**Política de Vulnerabilidades:**
- 0 vulnerabilidades críticas permitidas
- Máximo 40 warnings en escaneos de seguridad
(Entre ambos, web y backend)

#### Protección de Datos

Cumplimiento GDPR (Protección de Datos Europeo): derechos de acceso, rectificación, eliminación y portabilidad
Cumplimiento CCPA (Protección de Datos California): derechos de conocimiento, eliminación y opt-out (que NO se puedan vender los datos)
Políticas automatizadas de retención de datos
Anonimización de datos

### Extensibility

#### Sistemas de Extensión

- **REST API:** Permite integración con sistemas externos y desarrollo de extensiones
- **MCP Servers:** Protocolo para conectar nuevos servicios y capacidades de IA
- **Agregación de Dominios:** Posibilidad de incorporar nuevos bounded contexts al sistema

#### Arquitectura Extensible

- **Arquitectura Modular:** Microservicios independientes que pueden evolucionar separadamente
- **Versionado de APIs:** Soporte simultáneo de múltiples versiones

# Domain driven desing

## Identificación de Dominios Principales

### PromptContent Domains

**Content Domain:** (`ContentManagementService`) Gestión central de contenido y sus propiedades
- `ContentContract` - createContent(), updateContent(), getContent(), deleteContent()

**AI Domain:** (`AIContentGenerationService`) Integración y gestión de servicios de IA
- `AIContentContract` - generateText(), generateImage(), optimizeContent()

**Template Domain:** (`TemplateManagementService`) Plantillas y estructuras reutilizables
- `TemplateContract` - createTemplate(), applyTemplate(), cloneTemplate()

**Media Domain:** (`MediaAssetService`) Gestión de assets multimedia
- `MediaContract` - uploadMedia(), resizeMedia(), getMediaMetadata()

**Approval Domain:** (`ContentApprovalService`) Flujos de revisión y aprobación
- `ApprovalContract` - submitForApproval(), approveContent(), rejectContent()

**Versioning Domain:** (`ContentVersioningService`) Control de versiones e historial
- `VersioningContract` - createVersion(), rollbackVersion(), getVersionHistory()

**Rights Domain:** (`RightsManagementService`) Gestión de derechos de uso y licencias
- `RightsContract` - validateRights(), assignLicense(), checkPermissions()

**Channel Domain:** (`ChannelAdaptationService`) Adaptación de contenido por canal
- `ChannelContract` - adaptForChannel(), validateChannelFormat()

### PromptAds Domains

**Campaign Domain:** (`CampaignManagementService`) Gestión central de campañas
- `CampaignContract` - createCampaign(), pauseCampaign(), updateCampaign()

**Domain Services**:
- `ContentGenerationService`: Integración con OpenAI API, Adobe, Canva
- `ContentApprovalService`: Workflow de aprobación
- `ContentVersioningService`: Control de versiones

**Integraciones Externas**:
- OpenAI API, Anthropic API
- Canva API, Adobe Creative Cloud API
- Meta Business Suite
- Storage (Azure)

### **Bounded Context: PromptAds** 

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
**Audience Domain:** (`AudienceTargetingService`) Segmentación y gestión de públicos
- `AudienceContract` - segmentAudience(), updateSegments(), analyzeAudience()

**AI Domain:** (`AIOptimizationAdsService`) Integración y gestión de servicios de IA
- `AIOptimizationAdsContract` - predictPerformance(), optimizeBidding(), generateInsights()

**Budget Domain:** (BudgetService) Gestión y optimización de presupuestos
- `BudgetContract` - allocateBudget(), adjustBudget(), trackSpend()

**Creative Domain:** (`CreativeManagementService`) Gestión de creatividades publicitarias
- `CreativeContract` - createAdCreative(), testCreative(), optimizeCreative()

**Performance Domain:** (`PerformanceAnalyticsService`) Análisis y métricas de rendimiento
- `PerformanceContract` - trackMetrics(), generateReport(), calculateROI()

**Channel Domain:** (`MultiChannelService`) Gestión multi-canal
- `ChannelManagementContract` - deployToChannels(), syncChannels(), channelAnalytics()

**Optimization Domain:** (`CampaignOptimizationService`) Optimización automática de campañas
- `OptimizationContract` - autoOptimize(), aBTesting(), applyOptimizations()

**Payment Domain:** (`AdPaymentService`) Gestión de pagos y facturación
- `PaymentContract` - processPayment(), generateInvoice(), handleRefunds()

**Reaction Domain:** (`EngagementTrackingService`) Gestión de reacciones y engagement
- `ReactionContract` - trackEngagement(), analyzeSentiment(), responseManagement()

### PromptCrm Domains

**AI Domain:** (`AICustomerService`) Integración y gestión de servicios de IA
- `AICRMContract` - predictBehavior(), automateResponses(), sentimentAnalysis()

**Lead Domain:** (`LeadManagementService`) Gestión central de leads
- `LeadContract` - captureLead(), qualifyLead(), nurtureLead()

**Customer Domain:** (`CustomerManagementService`) Gestión de clientes
- `CustomerContract` - createCustomer(), updateProfile(), customerHistory()

**Interaction Domain:** (`InteractionTrackingService`) Historial de interacciones
- `InteractionContract` - logInteraction(), getTimeline(), interactionAnalytics()

**Scoring Domain:** (`LeadScoringService`) Scoring y clasificación
- `ScoringContract` - calculateScore(), updateScoringModel(), prioritizeLeads()

**Automation Domain:** (`WorkflowAutomationService`) Flujos automatizados
- `AutomationContract` - triggerWorkflow(), configureAutomation(), pauseWorkflow()

**Deal Domain:** (`DealManagementService`) Gestión de oportunidades
- `DealContract` - createDeal(), updatePipeline(), forecastRevenue()

**Conversation Domain:** (`ConversationService`) Chatbots y voicebots
- `ConversationContract` - handleMessage(), escalateToHuman(), botTraining()

**Integration Domain:** (`CRMIntegrationService`) Sincronización con CRMs externos
- `IntegrationContract` - syncData(), mapFields(), handleWebhooks()

**Marketing Domain:** (`MarketingService`) Gestión de marketing
- `MarketingContract` - createCampaign(), trackConversions(), leadScoring()

**Payment Domain:** (`CRMPaymentService`) Gestión de pagos CRM
- `CRMPaymentContract` - processSubscription(), handleBilling(), paymentHistory()


### PromptSales (Portal Unificado) Domains

**Strategy Domain:** (`StrategyDesignService`) Diseño y gestión de estrategias
- `StrategyContract` - createStrategy(), validateStrategy(), updateStrategy()

**Orchestration Domain:** (`CrossContextOrchestrationService`) Coordinación entre bounded contexts
- `OrchestrationContract` - coordinateWorkflows(), syncData(), handleEvents()

**Client Domain:** (`ClientManagementService`) Gestión de clientes y sus datos
- `ClientContract` - onboardClient(), updateClientData(), clientReporting()

**Analytics Domain:** (`ConsolidatedAnalyticsService`) Analítica consolidada
- `AnalyticsContract` - aggregateMetrics(), generateInsights(), predictiveAnalytics()

**Workflow Domain:** (`WorkflowManagementService`) Gestión de flujos de trabajo
- `WorkflowContract` - createWorkflow(), assignTasks(), trackProgress()

**Recommendation Domain:** (`AIRecommendationService`) Recomendaciones de IA
- `RecommendationContract` - generateRecommendations(), optimizeSuggestions(), learnFromFeedback()

**Scheduling Domain:** (`IntelligentSchedulingService`) Agenda y calendario inteligente
- `SchedulingContract` - scheduleCampaign(), optimizeTimeline(), handleConflicts()

**Approval Domain:** (`UnifiedApprovalService`) Flujos de aprobación unificados
- `ApprovalContract` - submitForApproval(), approvalWorkflow(), trackApprovals()

**Services Domain:** (`ServiceCatalogService`) Gestión de servicios y productos
- `ServiceContract` - manageServices(), bundleProducts(), serviceAnalytics()

**User Domain:** (`UserManagementService`) Gestión de usuarios y permisos
- `UserContract` - createUser(), assignRoles(), managePermissions()

## Contratos entre Dominios mediante Interfaces

### **Patrón: Domain Model Facade as API** 

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

**4. API REST Contracts (OpenAPI)** 

​Todos los facades se exponen también como APIs REST siguiendo el patrón **Aggregate Roots as API Endpoints**:

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

El sistema se estructura en capas con los siguientes bounded contexts:

**Relaciones entre Bounded Contexts**:

1. **PromptSales → PromptContent**: Customer/Supplier (Open Host Service)
2. **PromptSales → PromptAds**: Customer/Supplier (Open Host Service)
3. **PromptSales → PromptCrm**: Customer/Supplier (Open Host Service)
4. **Todos → Shared Kernel**: Shared (IAM, Billing, Notifications)
5. **Todos → External Systems**: Anti-Corruption Layer (ACL)
## Independencia entre Dominios

**Principios de Independencia**:

1. **Desacoplamiento de Datos**: Cada bounded context tiene su propia base de datos
2. **Comunicación Asíncrona**: Event-driven con NATS/Kafka para operaciones no críticas
3. **Comunicación Síncrona**: REST APIs con circuit breakers (Resilience4j) para operaciones críticas
4. **Versionado**: Cada API mantiene versiones independientes
5. **Deployment**: Cada bounded context se despliega independientemente en Kubernetes

**Pruebas por Dominio**:

**Unit Tests**: Lógica de dominio pura (sin dependencias externas)
**Integration Tests**: Facades y repositories con bases de datos de test