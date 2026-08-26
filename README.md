#Control Total — Documentación Técnica de Arquitectura

## 1. Arquitectura General

El sistema es multi-tenant, event-driven y desacoplado. La API síncrona solo recibe requests y encola trabajo. Todo procesamiento pesado ocurre en workers asíncronos. Los servicios se comunican por un bus de eventos, no por llamadas directas.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────────────────┐
│   Cliente   │────▶│  API Layer  │────▶│  Redis (Celery Broker)      │
│  Next.js    │     │  FastAPI    │     └─────────────────────────────┘
└─────────────┘     └─────────────┘                   │
                                                      ▼
                                            ┌─────────────────┐
                                            │  Celery Workers │
                                            │  (3 pools)      │
                                            │                 │
                                            │ • discovery     │
                                            │ • analysis      │
                                            │ • alerts        │
                                            │ • offboarding   │
                                            └─────────────────┘
                                                      │
                    ┌─────────────────┬───────────────┼───────────────┐
                    ▼                 ▼               ▼               ▼                 
            ┌─────────────┐   ┌─────────────┐ ┌────────────   ┌─────────────┐
            │ PostgreSQL  │   │   Kafka     │ │   Vault     │ │   MinIO     │
            │  +Timescale │   │  (Eventos)  │ │  (Secrets)  │ │   (S3)      │
            └─────────────┘   └─────────────┘ └─────────────┘ └─────────────┘   
```

---

## 2. Capa de Entrada

### API Gateway (Kong)

Punto único de entrada. Maneja rate limiting por tenant, validación de JWT, terminación TLS y reglas de WAF. No ejecuta lógica de negocio; solo autentica, autoriza y enruta hacia los servicios internos.

### API REST (FastAPI)

Expone endpoints para CRUD de suscripciones, usuarios, reportes y configuración de organizaciones. Usa Pydantic para validación estricta de schemas. Es stateless; cualquier instancia puede atender cualquier request.

---

## 3. Background Workers (Celery + Redis)

Redis actúa como broker de tareas y como backend de resultados. Celery ejecuta workers en pools separados por tipo de carga, para aislar el procesamiento pesado de la API.

### Pool de Discovery

Escanea todas las fuentes de gasto de la empresa.

- **Plaid/Yodlee Sync**: Conecta a agregadores bancarios vía OAuth. Obtiene transacciones de los últimos 90 días en modo read-only. El access token se almacena en Vault, no en la base de datos de la aplicación.
- **Email Scanner**: Se conecta a bandejas de email corporativas (Gmail, Outlook) vía OAuth con scope de solo lectura. Ejecuta búsquedas filtradas por keywords (receipt, invoice, subscription, renewal) y extrae datos estructurados del cuerpo del email y sus adjuntos PDF.
- **SSO Directory Sync**: Lee el directorio de identidad (Okta, Azure AD, Google Workspace) vía SCIM o LDAP. Mapea qué aplicaciones SaaS están provisionadas, cuántos usuarios tienen acceso y cuándo fue el último login.
- **HRIS Sync**: Mantiene sincronizado el catálogo de empleados activos. Cuando un empleado es dado de baja en el sistema de RRHH, este worker emite un evento que dispara el offboarding engine.

### Pool de Analysis

Procesa los datos crudos descubiertos y genera inteligencia.

- **Recurrence Detector**: Analiza transacciones históricas para identificar patrones de suscripción. Usa heurísticas de intervalo regular (mensual, trimestral, anual), consistencia de monto y una lista de merchants conocidos como recurrentes. Los resultados se persisten como borradores pendientes de validación humana.
- **Duplicate Detector**: Compara suscripciones activas por similitud semántica de nombre, categoría funcional y overlap de usuarios. Genera alertas cuando dos herramientas hacen lo mismo.
- **Unused License Detector**: Cruza los datos del SSO (quién tiene acceso y cuándo usó la app por última vez) contra las suscripciones pagadas. Identifica licencias asignadas a empleados inactivos o que nunca iniciaron sesión.
- **Spending Analyzer**: Genera proyecciones de gasto, detecta anomalías comparando mes a mes y calcula benchmarks por categoría.

### Pool de Alerts

Monitorea el estado del sistema y notifica a los stakeholders.

- **Renewal Monitor**: Revisa diariamente las fechas de renovación. Alerta a 90, 60 y 30 días con contexto completo: costo, nivel de uso, duplicados detectados y recomendación de acción. Para renovaciones anuales mayores a $5,000, bloquea la renovación automática y crea un workflow de aprobación.
- **Approval Gate**: Intercepta transacciones de nueva contratación. Pausa el pago y genera un flujo de aprobación secuencial: manager → IT → finanzas. Solo después de todas las aprobaciones, la suscripción se activa y se registra en el inventario.

### Pool de Offboarding

Gestiona la baja de accesos cuando un empleado renuncia.

- **Revocation Orchestrator**: Recibe el evento de baja desde el HRIS sync. Identifica todas las suscripciones donde el empleado tiene acceso. Genera un checklist de tareas de revocación, asigna cada tarea al administrador de la herramienta correspondiente y rastrea su completitud.
- **Revocation Monitor**: Revisa cada 12 horas el estado del checklist. Si una tarea sigue pendiente, re-notifica al administrador y escala al manager si supera las 48 horas. Cuando todo está completo, genera un certificado de offboarding auditado.

---

## 4. Capa de Datos

### PostgreSQL + TimescaleDB

PostgreSQL es la base de datos transaccional principal. Almacena organizaciones, usuarios, suscripciones, tareas, workflows de aprobación y configuraciones.

TimescaleDB (extensión de PostgreSQL) se usa para las series temporales de gasto: cada transacción descubierta se guarda con timestamp, permitiendo consultas analíticas rápidas sobre evolución de gasto mensual, proyecciones y detección de anomalías.

La segregación entre tenants se hace por row-level security (RLS): cada tabla tiene un campo `organization_id` y las políticas de PostgreSQL garantizan que un query solo vea filas de su propio tenant.

### Redis

Cumple tres roles:
- **Broker de Celery**: Encola tareas y distribuye workers.
- **Backend de resultados**: Almacena resultados de tareas async para que la API los consulte.
- **Cache de sesiones y rate limits**: Reduce carga en PostgreSQL para datos de alta frecuencia de lectura.

### Apache Kafka

Bus de eventos entre servicios. No se usa para comunicación síncrona, sino para notificar que algo ocurrió para que otros servicios reaccionen.

Eventos clave:
- `discovery.completed`: El analysis engine reacciona procesando las nuevas transacciones.
- `subscription.created`: El alert engine programa renovaciones y el audit logger registra la acción.
- `employee.offboarded`: El offboarding engine inicia revocaciones.
- `approval.required`: El alert engine notifica a los aprobadores correspondientes.

Kafka garantiza durabilidad (los eventos persisten por 7 días) y permite replay: si un servicio cae, puede reconstruir su estado leyendo los eventos desde el último offset confirmado.

### ClickHouse

Base de datos columnar para logs de auditoría de alto volumen. Almacena quién accedió a qué dato, cuándo, desde qué IP, con qué user-agent, y si la acción fue exitosa o fallida.

Su diseño columnar permite consultas analíticas rápidas sobre millones de registros: "muéstrame todos los accesos a datos financieros del tenant X en el último mes". Retención de 7 años para compliance.

### MinIO / S3

Almacenamiento de objetos para:
- Archivos de estado de cuenta subidos por usuarios (PDF, CSV, OFX).
- Backups encriptados de PostgreSQL.
- Certificados de offboarding generados.

Usa versionado de objetos y cifrado en reposo con SSE-S3.

### HashiCorp Vault

Gestor de secretos centralizado. Almacena:
- Access tokens de Plaid/Yodlee.
- Tokens OAuth de email y SSO.
- Claves de cifrado AES-256.
- Credenciales de base de datos (rotación dinámica).

Vault nunca expone un secret en texto plano a la aplicación de forma permanente. La app solicita un token de corta duración (lease) y Vault lo revoca automáticamente al expirar.

---

## 5. Seguridad

### Zero-Trust y Autenticación Mutua

Ningún servicio confía en otro por defecto. Toda comunicación interna usa mTLS (certificados X.509). Los JWT de usuario expiran en 15 minutos y se refrescan con rotating refresh tokens.

### Cifrado

- **En tránsito**: TLS 1.3 obligatorio para todo tráfico externo e interno.
- **En reposo**: PostgreSQL usa cifrado a nivel de disco (LUKS). Los campos sensibles (costos, tokens) se cifran adicionalmente a nivel de aplicación con AES-256 antes de persistirse.
- **En Vault**: Todos los secrets se cifran con AES-256-GCM con claves maestras en HSM.

### Gestión de Acceso

- **RBAC**: Roles predefinidos (admin, finance, IT, viewer) con permisos granulares por recurso.
- **MFA**: Obligatorio para todos los usuarios con acceso a datos financieros.
- **OAuth mínimo**: Los conectores a terceros solicitan solo los scopes estrictamente necesarios. Nunca se pide permiso de escritura en una fuente de solo lectura.

### Auditoría

Todo acceso a datos sensibles se registra en ClickHouse con:
- Timestamp preciso.
- ID de usuario y organización.
- IP origen y user-agent.
- Recurso accedido y acción ejecutada.
- Resultado (éxito o fallo).

### Compliance

| Estándar | Estado | Medidas |
|----------|--------|---------|
| SOC 2 Type II | Roadmap Q4 | Controles de acceso, monitoreo continuo, incident response, backups testeados |
| ISO 27001 | Roadmap Q4 | ISMS documentado, gestión de riesgos, políticas de seguridad |
| PCI DSS v4.0 | No aplica | La plataforma no almacena, procesa ni transmite PANs. Usa tokenización de terceros (Plaid/Stripe). Si en el futuro se agrega procesamiento de pagos, se implementará SAQ A con hosted payment page |
| GDPR / LGPD | Implementado | Derecho al olvido, portabilidad de datos, consentimiento explícito, DPO designado, retención definida |

---

## 6. Seguimiento de Suscripciones: Enfoque Realista

### El problema real con las APIs bancarias

En Estados Unidos y Canadá, Plaid y Yodlee cubren aproximadamente el 60-70% de las instituciones financieras. En Latinoamérica, la cobertura cae por debajo del 10%. Los bancos corporativos en la región no ofrecen APIs abiertas estandarizadas; quienes lo hacen requieren acuerdos comerciales de 4 a 8 semanas, fees por transacción sincronizada y no siempre permiten acceso de lectura a cuentas empresariales por políticas de seguridad interna.

Por eso, la plataforma no depende exclusivamente de APIs bancarias. El diseño es multi-fuente y tolerante a fallas: si una fuente no está disponible, las otras compensan.

### Estrategia multi-fuente

**Tier 1 — Automático (Plaid, Yodlee, Open Banking, Corporate Card APIs)**

Cuando están disponibles, son la fuente más confiable. Plaid maneja el flujo de autenticación OAuth del banco y entrega un access token de solo lectura. La plataforma nunca ve ni almacena las credenciales bancarias del usuario; solo el token revocable que Plaid proporciona. Este token se almacena en Vault con rotación automática cada 90 días.

**Tier 2 — Semi-automático (Email Scanning, SSO Directory, HRIS)**

Esta es la fuente principal para mercados sin Open Banking maduro.

- **Email Scanning**: Se conecta a Gmail o Outlook vía OAuth con scope de solo lectura. No lee todos los emails; ejecuta búsquedas filtradas por keywords predefinidas (receipt, invoice, subscription, renewal, pago confirmado) y solo procesa mensajes que coinciden. Extrae monto, merchant, fecha y descripción usando regex y NLP. Los emails procesados se marcan para evitar re-procesamiento.

- **SSO Directory**: Se conecta al proveedor de identidad (Okta, Azure AD, Google Workspace) vía SCIM 2.0 o LDAP. Lee qué aplicaciones SaaS están provisionadas, cuántos usuarios tienen acceso y cuándo fue el último login. Esto detecta suscripciones que el departamento de IT conoce pero que finanzas no tiene registradas.

- **HRIS Sync**: Mantiene sincronizado el catálogo de empleados activos. Cuando alguien es dado de baja, el sistema sabe exactamente qué accesos debe revocar.

**Tier 3 — Manual (Statement Upload, Manual Entry)**

Cuando ninguna fuente automática está disponible, el usuario sube estados de cuenta en PDF, CSV u OFX. El sistema extrae tablas de transacciones con OCR y las procesa igual que las transacciones de Plaid. Los resultados se guardan como borradores que el usuario valida antes de persistir.

### Fusión de fuentes (Entity Resolution)

Cuando una misma suscripción aparece en múltiples fuentes (ej: Adobe Creative Cloud detectada por Plaid, por email scanning y por SSO), el sistema las fusiona en un único registro canónico. El algoritmo compara similitud de nombre normalizado, consistencia de monto y frecuencia. Elige como registro base el que tenga más fuentes confirmadas y mayor confianza. El registro resultante incluye metadatos de todas las fuentes que lo corroboran.

### Detección de recurrencia

El motor de análisis recibe un conjunto de transacciones y agrupa por merchant normalizado. Para cada grupo con al menos dos transacciones, calcula los intervalos entre fechas consecutivas. Si los intervalos son consistentemente mensuales (25-35 días), trimestrales (85-95 días) o anuales (360-370 días), y los montos varían menos del 15%, se clasifica como suscripción con un score de confianza. Si el merchant está en una lista de servicios conocidos como recurrentes (Stripe, Adobe, Microsoft, AWS, etc.), el score aumenta. Solo los registros con score mayor a 0.7 se proponen como suscripciones.

---

## 7. Stack Tecnológico Resumido

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| Frontend | Next.js 14 + TypeScript | SSR, tipado fuerte, SEO |
| API REST | FastAPI | Alto rendimiento, validación automática, OpenAPI nativo |
| API GraphQL | Strawberry | Tipado nativo en Python, integración con FastAPI |
| Workers | Celery + Redis | Procesamiento async probado, colas por prioridad |
| Event Bus | Apache Kafka | Durabilidad, replay de eventos, particionamiento por tenant |
| Base de datos | PostgreSQL 15 + TimescaleDB | ACID, JSONB flexible, series temporales nativas |
| Cache | Redis 7 | Sesiones, rate limits, cola de tareas |
| Almacenamiento | MinIO / AWS S3 | Compatible S3, cifrado en reposo, versionado |
| Auth | Keycloak | SSO, MFA, RBAC, federación de identidades, sin vendor lock-in |
| Secrets | HashiCorp Vault | Rotación automática, leases de corta duración, políticas de acceso |
| Auditoría | ClickHouse | Columnar, alto volumen, consultas analíticas rápidas |
| Observabilidad | Grafana + Prometheus + Loki | Métricas, logs y traces en un stack unificado |
| Infraestructura | Kubernetes + Terraform | IaC, auto-scaling, blue-green deployments |
| CI/CD | GitHub Actions | Tests, SAST, build, deploy automatizado |

---

*Documento técnico v3.0 — Control Total*
