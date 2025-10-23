
# Caso 2

Integrantes: 
- Josué Salazar
- Javier Rodríguezz
- Jose David Chavez 
Curso: Diseño de Software

---

## Métricas de los requerimientos no funcionales

### Performance
Justificación metodológica: Las métricas de performance se calculan basándose en estándares de la industria para aplicaciones web empresariales y sistemas de mercadeo digital que requieren respuestas rápidas para mantener la productividad de los usuarios.​
#### **Portal Web - Tiempo de Respuesta**

- **Tiempo promedio para operaciones estándar**: ≤ 2.0 segundos (target), máximo 2.5 segundos (requerimiento)[Link](https://kpidepot.com/kpi/query-response-time)​
- **Percentil 95**: ≤ 2.4 segundos
- **Percentil 99**: ≤ 3.0 segundos
- **Tecnología**: Next.js en Vercel con edge functions, PostgreSQL con índices optimizados
- **Métrica de validación**: Usar Lighthouse CI y herramientas de Real User Monitoring (RUM)
- **Justificación**: Los estudios indican que respuestas bajo 2 segundos son percibidas como instantáneas, mientras que sobre 3 segundos incrementan significativamente el abandono del usuario[Link](https://odown.com/blog/api-response-time-standards/)  

#### **Redis Cache - Consultas Cacheadas**
- **Tiempo de respuesta target**: ≤ 300 milisegundos
- **Tiempo de respuesta máximo**: 400 milisegundos (requerimiento original)
- **Operaciones simples (GET)**: ≤ 100 milisegundos
- **Latencia sub-millisecond**: Para operaciones en memoria caliente
- **Tecnología**: Redis Cloud con clustering, configuración de persistent connections
- **Justificación**: Benchmarks de Redis demuestran latencias sub-milisegundo para operaciones en memoria. El objetivo de 300ms incluye overhead de red y serialización[Link](https://redis.io/blog/redisjson-public-preview-performance-benchmarking/)

#### **Generación de Contenido con IA**
- **Solicitudes simples**: ≤ 7 segundos (generación de textos cortos, sugerencias)
- **Solicitudes complejas**: ≤ 20 segundos (campañas completas, análisis profundo)
- **Timeout threshold**: 30 segundos
- **Tecnología**: OpenAI API con streaming, Azure OpenAI Service, caching agresivo de prompts similares
- **Justificación**: Las llamadas a APIs de LLM varían entre 3-15 segundos dependiendo de la longitud del token y complejidad del modelo

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

## Domain driven desing (En eso estoy ahorita xd)

