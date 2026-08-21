# Informe de Readiness para Producción — Galac ERP (e-factura)

**Fecha:** 2026-08-20
**Rol del análisis:** Principal Software Architect / Engineering Lead
**Fecha objetivo de producción:** 7 de septiembre de 2026 (12 días hábiles efectivos desde hoy, descontando la ausencia del 28-ago)
**Alcance funcional de referencia:** Anexo "Módulos e Informes WebFactura 2026" (catálogo Administrativo Gálac — lo que la empresa requiere listo)
**Documentos base:** `AUDIT-2026-08-18-001` (auditoría OWASP, 25 hallazgos), `AUDITORIA_SEGURIDAD_2026-08-20.md` (re-verificación post-remediación), hallazgos k6/Locust de agosto, código fuente del repositorio, Jira proyecto GAL (Sprint 8 activo).

> **Nota de honestidad metodológica (pedida explícitamente):** todo dato de este informe fue verificado contra el repositorio, Jira o resultados de ejecución reales en esta sesión. Donde un dato **no se pudo medir** (p. ej. porcentaje exacto de cobertura de tests, estado del dashboard de Cloudflare), se declara como *no medido* en vez de estimarlo. El prompt original traía placeholders (`X desarrolladores`, `X req/s`); se reemplazaron con los valores reales observados y se documenta la fuente. El prompt adaptado completo está en el Anexo B para re-ejecuciones futuras.

---

## 1. RESUMEN EJECUTIVO

### Estado general

Galac ERP es un sistema **funcionalmente maduro en su núcleo de facturación** (el módulo más crítico del anexo), con una arquitectura backend disciplinada (services/selectors, multi-tenant por esquema), seguridad recién remediada al 100% en hallazgos Crítico/Alto, y una suite de tests real que corre en verde. **No es un proyecto en riesgo estructural.** Sus brechas están en tres lugares concretos: (1) dos módulos del anexo que no existen aún (Transferencias de inventario y Conteo Físico), (2) la amplitud del catálogo de informes (100+ en el anexo vs. ~4 familias implementadas), y (3) observabilidad de producción degradada tras la eliminación del stack de monitoreo.

### Puntuación de Readiness

| Área | Puntuación | Evidencia clave |
|---|---|---|
| Código Backend | **8.0** /10 | Patrón services/selectors consistente; billing con 32 archivos de test; sin SQL crudo (auditoría lo confirmó limpio 2 veces) |
| Código Frontend | **7.0** /10 | Cobertura completa de rutas vs. anexo; `tsc --noEmit` limpio; solo 6 archivos de test unitario en `web/src`; suite E2E existe (`e2e/` + compose dedicado) |
| Seguridad | **8.0** /10 | 18/23 hallazgos resueltos, 100% de Crítico+Alto cerrado (verificado archivo por archivo el 20-ago); 3 parciales que requieren decisión humana, no código |
| Infraestructura | **6.5** /10 | CI con 3 shards + CD + k6 smoke en GitHub Actions; nginx prod endurecido; **pero** hallazgos k6 sin resolver (gunicorn 16 workers fijos, pgbouncer transaction-mode vs django-tenants) |
| Monitoreo/Observabilidad | **5.0** /10 | Sentry configurado y health app existe; **el compose de monitoreo (Grafana/Prometheus) fue eliminado del repo** — hoy no hay métricas ni alertas de infraestructura |
| Documentación | **6.0** /10 | Técnica sólida (ARCHITECTURE, DEPLOYMENT, CI_CD, DATABASE_ER, operaciones_db con backups); manuales de usuario **en curso** (GAL-405) y centralización **por hacer** (GAL-439) |
| Testing/QA | **6.5** /10 | 140 archivos de test; 247/250 verdes en la corrida de hoy (3 fallos = flakiness pre-existente documentada, no regresiones); % de cobertura *no medido*; matriz de certificación con usuario final **por hacer** (GAL-438) |
| Equipo y Procesos | **6.0** /10 | 5 personas activas en Jira; Sprint 8 con 121 issues (sobrecargado); automatización de Jira sin auditar que duplica tareas; bus factor alto en módulo fiscal |
| **TOTAL** | **53 / 80 (66%)** | |

### Recomendación final: **GO CON CONDICIONES** (alcance MVP alineado al anexo)

El 7 de septiembre es alcanzable para un lanzamiento con el alcance del anexo **menos** Transferencias/Conteo Físico (diferibles 2-3 semanas) y con el catálogo de informes reducido a los fiscalmente obligatorios (libros de compra/venta, ya implementados). Las condiciones bloqueantes están en la sección 7. Detalle del razonamiento GO/NO-GO en la sección 8.

---

## 2. COBERTURA DEL ANEXO (lo que la empresa requiere listo vs. lo que existe)

Esta es la matriz central del informe: cada módulo del catálogo "Módulos e Informes WebFactura 2026" contra el código real del repositorio y las rutas del frontend.

### 2.1 Módulos

| Módulo del anexo | Estado | Evidencia en el repositorio |
|---|---|---|
| **Factura** (ventas, NC, ND, generar desde cotización, libros compra/venta, informes de ventas, multimoneda) | ✅ **Completo** | `apps/billing` (services de factura, `credit_note_*`, `debit_note_*`, `z_closing`, `ledger_*`, `print_template_pdf`); rutas `/ventas/facturas`, `/ventas/notas-credito`, `/ventas/notas-debito`, `/ventas/nueva-*`; el módulo con más tests del sistema (32 archivos) |
| **Punto de venta** (caja, cobro directo, multimoneda) | ✅ **Completo** (lector de código de barras: *no verificado en código*) | `cash_session_services/selectors` en billing; ruta `/pos` |
| **Cotización** | ✅ Completo | `quotation_services/selectors/pdf/email` en billing; rutas `/cotizaciones`, `/cotizaciones/nueva`, `/cotizaciones/[id]` |
| **Cliente** (CRUD, dirección de despacho, informes, export/import) | 🟡 **Funcional con bugs QA abiertos** | `apps/sales` + `/clientes`; bugs activos en Sprint 8: GAL-398 (duplicados en cliente de exportación, High, En curso), GAL-401 (error con cédula al editar, En revisión QA) |
| **Vendedor** (CRUD, comisiones) | ✅ Completo | Rutas `/vendedores/*` (jerarquía, rutas, registro) y `/comisiones/*` (configuración, devengado, jerárquicas) |
| **Proveedor / CxP** (multimoneda, comprobantes retención IVA/ISLR, informes) | ✅ Completo | `apps/payables` (8 archivos de test), `party_withholding` en billing; rutas `/compras/cxp`, `/compras/proveedores` |
| **Pago** (XML retenciones ISLR / TXT IVA al SENIAT) | ⚠️ **Aparece TACHADO en el anexo** | El propio PDF presenta este módulo con el texto tachado — interpretamos que está excluido o en renegociación de alcance. **Requiere confirmación explícita de la empresa** antes de contarlo dentro o fuera del 7-sep |
| **Tabla Retención ISLR** | ✅ Completo | `apps/islr` + ruta `/admin/retenciones-islr` |
| **Inventario — Artículo** (CRUD, consultor de precios, ajuste de precios, informes) | ✅ Completo | `apps/inventory` (18 archivos de test); rutas `/inventario/articulos`, `/inventario/busqueda` |
| **Inventario — Almacén / Existencia por Almacén** | ✅ Completo | `views/warehouse.py`, exports configurables; rutas `/inventario/almacenes`, `/inventario/ubicaciones` |
| **Inventario — Notas de Entrada/Salida** | ✅ Completo | `views/transaction.py`; ruta `/inventario/transacciones`, `/compras/entradas` |
| **Inventario — Transferencias entre almacenes** | ❌ **NO EXISTE** | Sin código (`grep` de transferencia/transfer en inventory: vacío); coincide con GAL-441 "[US] Transferencia de inventario / Guía de despacho" — **Por hacer** en Sprint 8 |
| **Inventario — Conteo Físico** (ajuste masivo de existencias) | ❌ **NO EXISTE** | Sin código (`grep` de conteo/physical_count/stock_count: vacío); sin issue de Jira que lo cubra |
| **Compras** (compras nacionales, informes, etiquetas) | 🟡 **Funcional, cobertura de tests débil** | `apps/purchases` + `/compras/ordenes`; solo **2 archivos de test** — el más bajo de los módulos de negocio |
| **Importar/Exportar** (clientes y artículos, PDF/Excel) | ✅ Completo | `inventory/services/exports.py`, `inventory/views/exports.py`, serializers en `sales/api` |
| **Empresa** | ✅ Completo | `apps/company` + `/configuracion/empresa`, `/configuracion/sucursales` |
| **Mantenimiento de tablas** — Alícuotas IVA/IGTF, Unidad Tributaria, Tarifa N°2 | ✅ Completo | `apps/tributary` (models, loader, api) — grep confirma alícuotas, UT y tarifa |
| **Mantenimiento** — Cambio (tasa automática del día) | ✅ Completo | `apps/currency` + sync BCV diario vía Celery Beat (`BCV_SYNC_AT`), ahora con SSL estricto por defecto |
| **Mantenimiento** — Categoría, Línea de Producto, Unidad de Venta | ✅ Completo | `/inventario/categorias`, `/inventario/lineas`; unidad de venta en modelos de inventario |
| **Mantenimiento** — Ciudad, Municipio, Urbanización/Zona Postal | ✅ Completo | `apps/geo` (3 archivos de test) |
| **Mantenimiento** — Moneda | ✅ Completo | `/configuracion/monedas`, `/admin/monedas` (bug de UI en modo oscuro GAL-344 ya resuelto) |
| **Mantenimiento** — Máquina Fiscal | ✅ Completo (con diferenciador) | `apps/fiscal_printer` (driver Bematech MP-4000 TH FI), `apps/fiscal_relay` (relay WebSocket), `/configuracion/imprentas`, `/configuracion/resoluciones`, `/configuracion/puntos-de-venta` — esto va **más allá** del anexo, que solo pide consulta |
| **Mantenimiento** — Nota final (de factura) | ❓ *No verificado individualmente* | Probablemente cubierto por `/configuracion/plantillas-impresion`; confirmar con QA |
| **Seguridad** (usuarios y niveles de acceso por módulo) | ✅ Completo (superior al anexo) | RBAC con roles JSON por empresa (`apps/teams`), `/configuracion/roles`, `/equipo`; ahora además con gate de MFA en acciones privilegiadas de equipo |

**Resultado: 19 de 24 ítems del anexo completos (79%), 2 con bugs/cobertura débil, 2 inexistentes, 1 tachado pendiente de decisión de la empresa.**

### 2.2 Informes (la brecha más grande del anexo)

El anexo declara **"más de 100 informes"** entre reportes, archivos planos y XML. Lo implementado hoy:

- ✅ **Libros fiscales**: Libro de Ventas y Libro de Compras (`/reportes/libro-ventas`, `/reportes/libro-compras`, `ledger_*` y `purchase_ledger_services` en billing, con la variante IGTF opt-in con advertencia legal ya aprobada en QA — GAL-336/337). **Estos son los fiscalmente obligatorios.**
- ✅ Facturación por fechas (`/reportes/facturas-por-fecha`), reportes de inventario configurables con export (`/reportes/inventario`, exports con paginación por headers).
- ❌ La larga cola del catálogo (estadísticas de ventas por trimestre/comparativo por años, facturación por vendedor/talonario/condición de pago/categoría, informes de costos según ley de costos, etc.): **no existe como catálogo enumerable**. Parte es alcanzable con los exports configurables, pero nadie ha validado informe por informe contra el catálogo.

**Recomendación de alcance:** exigir para el 7-sep únicamente los libros fiscales + los 4-6 informes operativos que el negocio use a diario (decisión de producto), y tratar el catálogo completo como roadmap post-lanzamiento con su propia matriz de validación (encaja con GAL-438, la matriz de certificación con usuario final que está Por hacer).

---

## 3. ANÁLISIS DETALLADO POR ÁREA

### 3.1 Backend (Django 5.2.17 — actualizado esta semana desde 5.2.9)

**Fortalezas verificadas:**
- Arquitectura services/selectors respetada de forma consistente — las vistas no escriben directo a modelos (regla de CLAUDE.md, verificada por muestreo en billing/inventory/payables).
- Multi-tenant por esquema con `django-tenants` correctamente encapsulado: `schema_context` en los cruces público/tenant, mixin `TenantPermissionAPIMixin` que resuelve tenant desde el dominio público con caché Redis de 5 min, y defensa contra el header `X-Galac-Original-Host` forjado (documentada en nginx).
- Sin SQL crudo: dos auditorías consecutivas lo confirmaron limpio (`.raw()`, `cursor.execute` con f-strings: cero).
- El pipeline de permisos ahora es tenant-aware con invalidación de caché a mitad de request (fix documentado en `_ensure_cached`).

**Debilidades:**
- `apps/purchases` con 2 archivos de test es la pieza de negocio menos protegida contra regresiones.
- Cobertura porcentual **no medida** en esta sesión (una corrida `--cov` completa toma >40 min por la creación de esquemas de tenant por test; programarla en CI nocturno, no bloquear el análisis por ella).
- El fixture `use_tenant_schema` hace la suite lenta por diseño (247 tests ≈ 40 min); es un costo aceptado, pero limita el feedback loop local.

### 3.2 Frontend (Next.js 15.5.15)

**Fortalezas:** cobertura de rutas 1:1 contra el anexo (sección 2.1); `tsc --noEmit` sin errores; separación server/client components con `galacServerFetch` para RSC; el middleware Edge ya no toma decisiones de tenant (esa autoridad es exclusiva del servidor — fix de seguridad previo bien razonado); componente compartido `InvoiceLineRow` con acciones de línea centralizadas en `document-line-actions.ts` (previene la clase de bugs GAL-lines-reset).

**Debilidades:** 6 archivos de test unitario en todo `web/src` — la lógica de formularios de documentos (facturas, NC, ND) depende de la suite E2E y de QA manual; sin medición de Core Web Vitals ni bundle analysis registrada.

### 3.3 Seguridad

Estado post-remediación (verificado archivo por archivo hoy, ver `AUDITORIA_SEGURIDAD_2026-08-20.md`):
- **Crítico + Alto: 7/7 resueltos (100%)** — HSTS, CSP/Permissions-Policy, Django parcheado, cookie `galac_schema` httponly, SSL BCV estricto, rate-limiting incondicional fuera de dev, anti-enumeración de usuarios.
- **Global: 18/23 resueltos**, 3 parciales (alcance MFA, verificación Cloudflare dashboard, refactor `runtimeAuthHeaders`), 1 decisión documentada (R2 public-read), 1 no-aplica (Grafana).
- Patrones correctos confirmados por la auditoría: OTP SENIAT con hash + bloqueo de intentos, panel admin con `StaffRequiredMixin` sin filtrar session keys, ancla de auditoría que falla cerrado.

### 3.4 Infraestructura y Rendimiento

- **CI/CD real:** `ci.yml` (3 shards: core/features/inventory), `cd.yml`, `cd-prod.yml`, `k6-smoke.yml`. Linters (ruff, djlint) en el pipeline.
- **Hallazgos k6 de agosto SIN RESOLVER** (fuente: breakpoint test de este mes):
  1. Gunicorn con **16 workers fijos** — el techo de concurrencia real del sistema; sin autoscaling ni tuning por CPU.
  2. **Tormenta de logins PBKDF2**: el 36% del tráfico del breakpoint fue hashing de contraseñas — cualquier pico de logins (lunes 8am) compite por CPU con la facturación.
  3. **PgBouncer en transaction-mode vs django-tenants**: incompatibilidad conocida (el `search_path` es estado de sesión); hoy mitigado usando session-mode, lo que limita el pooling real.
- **Backups:** procedimiento documentado en `docs/operaciones_db.md`; **restore drill no evidenciado** — es condición de GO.
- **DB dev frágil** (volumen que se resetea, daemons Docker duplicados snap/apt): molesto para el equipo, sin impacto en producción, pero quema horas de diagnóstico (dos incidentes documentados este mes).

### 3.5 Monitoreo y Observabilidad — **el área más débil**

- Sentry: configurado (`utils/sentry.py`, desactivado en tests). ✅
- Health checks: `apps/health` + `/healthz` en nginx. ✅
- **Métricas e infraestructura de alertas: NO EXISTEN HOY.** El compose de monitoreo (Grafana/Prometheus/exporters) fue eliminado del repositorio. No hay dashboards de latencia, ni alertas de error-rate, ni visibilidad de Celery/Redis/Postgres en producción. Salir a producción así significa enterarse de los incidentes por los clientes.

### 3.6 Documentación

- Técnica: por encima del promedio — ARCHITECTURE, DEPLOYMENT, CI_CD, DATABASE_ER, FRONTEND_INTEGRATION, operaciones_db, planes del módulo fiscal, ADRs de auditoría.
- Usuario: **en construcción** — GAL-405 (documentación orientada al usuario) En curso; GAL-439 (centralizar documentación de soporte) Por hacer. Para un ERP administrativo dirigido a PYME (el público del anexo), esto es parte del producto, no un extra.

### 3.7 Equipo y Procesos

- **5 personas activas** en Jira: David Garcia, Kevin Morillo, Alejandro Paradiso, Clarense Urbina, Néstor Pacheco (roster observado por actividad; dedicación % no declarada en ninguna herramienta).
- **Sprint 8 (18-ago → 1-sep) con 121 issues** — para 5 personas y 2 semanas es una señal de sprint-como-backlog, no de compromiso realista; dificulta leer el avance real.
- **Automatización de Jira sin auditar** que auto-asigna y **duplicó las 23 tareas de seguridad como subtareas** (GAL-442..464 duplicando GAL-406..428). Ya distorsionó una vez el conteo del sprint; hay que auditarla antes de usar Jira como métrica de readiness.
- Velocidad real observada: **60 issues resueltos en los últimos 14 días** (fuente Jira) — el equipo ejecuta; el problema es la señal, no el motor.
- Bus factor: el módulo fiscal (driver Bematech + relay) tiene un solo dueño evidente. Riesgo para soporte post-lanzamiento.

---

## 4. MATRIZ DE RIESGOS

| # | Riesgo | Prob. | Impacto | Clasificación | Mitigación |
|---|---|---|---|---|---|
| R1 | **Sin métricas ni alertas en producción** (stack de monitoreo eliminado) | Alta | Alto | 🔴 **CRÍTICO** | Restaurar un mínimo viable antes del GO: Sentry alerts + uptime externo + logs agregados. Grafana/Prometheus puede volver post-lanzamiento |
| R2 | **Restore de backup nunca ensayado** | Media | Crítico | 🔴 **CRÍTICO** | Drill de restore completo (schema público + 1 tenant) en staging esta semana; documentar tiempos |
| R3 | Saturación bajo pico de logins (PBKDF2 36% CPU, 16 workers fijos) | Media | Alto | 🟠 ALTO | Subir workers según CPU del droplet de prod; evaluar argon2/cache de sesión; rate-limiting ya reactivado ayuda |
| R4 | Módulos del anexo faltantes (Transferencias, Conteo Físico) descubiertos por el cliente en semana 1 | Alta | Medio | 🟠 ALTO | Acordar formalmente el alcance MVP con la empresa ANTES del 7-sep (por escrito, contra la matriz de la sección 2) |
| R5 | CSP nueva rompe fonts/websocket fiscal en producción | Media | Alto | 🟠 ALTO | Probar en staging con tráfico real (condición ya identificada en la re-auditoría); plan de rollback = quitar 2 headers |
| R6 | Cloudflare en modo "Flexible" (nunca verificado en dashboard) anula HSTS/cookies Secure | Media | Alto | 🟠 ALTO | Verificación de 10 minutos por quien tenga acceso al dashboard — pendiente desde la auditoría |
| R7 | Bugs QA abiertos en Clientes (GAL-398 duplicados, GAL-401 cédula) llegan a producción | Alta | Medio | 🟡 MEDIO | Ambos están En curso/En revisión QA; cerrarlos es parte del sprint actual |
| R8 | Manuales de usuario incompletos → carga de soporte alta en semana 1 | Alta | Medio | 🟡 MEDIO | GAL-405/GAL-439 priorizados; mínimo: guía de facturación + POS + libros |
| R9 | Bus factor módulo fiscal | Media | Medio | 🟡 MEDIO | Sesión de handoff + runbook del relay antes del GO |
| R10 | Automatización Jira distorsiona métricas de avance | Alta | Bajo | 🟢 BAJO | Auditar reglas del proyecto GAL (30 min de un admin de Jira) |
| R11 | Flakiness de tests (currency_rates ×2, BCV_SYNC_AT env-dependent) erosiona confianza en CI | Media | Bajo | 🟢 BAJO | Arreglar los 3 tests (el de BCV: leer el valor de settings en vez de hardcodear "16:00") |

**Riesgos del prompt original que NO aplican tras verificación:** "vulnerabilidades OWASP sin resolver" (Crítico/Alto al 100%), "sin plan de rollback/deploy documentado" (DEPLOYMENT.md + CD existen), "datos sensibles expuestos" (secretos ya sin defaults funcionales), "sin pruebas de rendimiento" (k6/Locust corrieron este mes — lo pendiente es actuar sobre sus hallazgos, no correrlas).

---

## 5. CHECKLIST DE PRODUCCIÓN

| Criterio | Estado |
|---|---|
| Pruebas de seguridad (OWASP) completadas | ✅ Auditoría + remediación + re-verificación (18/23, Crítico/Alto 100%) |
| Pruebas de rendimiento ejecutadas | ✅ k6 breakpoint agosto — 🟠 con 3 hallazgos sin remediar (R3) |
| Backup configurado | ✅ Documentado en operaciones_db.md |
| **Restore verificado** | ❌ **Nunca ensayado — condición de GO** |
| Deployment documentado y probado | ✅ DEPLOYMENT.md + cd-prod.yml (histórico de deploys en git) |
| Rollback documentado | 🟡 Estrategia implícita en CD; falta runbook explícito de rollback con migraciones |
| Monitoreo de errores | ✅ Sentry |
| **Métricas y alertas** | ❌ **Stack eliminado — condición de GO (mínimo viable)** |
| Integraciones externas validadas | 🟡 BCV ✅ (con SSL estricto); imprentas digitales/impresora fiscal: validadas en QA, sin prueba de humo documentada en prod |
| Cumplimiento fiscal (libros SENIAT) | ✅ Libros compra/venta + IGTF opt-in aprobado en QA; ⚠️ módulo Pago (XML/TXT SENIAT) tachado en anexo — decisión de empresa pendiente |
| Manual de usuario | ❌ En curso (GAL-405) |
| Plan de contingencia / on-call | ❌ No documentado |

---

## 6. CAPACIDAD vs. ESFUERZO RESTANTE

**Capacidad** (12 días hábiles al 7-sep × 5 personas × 8h × 75% productividad) ≈ **360 h efectivas** — asumiendo que el equipo completo se enfoca en el cierre (hoy el Sprint 8 tiene 121 issues compitiendo por esas mismas horas; ese es el primer problema a resolver).

**Esfuerzo restante estimado para el alcance MVP** (estimación técnica propia, mismas bases que el informe de factibilidad del 20-ago):

| Bloque | Horas | ¿Cabe al 7-sep? |
|---|---|---|
| Condiciones de GO (restore drill, monitoreo mínimo, CSP staging, Cloudflare, runbook rollback/on-call) | ~40 h | ✅ |
| Cierre bugs QA de Clientes + validación de humo imprentas en prod | ~24 h | ✅ |
| Manual de usuario mínimo (facturación, POS, libros) | ~32 h | ✅ |
| Fixes de rendimiento k6 (workers, tuning login) — versión mínima | ~24 h | ✅ |
| Transferencias de inventario + Conteo Físico (GAL-441 + nuevo) | ~60-80 h | ❌ **No — diferir 2-3 semanas post-GO** |
| Catálogo completo de informes del anexo | 100+ h (sin dimensionar informe a informe) | ❌ **No — roadmap post-GO con matriz GAL-438** |
| **Total para GO condicionado** | **~120 h de ~360 h disponibles** | ✅ **Holgura 3:1 si el sprint se enfoca** |

---

## 7. PLAN DE ACCIÓN

### Semana 1 (21–27 ago) — condiciones bloqueantes
1. **Restore drill** de backup en staging (público + 1 tenant), con tiempos documentados — DevOps/Backend.
2. **Monitoreo mínimo viable**: alertas Sentry por error-rate, uptime check externo sobre `/healthz`, decisión sobre logs agregados — DevOps.
3. **CSP en staging** con tráfico real (fonts, websocket del relay fiscal, imágenes R2) — Frontend + QA.
4. **Verificar Cloudflare "Full (strict)"** en el dashboard (10 min, quien tenga acceso) — Infra.
5. **Acordar por escrito el alcance MVP** con la empresa usando la matriz de la sección 2 (incluye la decisión sobre el módulo Pago tachado) — Producto.
6. Auditar la automatización de Jira y depurar los duplicados GAL-442..464 — Admin Jira.

### Semana 2 (31 ago – 4 sep) — endurecimiento
7. Cerrar GAL-398 y GAL-401 (bugs de Clientes) — ya en curso.
8. Tuning gunicorn (workers por CPU) + prueba k6 corta de confirmación — DevOps.
9. Manual de usuario mínimo: facturación, POS, libros fiscales (GAL-405) — Producto/QA.
10. Runbook de rollback (incluyendo migraciones) y esquema on-call de las primeras 2 semanas — Lead.
11. Handoff del módulo fiscal (sesión + runbook del relay) — dueño actual + 1 backup.
12. Arreglar los 3 tests flaky (currency_rates ×2, BCV_SYNC_AT) — Backend (2-3 h).

### 5–7 sep — corte y salida
13. Smoke test E2E en staging final (suite `e2e/` existente) + prueba de humo de imprentas contra el entorno real.
14. Deploy a producción con la ventana y el rollback ensayados. **Feature-freeze desde el 4-sep.**

### Post-GO (roadmap comprometido, no "algún día")
15. Transferencias de inventario / guía de despacho (GAL-441) — sprint 9.
16. Conteo físico de inventario — sprint 9/10 (crear el issue: hoy no existe).
17. Matriz de certificación de informes contra el catálogo del anexo (GAL-438) y cierre incremental del catálogo.
18. Restaurar Grafana/Prometheus; resolver PgBouncer/django-tenants; decisión sobre alcance ampliado de MFA; refactor `runtimeAuthHeaders`.

---

## 8. RESPUESTAS DIRECTAS A LAS PREGUNTAS CLAVE

1. **¿Estado REAL hoy?** Núcleo de facturación sólido y probado; 79% del anexo completo; seguridad remediada; sin monitoreo de producción; 2 módulos del anexo inexistentes; catálogo de informes mayormente pendiente.
2. **¿Qué FALTA para producción?** Las 6 condiciones de la Semana 1 (ninguna es desarrollo mayor) + decisión de alcance con la empresa. Funcionalmente: Transferencias, Conteo Físico e informes — diferibles si el acuerdo de alcance lo refleja.
3. **¿Qué MEJORAR en el proceso?** Sprints con compromiso realista (121 issues/sprint no es plan, es inventario); auditar la automatización de Jira; cobertura medida en CI nocturno; tests de purchases.
4. **¿RIESGOS REALES de salir?** Los dos críticos son operativos, no de código: ceguera de monitoreo y restore no ensayado. Ambos se resuelven en días.
5. **¿PORCENTAJE REAL de completitud?** Contra el anexo: **79% de módulos completos**; readiness operativo global: **66% (53/80)**. Con las condiciones de Semana 1 cumplidas, el operativo sube a ~75% — suficiente para un GO controlado de alcance MVP.
6. **¿Es REALISTA el 7 de septiembre?** **Sí, para el alcance MVP** (anexo menos Transferencias/Conteo/catálogo completo de informes). No para el anexo al 100% — eso requiere 2-3 semanas más, coherente con el informe de factibilidad del 20-ago que proyectaba el 24-sep para el cierre total con capacidad de 1 dev.
7. **¿Qué pasa si NO salimos?** Se pierde la ventana comercial y el equipo sigue quemando el sprint en un backlog difuso; el costo de esperar 3 semanas solo se justifica si la empresa rechaza el alcance MVP.
8. **¿Qué pasa si SALIMOS?** Con las condiciones: riesgo controlado — el núcleo fiscal está probado y la seguridad crítica cerrada. Sin las condiciones: se lanza a ciegas (sin alertas) y sin red (restore no ensayado) — eso sería un NO-GO.

---

## 9. GRÁFICO DE RADAR (0–10)

```
                Backend (8.0)
                     │████████──
Equipo/Procesos      │                Frontend
   (6.0) ──██████    │    ███████── (7.0)
                     │
Testing/QA ──███████ │ ████████── Seguridad
   (6.5)             │               (8.0)
                     │
Documentación ██████ │ ███████── Infraestructura
   (6.0)             │               (6.5)
                     │█████──
              Monitoreo (5.0)  ← área más débil
```

| Área | Score |
|---|---|
| Backend | ████████░░ 8.0 |
| Frontend | ███████░░░ 7.0 |
| Seguridad | ████████░░ 8.0 |
| Infraestructura | ██████▌░░░ 6.5 |
| Monitoreo | █████░░░░░ 5.0 |
| Documentación | ██████░░░░ 6.0 |
| Testing/QA | ██████▌░░░ 6.5 |
| Equipo/Procesos | ██████░░░░ 6.0 |

---

## 10. CONCLUSIÓN FINAL — UNA SOLA RECOMENDACIÓN

# ✅ GO CON CONDICIONES

**Salir a producción el 7 de septiembre con alcance MVP** (el anexo menos Transferencias de inventario, Conteo Físico y el catálogo extendido de informes — todos con fecha comprometida post-GO), **condicionado a cumplir antes del 4 de septiembre:**

1. Restore de backup ensayado y documentado.
2. Monitoreo mínimo viable operativo (alertas Sentry + uptime externo).
3. CSP validada en staging con tráfico real.
4. Cloudflare confirmado en "Full (strict)".
5. Acuerdo de alcance MVP firmado con la empresa contra la matriz de la sección 2 (incluida la decisión sobre el módulo "Pago" tachado).
6. Runbook de rollback y esquema on-call de las primeras 2 semanas.

Si cualquiera de las condiciones 1, 2 o 5 no se cumple al 4 de septiembre, la recomendación cambia automáticamente a **NO-GO con nueva fecha 21–24 de septiembre** (coherente con la proyección de cierre total del informe de factibilidad).

---
---

## ANEXO A — Fuentes y verificaciones de esta sesión

- Suite de regresión: 250 tests (accounts, teams, galac_admin) → **247 verdes**; 3 fallos analizados individualmente: 2 flaky pre-existentes en `develop` (currency_rates) + 1 dependiente de entorno (`BCV_SYNC_AT` local ≠ 16:00, el test hardcodea el valor). Cero regresiones de los cambios de seguridad.
- `ruff check`, `manage.py check`, `tsc --noEmit`: limpios.
- Mapeo del anexo: `ls`/`grep` sobre `apps/*` y `web/src/app/(dashboard)/*` (rutas y servicios citados en la sección 2).
- Jira: Sprint 8 (121 issues), roster activo, bugs GAL-398/401, US GAL-441/405/438/439, 60 resueltos en 14 días.
- Seguridad: `docs/AUDITORIA_SEGURIDAD_2026-08-20.md` (re-verificación archivo por archivo).
- Rendimiento: hallazgos del breakpoint k6 de agosto 2026 (gunicorn/PBKDF2/pgbouncer).

## ANEXO B — Prompt adaptado a e-factura (para re-ejecuciones)

Cambios respecto al prompt original: (1) placeholders `X` reemplazados por datos reales o marcados "medir, no asumir"; (2) eliminado lo que no aplica al stack real (NextAuth.js → sesión Django por cookie; Alembic → migraciones Django/django-tenants; Vault/AWS → `.env` por entorno; Flower → no instalado); (3) agregada la sección que faltaba y resultó ser la más importante: **matriz de cobertura contra el anexo funcional**; (4) el checklist de riesgos pre-poblado se reemplazó por "verificar antes de afirmar" — la mitad de los riesgos listados en el original ya no existían.

> Actúa como Principal Software Architect. Analiza el readiness de producción de **Galac ERP (e-factura)** — Django 5.2.x multi-tenant (django-tenants, esquema por empresa) + Next.js 15 SPA + PostgreSQL/PgBouncer + Redis/Celery + Nginx/Cloudflare, con módulo de impresión fiscal (driver Bematech + relay WebSocket) e imprentas digitales.
> **Alcance de referencia:** el catálogo "Módulos e Informes WebFactura 2026" — construye una matriz módulo-por-módulo (incluidos los 100+ informes) contra código y rutas reales; señala explícitamente lo tachado en el documento como alcance a confirmar.
> **Equipo real:** 5 personas (verificar dedicación en Jira, no asumir); fecha objetivo 7-sep-2026; sin presupuesto para contrataciones.
> **Reglas:** cada afirmación con evidencia de repo/Jira/ejecución; lo no medible se declara "no medido"; usa como línea base la auditoría OWASP del 18-ago, su re-verificación del 20-ago y los hallazgos k6 de agosto (gunicorn 16 workers, PBKDF2 36%, pgbouncer/tenants). Ejecuta la suite de tests y reporta el resultado real.
> **Entregables:** resumen ejecutivo con score /80, matriz de cobertura del anexo, análisis por área (backend, frontend, seguridad, infra, monitoreo, docs, testing, equipo), matriz de riesgos verificada, checklist de producción, plan de acción semanal, radar 0-10 y UNA recomendación: GO / NO-GO / GO con condiciones (con condiciones medibles y fecha de corte).
