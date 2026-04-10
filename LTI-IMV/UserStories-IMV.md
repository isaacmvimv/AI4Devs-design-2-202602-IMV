# LTI ATS — User Stories · MVP v1.0

> **Autor:** Product Owner · **Basado en:** LTI PRD v1.0 · **Estado:** Draft
>
> **Roles del sistema:**
> - **Reclutador** — miembro de RR.HH. que gestiona el proceso de selección.
> - **Responsable de Selección** — manager que toma decisiones finales de contratación.
> - **Entrevistador** — miembro del panel técnico o funcional.
> - **Candidato** — persona que solicita un puesto.
> - **Admin** — administrador del tenant/empresa en LTI.

---

## Nota sobre el análisis de gaps del MVP

Durante el análisis del PRD se han detectado los siguientes aspectos **no documentados explícitamente** pero **imprescindibles para un MVP funcional**:

| # | Gap detectado | US generada |
|---|---|---|
| G1 | Sin flujo de autenticación / gestión de usuarios | US-01 |
| G2 | Sin gestión de notificaciones in-app al candidato (más allá del portal) | US-11 |
| G3 | Sin flujo de onboarding / configuración inicial del tenant | US-02 |
| G4 | Sin gestión explícita de permisos/roles dentro del tenant | Incluida en US-01 y US-02 |

---

## Bloque 1 — User Stories

---

### US-01 · Autenticación y gestión de acceso

**Título:** Iniciar sesión de forma segura en LTI

**Historia:**
> Como **usuario de LTI** (reclutador, responsable, entrevistador o admin),
> quiero poder registrarme e iniciar sesión con mis credenciales corporativas,
> para acceder únicamente a las funcionalidades que mi rol permite.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ✅ No depende de otras US para existir |
| Negotiable | ✅ El mecanismo de auth (email/password vs SSO) es negociable para MVP |
| Valuable | ✅ Sin auth no hay producto |
| Estimable | ✅ Implementación estándar con librerías probadas |
| Small | ✅ Acotable a email+password en MVP; SSO en fase 2 |
| Testable | ✅ Criterios de acceso/rechazo claros |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — inicio de sesión exitoso
Scenario: Usuario registrado inicia sesión con credenciales válidas
  Given el usuario existe en el sistema con email "ana@empresa.com" y contraseña válida
  And el usuario está asignado al rol "Reclutador"
  When introduce sus credenciales correctas y pulsa "Iniciar sesión"
  Then accede al dashboard del reclutador
  And se genera un token JWT con expiración de 8 horas
  And se registra el evento en REGISTRO_AUDITORIA

# Edge case — sesión expirada
Scenario: Usuario intenta acceder con token expirado
  Given el usuario tiene un token JWT caducado
  When realiza una petición autenticada a cualquier endpoint
  Then recibe un error 401 "Sesión expirada"
  And es redirigido a la pantalla de login

# Error — credenciales incorrectas
Scenario: Usuario introduce contraseña incorrecta
  Given el usuario existe en el sistema
  When introduce una contraseña errónea 3 veces consecutivas
  Then su cuenta queda bloqueada temporalmente durante 15 minutos
  And recibe un email de aviso con opción de resetear contraseña

# Edge case — usuario inactivo
Scenario: Admin desactiva una cuenta de usuario
  Given el admin ha marcado el usuario como inactivo (activo = false)
  When el usuario intenta iniciar sesión
  Then recibe el mensaje "Cuenta desactivada. Contacta con tu administrador."
  And no se genera ningún token de sesión
```

**Estimación de complejidad:** `M`

---

### US-02 · Configuración inicial del tenant

**Título:** Configurar la empresa y los roles del equipo de selección

**Historia:**
> Como **Admin**,
> quiero configurar los datos de la empresa, crear usuarios y asignarles roles,
> para que el equipo de selección pueda operar en LTI desde el primer día.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ✅ Independiente del resto de flujos |
| Negotiable | ✅ El alcance del onboarding es negociable (wizard vs manual) |
| Valuable | ✅ Prerequisito operativo de toda la plataforma |
| Estimable | ✅ CRUD estándar de usuarios y configuración |
| Small | ✅ Acotable a creación de empresa + usuarios en MVP |
| Testable | ✅ Validaciones de datos y permisos claros |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — crear empresa y primer admin
Scenario: Admin crea la configuración inicial de la empresa
  Given el admin accede por primera vez con credenciales de activación
  When introduce el nombre de empresa, sector y subdominio
  Then se crea el registro EMPRESA en base de datos
  And se asigna el subdominio único al tenant
  And el admin puede invitar a los primeros usuarios por email

# Happy path — asignar rol a usuario
Scenario: Admin asigna el rol "Reclutador" a un nuevo usuario
  Given existe un usuario pendiente de activación
  When el admin le asigna el rol "Reclutador" y pulsa "Guardar"
  Then el usuario recibe email de bienvenida con enlace de activación
  And al activar la cuenta, accede al dashboard con los permisos del rol "Reclutador"

# Edge case — subdominio duplicado
Scenario: Admin intenta registrar un subdominio ya en uso
  Given el subdominio "acme" ya está asignado a otra empresa
  When el admin intenta crear la empresa con subdominio "acme"
  Then recibe el error "Subdominio no disponible"
  And debe elegir uno alternativo

# Error — invitación a email corporativo no válido
Scenario: Admin invita a un usuario con email de dominio no corporativo
  Given la empresa tiene configurada la restricción de dominio "@empresa.com"
  When el admin intenta invitar a "usuario@gmail.com"
  Then recibe el aviso "El email no pertenece al dominio corporativo permitido"
```

**Estimación de complejidad:** `M`

---

### US-03 · Creación de requisición de empleo

**Título:** Crear una nueva requisición de empleo con flujo de aprobación

**Historia:**
> Como **Reclutador**,
> quiero crear una requisición de empleo con título, descripción, requisitos y banda salarial,
> para iniciar el proceso de contratación de forma estructurada y trazable.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ✅ No depende de otras US de candidatos |
| Negotiable | ✅ El flujo de aprobación puede simplificarse en MVP |
| Valuable | ✅ Es el punto de entrada de todo el proceso |
| Estimable | ✅ Formulario + workflow de estados |
| Small | ✅ Alcance claro y acotado |
| Testable | ✅ Validaciones de campos y transiciones de estado |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — crear requisición
Scenario: Reclutador crea una requisición completando todos los campos
  Given el reclutador está autenticado en LTI
  When completa el formulario de nueva requisición con título, departamento,
       ubicación, tipo de contrato, banda salarial y descripción
  And pulsa "Crear Requisición"
  Then se crea el registro REQUISICION_EMPLEO con estado "Borrador"
  And se genera un evento en REGISTRO_AUDITORIA
  And el reclutador recibe confirmación visual en pantalla

# Happy path — requisición enviada a aprobación
Scenario: Reclutador envía la requisición al Responsable para aprobación
  Given existe una requisición en estado "Borrador"
  When el reclutador pulsa "Enviar para aprobación"
  Then la requisición cambia de estado a "Pendiente de aprobación"
  And se notifica al Responsable de Selección asignado

# Edge case — campo descripción vacío
Scenario: Reclutador intenta crear requisición sin descripción del puesto
  Given el reclutador está rellenando el formulario de requisición
  When deja el campo "Descripción" vacío y pulsa "Crear"
  Then el formulario muestra el error "La descripción del puesto es obligatoria"
  And la requisición no se crea

# Error — banda salarial incoherente
Scenario: El salario mínimo es mayor que el máximo
  Given el reclutador introduce salario_min = 50.000 y salario_max = 30.000
  When pulsa "Crear Requisición"
  Then recibe el error "El salario mínimo no puede superar al máximo"
```

**Estimación de complejidad:** `M`

---

### US-04 · Publicación multicanal de la oferta de empleo

**Título:** Publicar una oferta en múltiples portales de empleo con un solo clic

**Historia:**
> Como **Reclutador**,
> quiero publicar una oferta aprobada en LinkedIn, Indeed y el portal corporativo simultáneamente,
> para maximizar la captación de candidatos sin duplicar el trabajo manual.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-03 (requisición aprobada) |
| Negotiable | ✅ Los canales incluidos en MVP son negociables |
| Valuable | ✅ Core del proceso de atracción de talento |
| Estimable | ✅ Integraciones con APIs conocidas |
| Small | ⚠️ Puede necesitar subdivisión por canal en la implementación |
| Testable | ✅ Verificable por estado de publicación y URL externa |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — publicación simultánea
Scenario: Reclutador publica oferta en LinkedIn e Indeed a la vez
  Given existe una requisición en estado "Aprobada"
  When el reclutador selecciona los canales "LinkedIn" e "Indeed" y pulsa "Publicar"
  Then se crean dos registros PUBLICACION_EMPLEO (uno por canal) con estado "Activa"
  And cada registro contiene la url_externa devuelta por el portal
  And el reclutador ve el estado de publicación por canal en tiempo real

# Happy path — publicación en portal corporativo
Scenario: La oferta se publica automáticamente en el site de carreras corporativo
  Given el canal "Portal Corporativo" está habilitado por defecto en la configuración
  When se aprueba y publica cualquier requisición
  Then se genera automáticamente la publicación en el portal con la marca de la empresa

# Edge case — canal con credenciales caducadas
Scenario: Las credenciales de integración con Indeed han expirado
  Given la integración con Indeed tiene token de API caducado
  When el reclutador intenta publicar en Indeed
  Then recibe el aviso "La integración con Indeed requiere reautenticación"
  And la oferta se publica correctamente en los demás canales seleccionados

# Error — publicar oferta sin aprobación
Scenario: Reclutador intenta publicar una requisición en estado "Borrador"
  Given la requisición está en estado "Borrador"
  When el reclutador accede a la opción de publicar
  Then el botón "Publicar" aparece deshabilitado
  And se muestra el mensaje "Solo pueden publicarse requisiciones aprobadas"
```

**Estimación de complejidad:** `L`

---

### US-05 · Envío de candidatura por el candidato

**Título:** Enviar candidatura y CV a través del portal del candidato

**Historia:**
> Como **Candidato**,
> quiero enviar mi candidatura y adjuntar mi CV desde el portal de la empresa,
> para postularme a un puesto de forma sencilla y recibir confirmación inmediata.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Requiere oferta publicada (US-04) |
| Negotiable | ✅ Campos del formulario negociables |
| Valuable | ✅ Es la acción central del candidato |
| Estimable | ✅ Formulario + upload de fichero |
| Small | ✅ Bien delimitada |
| Testable | ✅ Verificable por registro en BD y confirmación |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — envío exitoso
Scenario: Candidato completa y envía su candidatura
  Given el candidato accede a una oferta publicada en el portal
  When rellena nombre, email, teléfono y adjunta su CV en PDF
  And pulsa "Enviar candidatura"
  Then se crea un registro CANDIDATO (si no existe) y un registro CANDIDATURA
  And el CV se almacena en S3 y se crea el registro CURRICULUM
  And la candidatura queda en estado "Recibida" con etapa "Cribado"
  And el candidato recibe email de confirmación con número de referencia
  And se encola automáticamente un trabajo en el Motor de Cribado con IA

# Edge case — candidato ya había postulado al mismo puesto
Scenario: Candidato intenta enviar una segunda candidatura al mismo puesto
  Given el candidato con email "juan@email.com" ya tiene una candidatura activa para la requisición
  When intenta enviar una nueva candidatura para la misma requisición
  Then recibe el mensaje "Ya tienes una candidatura activa para este puesto"
  And no se genera un registro duplicado

# Edge case — CV en formato no soportado
Scenario: Candidato adjunta un archivo en formato .pages
  Given el formulario de candidatura acepta únicamente PDF y DOCX
  When el candidato adjunta un archivo .pages
  Then recibe el error "Formato de archivo no admitido. Por favor, sube un PDF o DOCX."
  And no se procesa el envío

# Error — oferta cerrada
Scenario: Candidato intenta postular a una oferta ya cerrada
  Given la publicación de la oferta tiene estado "Cerrada"
  When el candidato accede a la URL de la oferta
  Then ve el mensaje "Esta oferta ya no está disponible"
  And no puede enviar candidatura
```

**Estimación de complejidad:** `M`

---

### US-06 · Cribado automático de candidatos con IA

**Título:** Cribar y puntuar candidatos automáticamente con IA al recibir su candidatura

**Historia:**
> Como **Reclutador**,
> quiero que el sistema analice automáticamente cada CV recibido y genere una puntuación de adecuación con resumen en lenguaje natural,
> para reducir el tiempo de cribado manual y centrarme en los candidatos más relevantes.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-05 (candidatura con CV) |
| Negotiable | ✅ Las dimensiones de competencia y pesos son configurables |
| Valuable | ✅ Diferenciador estratégico clave del producto |
| Estimable | ✅ Motor documentado en PRD a nivel C4 |
| Small | ⚠️ Grande internamente; acotable en MVP a puntuación básica |
| Testable | ✅ Verificable por presencia de puntuación y resumen en CANDIDATURA |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — cribado exitoso con IA
Scenario: Motor de IA procesa una candidatura y genera puntuación
  Given existe una candidatura con CV en S3 y la requisición tiene descripción y competencias definidas
  When el Motor de Cribado con IA consume el trabajo de la cola
  Then extrae el texto del CV y construye el prompt con la descripción del puesto
  And llama a la API del LLM y recibe el análisis JSON
  And persiste puntuacion_adecuacion_ia (0-100) y resumen_ia en CANDIDATURA
  And publica el evento "cribado_completado" en el bus de eventos
  And el reclutador ve la puntuación actualizada en tiempo real en la UI

# Happy path — lista reducida ordenada
Scenario: Reclutador visualiza candidatos ordenados por puntuación de IA
  Given existen 50 candidaturas procesadas para una requisición
  When el reclutador accede a la vista de candidatos de esa requisición
  Then ve la lista ordenada de mayor a menor puntuacion_adecuacion_ia
  And cada candidato muestra la puntuación codificada por color (verde/amarillo/rojo)
  And puede expandir el resumen en lenguaje natural de cada uno

# Edge case — API del LLM no disponible
Scenario: La API del LLM devuelve error 503 tras los reintentos
  Given el LLM externo está caído
  When el Motor de Cribado agota los reintentos con backoff exponencial
  Then invoca el HeuristicScorer como fallback
  And persiste la puntuación heurística con ResponseSource = "HEURISTIC_FALLBACK"
  And en la UI el candidato muestra un indicador visual "Puntuación estimada (sin IA)"

# Edge case — CV con texto no extraíble (imagen escaneada)
Scenario: El CV es un PDF de imagen sin capa de texto
  Given el CV subido es un PDF escaneado sin OCR
  When el parser intenta extraer texto
  Then registra el fallo de extracción y marca la candidatura con flag "revision_manual_requerida"
  And notifica al reclutador: "No se pudo analizar el CV de [nombre]. Revisión manual necesaria."

# Error — respuesta del LLM con JSON malformado
Scenario: La respuesta del LLM no cumple el esquema JSON esperado
  Given el LLM devuelve una respuesta con estructura inválida
  When el ResponseParser intenta validar el JSON
  Then enruta el trabajo a la dead-letter queue
  And registra el error en el panel de monitorización de Prometheus
  And no escribe datos corruptos en CANDIDATURA
```

**Estimación de complejidad:** `L`

---

### US-07 · Espacio de trabajo colaborativo para evaluación de candidatos

**Título:** Evaluar y comentar candidatos en tiempo real con el equipo de selección

**Historia:**
> Como **Responsable de Selección**,
> quiero ver el perfil completo del candidato con las scorecards del equipo y poder dejar comentarios con @menciones,
> para tomar decisiones de contratación fundamentadas sin necesidad de reuniones adicionales.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-06 (candidatos procesados) y US-10 (scorecards) |
| Negotiable | ✅ La profundidad de la vista colaborativa es negociable |
| Valuable | ✅ Diferenciador central frente a ATS heredados |
| Estimable | ✅ WebSocket + comentarios estructurados |
| Small | ⚠️ Puede subdividirse en "vista perfil" y "comentarios" |
| Testable | ✅ Verificable por visibilidad de datos en tiempo real |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — ver perfil consolidado
Scenario: Responsable visualiza el perfil completo de un candidato con datos en tiempo real
  Given existen scorecards enviadas por 2 entrevistadores para la candidatura de "Carlos López"
  When el Responsable accede al perfil de "Carlos López"
  Then ve el resumen de IA, la puntuación de adecuación, el CV y las scorecards de cada entrevistador
  And ve la vista de consenso con puntuaciones agregadas por competencia
  And cualquier nueva scorecard enviada mientras está en la pantalla se actualiza sin recargar

# Happy path — @mención a un entrevistador
Scenario: Responsable menciona a un entrevistador en un comentario
  Given el Responsable está en el perfil de un candidato
  When escribe un comentario con "@Ana, ¿puedes ampliar tu valoración técnica?"
  Then el comentario se guarda con marca de tiempo y queda visible para todo el equipo
  And "Ana" recibe una notificación in-app con el texto del comentario y enlace al perfil

# Edge case — dos usuarios editan simultáneamente
Scenario: Reclutador y Responsable añaden comentarios al mismo tiempo
  Given ambos usuarios tienen abierto el mismo perfil de candidato
  When ambos envían un comentario en el mismo segundo
  Then ambos comentarios se persisten correctamente y se muestran ordenados por timestamp
  And ninguno sobreescribe al otro (no hay condición de carrera)

# Error — intento de modificar scorecard ya enviada
Scenario: Entrevistador intenta editar una scorecard ya enviada
  Given una scorecard fue enviada hace más de 30 minutos
  When el entrevistador intenta modificarla
  Then recibe el mensaje "Las scorecards enviadas no pueden modificarse. Contacta con el reclutador."
  And la scorecard original permanece inalterada en base de datos
```

**Estimación de complejidad:** `L`

---

### US-08 · Asignación de evaluaciones en línea a candidatos

**Título:** Asignar una prueba técnica o de competencias a un candidato durante el proceso

**Historia:**
> Como **Reclutador**,
> quiero asignar una evaluación técnica a un candidato en cualquier etapa del proceso,
> para obtener datos objetivos de competencia antes de la entrevista técnica.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-05 (candidatura activa) |
| Negotiable | ✅ Los proveedores externos son negociables; nativo en MVP básico |
| Valuable | ✅ Aporta datos objetivos al proceso |
| Estimable | ✅ CRUD de asignación + integración con proveedor |
| Small | ✅ Bien delimitada por las tres entidades del modelo |
| Testable | ✅ Verificable por estado de ASIGNACION_EVALUACION |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — asignación exitosa
Scenario: Reclutador asigna una prueba de programación a un candidato
  Given existe una EVALUACION activa de tipo "Reto de programación" con proveedor "HackerRank"
  And existe una candidatura activa en etapa "Entrevista técnica"
  When el reclutador selecciona la evaluación y pulsa "Asignar"
  Then se crea un registro ASIGNACION_EVALUACION con estado "Pendiente" y fecha de expiración a 5 días
  And el candidato recibe un email con la url_acceso_candidato para realizar la prueba
  And el reclutador ve el estado "Pendiente" en el perfil del candidato

# Happy path — resultado registrado automáticamente
Scenario: Candidato completa la evaluación y el resultado se importa
  Given el candidato ha completado la prueba en la plataforma externa
  When el proveedor envía el webhook de resultado a LTI
  Then se crea el registro RESULTADO_EVALUACION con puntuacion_bruta, puntuacion_normalizada y aprobado
  And el resultado se adjunta al perfil del candidato visible para el equipo
  And la puntuacion_adecuacion_ia de la CANDIDATURA se recalcula incorporando el resultado

# Edge case — evaluación expirada sin completar
Scenario: El candidato no completa la prueba antes de la fecha de expiración
  Given la ASIGNACION_EVALUACION tiene expira_en en el pasado y estado "Pendiente"
  When el sistema ejecuta el job de revisión de expiradas
  Then cambia el estado de la asignación a "Expirada"
  And notifica al reclutador: "La evaluación de [candidato] ha expirado sin completarse."

# Error — candidato intenta acceder a una evaluación ya completada
Scenario: Candidato accede a la URL de una evaluación ya entregada
  Given la ASIGNACION_EVALUACION tiene estado "Completada"
  When el candidato accede a la url_acceso_candidato
  Then recibe el mensaje "Esta evaluación ya ha sido completada."
  And no puede reiniciar la prueba
```

**Estimación de complejidad:** `L`

---

### US-09 · Programación automatizada de entrevistas

**Título:** Programar una entrevista leyendo la disponibilidad del panel en los calendarios

**Historia:**
> Como **Reclutador**,
> quiero que el sistema proponga franjas horarias disponibles al candidato leyendo los calendarios del panel de entrevistadores,
> para coordinar la entrevista sin intercambio de correos y reducir el tiempo de programación.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-03 (requisición con etapas) y US-05 (candidatura) |
| Negotiable | ✅ La integración de calendarios (Google vs Outlook) es negociable |
| Valuable | ✅ Elimina la principal queja de los reclutadores |
  | Estimable | ✅ Algoritmo de slots + integración de calendar APIs |
| Small | ⚠️ Puede subdividirse en "leer calendario" y "proponer y reservar" |
| Testable | ✅ Verificable por creación de ENTREVISTA y confirmaciones |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — programación exitosa
Scenario: Reclutador inicia la programación para un panel de 3 entrevistadores
  Given la requisición tiene configurada la etapa "Entrevista técnica" con 3 panelistas
  And los 3 panelistas tienen sus calendarios integrados con LTI
  When el reclutador pulsa "Programar entrevista" para la candidatura de "María García"
  Then LTI lee la disponibilidad de los 3 calendarios en los próximos 5 días laborables
  And calcula 3 franjas horarias en las que todos están disponibles
  And envía al candidato un email con las 3 opciones para que elija

# Happy path — candidato selecciona franja
Scenario: Candidato elige una franja horaria desde el email
  Given el candidato ha recibido el email con 3 opciones de franja
  When selecciona la franja del "martes 15 de abril a las 10:00"
  Then LTI crea el registro ENTREVISTA con estado "Programada" y la franja elegida
  And envía invitaciones de calendario a todos los panelistas y al candidato
  And programa recordatorios automáticos 24h y 1h antes de la entrevista
  And el reclutador ve la entrevista en estado "Confirmada" en el dashboard

# Edge case — no hay franjas disponibles en los próximos 5 días
Scenario: Los calendarios del panel no tienen coincidencia en la ventana de búsqueda
  Given los 3 panelistas tienen agendas completas durante los próximos 5 días
  When LTI calcula franjas disponibles
  Then amplía la ventana de búsqueda a 10 días laborables
  And si sigue sin haber coincidencia, notifica al reclutador: "No se encontraron franjas disponibles. Considera reducir el panel o ampliar el horizonte."

# Error — integración de calendario desconectada
Scenario: Uno de los panelistas no tiene el calendario integrado
  Given el panelista "Pedro Ruiz" no ha autorizado la integración de calendario
  When el reclutador intenta programar incluyendo a Pedro
  Then LTI advierte: "Pedro Ruiz no tiene el calendario conectado. Se procederá con disponibilidad manual."
  And permite al reclutador introducir manualmente las franjas para ese panelista
```

**Estimación de complejidad:** `L`

---

### US-10 · Realización y envío de scorecard estructurada por el entrevistador

**Título:** Completar y enviar la scorecard de la entrevista inmediatamente tras la sesión

**Historia:**
> Como **Entrevistador**,
> quiero completar una scorecard estructurada con puntuaciones por competencia y recomendación al terminar la entrevista,
> para que el equipo de selección tenga mi valoración disponible de forma inmediata y en un formato comparable.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-09 (entrevista programada) |
| Negotiable | ✅ El número de competencias y escala de puntuación son negociables |
| Valuable | ✅ Fuente de datos estructurados para la decisión final |
| Estimable | ✅ Formulario con campos definidos en el modelo de datos |
| Small | ✅ Bien acotada |
| Testable | ✅ Verificable por registro SCORECARD en BD y visibilidad en US-07 |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — scorecard enviada correctamente
Scenario: Entrevistador envía scorecard tras la entrevista
  Given existe una ENTREVISTA en estado "Realizada" con el entrevistador como panelista
  And el kit de entrevista tiene 4 competencias definidas para el puesto
  When el entrevistador puntúa cada competencia (escala 1–5), escribe comentario general
       y selecciona recomendación "Avanzar"
  And pulsa "Enviar Scorecard"
  Then se crea el registro SCORECARD con los ELEMENTO_SCORECARD correspondientes
  And la scorecard es inmediatamente visible en el perfil del candidato para el Responsable
  And el Responsable recibe notificación in-app: "Nueva scorecard disponible para [candidato]"

# Edge case — entrevistador intenta enviar scorecard sin puntuar todas las competencias
Scenario: Entrevistador deja una competencia sin puntuar
  Given el formulario de scorecard tiene 4 competencias obligatorias
  When el entrevistador deja la competencia "Resolución de problemas" sin puntuación
  And pulsa "Enviar Scorecard"
  Then recibe el error inline "Debes puntuar todas las competencias antes de enviar"
  And la scorecard no se persiste hasta que esté completa

# Edge case — entrevista sin kit de preguntas preconfigurado
Scenario: La requisición no tiene kit de entrevista definido
  Given la etapa "Entrevista cultural" no tiene competencias asignadas
  When el entrevistador accede al formulario de scorecard
  Then LTI muestra un formulario genérico con las competencias por defecto de la empresa
  And añade el aviso: "Esta etapa no tiene kit personalizado. Se están usando competencias genéricas."

# Error — entrevistador no asignado como panelista intenta enviar scorecard
Scenario: Un usuario que no es panelista de la entrevista accede al formulario
  Given el usuario "Luis Martínez" no está en el registro PANELISTA_ENTREVISTA de la entrevista
  When intenta acceder a la URL del formulario de scorecard
  Then recibe el error 403 "No tienes permiso para evaluar esta entrevista"
```

**Estimación de complejidad:** `M`

---

### US-11 · Seguimiento del estado de candidatura desde el portal del candidato

**Título:** Consultar el estado actualizado de la candidatura en tiempo real desde el portal

**Historia:**
> Como **Candidato**,
> quiero acceder al portal de LTI para ver en qué etapa está mi candidatura y recibir notificaciones de cambios de estado,
> para tener transparencia sobre mi proceso sin necesidad de contactar directamente con la empresa.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-05 (candidatura existente) |
| Negotiable | ✅ El nivel de detalle visible al candidato es negociable |
| Valuable | ✅ Mejora la experiencia del candidato y protege la marca empleadora |
| Estimable | ✅ Portal ligero con consulta de estado |
| Small | ✅ Bien acotada para MVP |
| Testable | ✅ Verificable por estado visible y notificaciones enviadas |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — candidato ve su estado actualizado
Scenario: Candidato accede al portal y ve su candidatura en etapa "Entrevista técnica"
  Given el candidato tiene una candidatura activa con etapa "Entrevista técnica"
  When accede al portal con su email y token de acceso recibido por email
  Then ve el progreso visual de su candidatura (Recibida → Cribado → Evaluación → Entrevista → Oferta)
  And ve que la etapa actual "Entrevista técnica" está destacada
  And ve la fecha de último cambio de estado

# Happy path — notificación de cambio de estado
Scenario: La candidatura avanza de etapa y el candidato recibe notificación
  Given el reclutador mueve la candidatura a la etapa "Oferta"
  Then el sistema envía automáticamente un email al candidato: "¡Buenas noticias! Tu candidatura ha avanzado."
  And el portal actualiza el estado en tiempo real si el candidato tiene la sesión abierta

# Edge case — candidato con múltiples candidaturas activas
Scenario: Candidato tiene candidaturas en dos empresas distintas
  Given el candidato con email "sofia@email.com" tiene candidaturas en "Empresa A" y "Empresa B"
  When accede al portal de "Empresa A"
  Then solo ve la candidatura correspondiente a "Empresa A"
  And los datos de "Empresa B" no son accesibles desde este portal (aislamiento de tenant)

# Error — candidatura rechazada
Scenario: El reclutador rechaza la candidatura
  Given el reclutador marca la candidatura con estado "Descartada"
  Then el candidato recibe email automático: "Hemos revisado tu candidatura y en esta ocasión no seguiremos adelante."
  And en el portal ve el estado "Proceso finalizado"
  And no se muestran detalles internos del motivo del rechazo
```

**Estimación de complejidad:** `M`

---

### US-12 · Gestión y envío de oferta digital

**Título:** Generar, enviar y registrar la aceptación de una oferta de empleo desde LTI

**Historia:**
> Como **Reclutador**,
> quiero generar una carta de oferta digital con los datos del puesto y enviarla al candidato para que la acepte dentro de la plataforma,
> para cerrar el proceso de contratación de forma trazable y sin herramientas externas.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de US-03 (requisición) y US-07 (decisión tomada) |
| Negotiable | ✅ La integración con firma electrónica es negociable para MVP básico |
| Valuable | ✅ Cierra el ciclo end-to-end del proceso |
| Estimable | ✅ Generación de documento + estados de oferta definidos |
| Small | ✅ Bien delimitada |
| Testable | ✅ Verificable por estados de OFERTA y notificaciones |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — oferta enviada y aceptada
Scenario: Reclutador genera y envía una oferta que el candidato acepta
  Given la candidatura de "Jorge Fernández" está en etapa "Oferta"
  When el reclutador rellena los datos de la oferta (título, compensación, fecha de inicio)
       y pulsa "Enviar oferta"
  Then se crea el registro OFERTA con estado "Enviada" y se almacena el documento en S3
  And el candidato recibe un email con enlace para ver y aceptar la oferta en el portal
  When el candidato pulsa "Aceptar oferta" en el portal
  Then el estado de OFERTA cambia a "Aceptada"
  And la CANDIDATURA cambia a estado "Contratado"
  And el reclutador y el Responsable reciben notificación de aceptación
  And LTI activa el traspaso de incorporación al HRIS configurado

# Edge case — candidato solicita más tiempo para decidir
Scenario: El candidato no responde en el plazo establecido
  Given la OFERTA tiene fecha de expiración a 5 días
  When el plazo expira sin respuesta del candidato
  Then el estado de OFERTA cambia a "Expirada"
  And el reclutador recibe notificación: "La oferta para [candidato] ha expirado sin respuesta."

# Edge case — candidato rechaza la oferta
Scenario: El candidato rechaza la oferta desde el portal
  Given el candidato tiene una OFERTA en estado "Enviada"
  When pulsa "Rechazar oferta"
  Then el estado cambia a "Rechazada"
  And el reclutador recibe notificación con la opción de reabrir el proceso o cancelar la requisición

# Error — oferta enviada cuando ya existe otra oferta activa para la misma candidatura
Scenario: Reclutador intenta crear una segunda oferta para la misma candidatura
  Given ya existe una OFERTA en estado "Enviada" para la candidatura
  When el reclutador intenta crear una nueva oferta
  Then recibe el error "Ya existe una oferta activa para este candidato."
  And se le ofrece la opción de retirar la anterior antes de enviar una nueva
```

**Estimación de complejidad:** `M`

---

### US-13 · Constructor de automatización de procesos (no-code)

**Título:** Configurar reglas de automatización sin código para el flujo de contratación

**Historia:**
> Como **Reclutador**,
> quiero crear reglas visuales de automatización (ej. "si el candidato supera la evaluación, moverlo automáticamente a entrevista"),
> para eliminar tareas administrativas repetitivas y asegurar que los SLA del proceso se cumplan.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Depende de que existan etapas (US-03) y evaluaciones (US-08) |
| Negotiable | ✅ El número de acciones disponibles en MVP es negociable |
| Valuable | ✅ Ahorra tiempo operativo y reduce errores humanos |
| Estimable | ✅ Motor de reglas documentado en PRD |
| Small | ⚠️ Candidata a subdivisión; recomendable empezar con 2-3 tipos de acciones en MVP |
| Testable | ✅ Verificable por ejecución de acciones ante eventos |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — creación de regla de automatización
Scenario: Reclutador crea una regla para avanzar candidatos que superan la evaluación
  Given la requisición tiene la etapa "Evaluación técnica" configurada
  When el reclutador crea la regla: CUANDO "Evaluación completada" Y "puntuacion_normalizada >= 70"
       ENTONCES "Mover candidato a etapa Entrevista técnica" Y "Notificar al Responsable"
  And pulsa "Activar regla"
  Then se crea el registro REGLA_AUTOMATIZACION con activo = true
  And la regla queda visible en el panel de automatizaciones de la requisición

# Happy path — ejecución de la regla
Scenario: La regla se ejecuta automáticamente al recibirse un resultado de evaluación
  Given la regla anterior está activa
  When el candidato "Elena Torres" completa la evaluación con puntuación 85/100
  Then el Motor de Automatización detecta el evento "evaluacion_completada"
  And evalúa la condición (85 >= 70) → verdadera
  And mueve automáticamente la candidatura de "Elena Torres" a la etapa "Entrevista técnica"
  And envía la notificación al Responsable de Selección

# Edge case — regla con condición nunca cumplida (regla huérfana)
Scenario: La regla hace referencia a una etapa que fue eliminada
  Given la regla apunta a la etapa "Entrevista cultural" que fue borrada
  When el Motor de Automatización intenta ejecutar la acción
  Then registra el error "Etapa destino no encontrada" en el log de la regla
  And marca la regla como "Error de configuración" visible para el reclutador
  And no realiza ninguna acción sobre la candidatura

# Error — bucle de automatización
Scenario: Dos reglas se activan mutuamente en bucle infinito
  Given existe la regla A: "si etapa = X → mover a Y" y la regla B: "si etapa = Y → mover a X"
  When se ejecuta la regla A
  Then el motor detecta el ciclo tras 3 iteraciones
  And detiene la ejecución, registra el evento "automation_loop_detected"
  And notifica al administrador del tenant para que revise la configuración
```

**Estimación de complejidad:** `L`

---

### US-14 · Dashboard de analítica e informes de contratación

**Título:** Consultar métricas clave del proceso de selección en un dashboard centralizado

**Historia:**
> Como **Responsable de Selección**,
> quiero acceder a un dashboard con métricas de tiempo hasta la contratación, tasas de conversión y fuente de candidatos,
> para tomar decisiones basadas en datos y optimizar el proceso de contratación.

**Evaluación INVEST:**
| Criterio | Valoración |
|---|---|
| Independent | ⚠️ Requiere datos generados por otras US |
| Negotiable | ✅ El set de métricas del MVP es negociable |
| Valuable | ✅ Permite mejora continua del proceso |
| Estimable | ✅ Queries sobre el modelo de datos existente |
| Small | ⚠️ Limitar el MVP a los 5 KPIs core del Lean Canvas |
| Testable | ✅ Verificable por exactitud de los datos mostrados |

**Acceptance Criteria (BDD — Gherkin):**

```gherkin
# Happy path — visualización del dashboard principal
Scenario: Responsable accede al dashboard de analítica
  Given existen datos de candidaturas y contrataciones en el último trimestre
  When el Responsable accede a la sección "Analítica"
  Then ve los siguientes KPIs actualizados:
       - Tiempo medio hasta la contratación (días)
       - Tasa de conversión por etapa del proceso (%)
       - Top 3 fuentes de candidatos por volumen
       - Tasa de aceptación de ofertas (%)
       - Número de posiciones abiertas/cerradas en el período
  And puede filtrar por requisición, departamento y rango de fechas

# Happy path — exportación de informe
Scenario: Responsable exporta los datos del dashboard a CSV
  Given el Responsable tiene aplicado un filtro por "departamento: Ingeniería"
  When pulsa "Exportar a CSV"
  Then descarga un archivo CSV con las métricas filtradas
  And el archivo incluye cabeceras descriptivas en castellano

# Edge case — dashboard sin datos suficientes
Scenario: La empresa acaba de activar LTI y solo tiene 2 candidaturas
  Given el tenant tiene menos de 5 candidaturas registradas
  When el Responsable accede al dashboard
  Then ve los widgets de KPIs con valor "–" y el mensaje "Datos insuficientes para mostrar esta métrica"
  And no se generan divisiones por cero ni errores visuales

# Error — usuario sin permiso de analítica intenta acceder
Scenario: Un Entrevistador intenta acceder a la sección de Analítica
  Given el rol "Entrevistador" no tiene el permiso "ver_analytics"
  When el entrevistador navega a "/analytics"
  Then recibe el error 403 con el mensaje "No tienes acceso a esta sección."
  And se registra el intento en REGISTRO_AUDITORIA
```

**Estimación de complejidad:** `M`

---

## Bloque 2 — Validación de consistencia entre User Stories relacionadas

### Mapa de dependencias

```
US-01 (Auth)
  └── US-02 (Tenant config)
        └── US-03 (Crear requisición)
              └── US-04 (Publicar oferta)
                    └── US-05 (Enviar candidatura)
                          ├── US-06 (Cribado IA)           → feed data → US-07 (Espacio colaborativo)
                          ├── US-08 (Asignación evaluación) → feed data → US-07
                          ├── US-09 (Programar entrevista)
                          │     └── US-10 (Scorecard)      → feed data → US-07
                          └── US-11 (Portal candidato)
              └── US-13 (Automatizaciones)
US-07 (Espacio colaborativo)
  └── US-12 (Oferta digital)
        └── US-11 (Portal candidato — aceptación)
US-14 (Analytics) — consume datos de todas las anteriores
```

---

### Análisis de consistencia por par de US relacionadas

#### US-03 ↔ US-04 · Requisición → Publicación
- **Consistencia:** ✅ La US-04 exige que la requisición esté en estado "Aprobada". El modelo de estados de US-03 define el ciclo Borrador → Pendiente aprobación → Aprobada, lo que habilita correctamente la transición.
- **Riesgo detectado:** El PRD no define explícitamente quién aprueba la requisición (¿solo el Responsable? ¿también un Director?). **Recomendación:** aclarar en US-03 los roles con permiso de aprobación antes del desarrollo.

#### US-04 ↔ US-05 · Publicación → Candidatura
- **Consistencia:** ✅ La candidatura referencia `publicacion_origen_id` (FK a PUBLICACION_EMPLEO), lo que asegura trazabilidad de fuente. La US-05 valida correctamente el estado "Cerrada" de la oferta.
- **Riesgo detectado:** Si se cierra la publicación (US-04) mientras hay candidaturas en curso (US-05), el AC de US-05 debe garantizar que las candidaturas ya recibidas no se vean afectadas. **Recomendación:** añadir AC de separación entre cierre de publicación y estado de candidaturas en curso.

#### US-05 ↔ US-06 · Candidatura → Cribado IA
- **Consistencia:** ✅ El encolado del trabajo de cribado es parte del AC de US-05 (happy path), lo que garantiza que el Motor de IA siempre recibe el trigger. El modelo CURRICULUM tiene los campos necesarios (`texto_extraido`, `competencias_extraidas`) que el Motor necesita escribir.
- **Riesgo detectado:** Si la IA falla y usa el fallback heurístico (AC edge case de US-06), la US-07 y US-14 deben saber interpretar el flag `ResponseSource = HEURISTIC_FALLBACK` para no mezclar métricas de calidad de cribado. **Recomendación:** añadir este flag como dato visible en el dashboard de US-14.

#### US-06 ↔ US-07 · Cribado IA → Espacio colaborativo
- **Consistencia:** ✅ El evento "cribado_completado" publicado en US-06 es el mecanismo que dispara la actualización en tiempo real de la UI en US-07. La arquitectura WebSocket documentada en el PRD da soporte a esto.
- **Riesgo detectado:** En US-07 se menciona "vista de consenso con puntuaciones agregadas por competencia" que combina datos de US-06 (IA) y US-10 (scorecards). Debe definirse la fórmula de agregación (¿media ponderada? ¿suma?). **Recomendación:** definir la lógica de consenso como AC adicional en US-07 antes del sprint de implementación.

#### US-08 ↔ US-06 · Evaluación → Cribado IA
- **Consistencia:** ✅ El AC happy path de US-08 incluye que al registrar el RESULTADO_EVALUACION se recalcula `puntuacion_adecuacion_ia`. Esto es coherente con el diseño del Motor de IA del PRD que integra resultados de evaluación en la puntuación.
- **Riesgo detectado:** El recálculo implica re-ejecutar (o actualizar) el trabajo de IA. El PRD no documenta si esto es un nuevo trabajo en la cola o una actualización directa. **Recomendación:** definir en US-08 si el recálculo es un nuevo trabajo de IA o una actualización ponderada del score existente.

#### US-09 ↔ US-10 · Programación → Scorecard
- **Consistencia:** ✅ La US-10 requiere una ENTREVISTA en estado "Realizada". La US-09 gestiona el ciclo Programada → Confirmada; el estado "Realizada" debe establecerse automáticamente tras pasar la fecha/hora de la entrevista o manualmente por el reclutador. Este estado no aparece explícitamente en ninguna US.
- **Riesgo detectado:** **Gap**: ninguna US documenta la transición "Confirmada → Realizada" de la ENTREVISTA. **Recomendación:** añadir AC en US-09 o crear una micro-US para la gestión del estado post-entrevista.

#### US-10 ↔ US-07 · Scorecard → Espacio colaborativo
- **Consistencia:** ✅ La US-10 genera SCORECARD + ELEMENTO_SCORECARD que son leídos en US-07. El AC de US-10 incluye la notificación al Responsable, coherente con el espacio colaborativo de US-07. Los modelos de datos son consistentes.
- **Sin riesgos adicionales detectados.**

#### US-07 ↔ US-12 · Espacio colaborativo → Oferta
- **Consistencia:** ✅ La decisión de avanzar a oferta (US-07) debe cambiar la etapa de la CANDIDATURA a "Oferta", que es la precondición del happy path de US-12. Esta transición de etapa está implícita pero no es un AC explícito de US-07.
- **Riesgo detectado:** **Gap**: no existe una US explícita para la acción "Tomar decisión: Avanzar a Oferta" que figure en el Caso de Uso 2 del PRD. Esta acción es la bisagra entre US-07 y US-12. **Recomendación:** añadir como AC adicional en US-07 la acción de decisión final con sus estados resultantes, o crear una micro-US de "Decisión final de contratación".

#### US-12 ↔ US-11 · Oferta → Portal del candidato
- **Consistencia:** ✅ El AC de US-12 menciona que el candidato acepta desde el portal, y US-11 cubre la experiencia del candidato en ese portal. Son complementarias y consistentes. El estado "Contratado" de CANDIDATURA tras aceptación en US-12 es coherente con los estados finales del modelo.
- **Sin riesgos adicionales detectados.**

#### US-13 ↔ US-06 / US-08 / US-09 · Automatizaciones → Otros servicios
- **Consistencia:** ✅ La US-13 depende de eventos publicados por US-06 ("cribado_completado"), US-08 ("evaluacion_completada") y US-10 ("scorecard_enviada"). El bus de eventos documentado en el PRD da soporte a esto.
- **Riesgo detectado:** El contrato de los eventos (nombre, payload, versión) no está documentado en el PRD ni en las US. **Recomendación:** definir un event catalog básico antes de implementar US-13 para evitar acoplamiento frágil entre servicios.

#### US-14 ↔ Todas las US
- **Consistencia:** ✅ Los KPIs definidos en US-14 (tiempo hasta la contratación, conversión por etapa, fuente de candidatos, aceptación de ofertas) están todos respaldados por los datos que generan las US anteriores (fechas en CANDIDATURA, OFERTA, PUBLICACION_EMPLEO, ETAPA_PROCESO).
- **Riesgo detectado:** La métrica "Precisión del cribado con IA" (KPI del Lean Canvas) no está incluida en US-14 ni se define cómo se mide. **Recomendación:** añadir esta métrica en el AC de US-14, definiendo que se mide como correlación entre `puntuacion_adecuacion_ia` y la decisión final de contratación.

---

### Resumen de consistencia y riesgos

| ID Riesgo | US afectadas | Descripción | Acción recomendada |
|---|---|---|---|
| R1 | US-03 | Roles con permiso de aprobación no definidos | Añadir AC de roles en US-03 |
| R2 | US-04 / US-05 | Impacto del cierre de publicación en candidaturas en curso | Añadir AC en US-05 |
| R3 | US-06 / US-14 | Flag HEURISTIC_FALLBACK no reflejado en analítica | Añadir AC en US-14 |
| R4 | US-06 / US-07 | Fórmula de consenso entre puntuación IA y scorecards no definida | Definir lógica en AC de US-07 |
| R5 | US-08 / US-06 | Mecanismo de recálculo de score IA tras resultado de evaluación ambiguo | Clarificar en AC de US-08 |
| R6 | US-09 / US-10 | Transición "Confirmada → Realizada" de entrevista no cubierta | Añadir AC en US-09 o crear micro-US |
| R7 | US-07 / US-12 | Acción "Avanzar a Oferta" no es AC explícito de ninguna US | Añadir AC en US-07 o crear micro-US |
| R8 | US-13 | Event catalog no definido | Crear documento de contrato de eventos antes del sprint de US-13 |
| R9 | US-14 | Métrica "Precisión del cribado con IA" no definida | Añadir AC de definición de la métrica en US-14 |

---

## Bloque 2 — Product Backlog priorizado para el MVP

### Criterios de priorización

Cada User Story se ha evaluado en una escala **1–5** (siendo 5 el valor más alto/crítico) sobre los siguientes ejes:

| Criterio | Descripción |
|---|---|
| **Valor de negocio** | Impacto directo en la propuesta de valor de LTI, ingresos o retención de clientes |
| **Urgencia** | Necesidad de que esté disponible en el MVP para que el producto sea funcional o vendible |
| **Dependencias** | Número de otras US que requieren esta como prerequisito (más dependientes = mayor urgencia de implementar primero) |
| **Coste de implementación** | Esfuerzo estimado inverso: complejidad alta → coste alto → puntuación baja |
| **Riesgo técnico** | Madurez tecnológica y obstáculos potenciales (riesgo alto → puntuación baja) |

La **puntuación de prioridad** se calcula como:

> `Prioridad = (Valor negocio × 2) + (Urgencia × 2) + Dependencias - Coste - Riesgo`

Este esquema de ponderación da más peso al valor y urgencia, mientras penaliza el coste y el riesgo técnico.

---

### Scoring detallado por US

| US | Título | Valor negocio (×2) | Urgencia (×2) | Dependencias bloqueadas | Coste impl. (inv.) | Riesgo técnico (inv.) | **Score total** |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|
| US-01 | Autenticación y acceso | 5 (10) | 5 (10) | 5 | −2 | −1 | **22** |
| US-02 | Configuración inicial del tenant | 5 (10) | 5 (10) | 5 | −2 | −1 | **22** |
| US-03 | Crear requisición de empleo | 5 (10) | 5 (10) | 4 | −2 | −1 | **21** |
| US-05 | Envío de candidatura | 5 (10) | 5 (10) | 4 | −2 | −1 | **21** |
| US-04 | Publicación multicanal | 4 (8) | 5 (10) | 3 | −3 | −2 | **16** |
| US-06 | Cribado IA | 5 (10) | 4 (8) | 3 | −4 | −3 | **14** |
| US-11 | Portal del candidato | 4 (8) | 4 (8) | 2 | −2 | −1 | **15** |
| US-10 | Scorecard del entrevistador | 4 (8) | 4 (8) | 3 | −2 | −1 | **16** |
| US-09 | Programación de entrevistas | 4 (8) | 4 (8) | 2 | −3 | −3 | **12** |
| US-07 | Espacio colaborativo | 5 (10) | 3 (6) | 2 | −3 | −2 | **13** |
| US-12 | Gestión de oferta digital | 5 (10) | 4 (8) | 1 | −2 | −1 | **16** |
| US-08 | Asignación de evaluaciones | 3 (6) | 3 (6) | 1 | −3 | −3 | **7** |
| US-14 | Dashboard de analítica | 4 (8) | 3 (6) | 0 | −2 | −1 | **11** |
| US-13 | Constructor de automatizaciones | 3 (6) | 2 (4) | 0 | −4 | −4 | **2** |

---

### Backlog priorizado — Orden recomendado para el MVP

---

#### Sprint 0 — Fundaciones (prerequisitos no negociables)

**Objetivo:** Habilitar que cualquier usuario pueda acceder al sistema y que la empresa esté configurada. Sin estas dos US, ninguna otra puede funcionar.

---

##### Posición 1 · US-01 — Autenticación y gestión de acceso
**Score: 22** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Sin autenticación no existe producto. Es el prerequisito absoluto de toda la plataforma.
- **Urgencia (5/5):** Debe estar en producción antes que cualquier otra funcionalidad.
- **Dependencias:** Bloquea el 100 % del resto de US (todas requieren usuario autenticado).
- **Coste (M):** Implementación estándar con librerías probadas (JWT, bcrypt). Coste razonable.
- **Riesgo técnico (bajo):** Tecnología madura. El único riesgo es el alcance del MVP (email/password vs SSO); recomendar dejar SSO para fase 2.
- **Madurez tecnológica:** Alta. Patrones bien establecidos en el stack Node.js/React del PRD.

---

##### Posición 2 · US-02 — Configuración inicial del tenant
**Score: 22** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Sin empresa configurada y usuarios con roles asignados, el producto no puede ser operado por ningún cliente.
- **Urgencia (5/5):** Es el onboarding mínimo viable para que un cliente pueda empezar a usar LTI.
- **Dependencias:** Bloquea US-03 a US-14. La arquitectura multiinquilino del PRD exige que `empresa_id` esté presente en todas las entidades.
- **Coste (M):** CRUD de empresa y usuarios. Bien delimitado.
- **Riesgo técnico (bajo):** La decisión de subdominio único por tenant debe validarse con el equipo de infraestructura para el routing en el API Gateway.
- **Madurez tecnológica:** Alta.

---

#### Sprint 1 — Flujo de captación (core del ciclo de contratación)

**Objetivo:** Habilitar el flujo completo de creación de oferta y recepción de candidaturas. Al final de este sprint, el producto puede recibir su primer candidato real.

---

##### Posición 3 · US-03 — Crear requisición de empleo
**Score: 21** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Es el punto de entrada de todo el ciclo de contratación. Sin requisición no hay oferta ni candidato.
- **Urgencia (5/5):** Dependencia directa de US-04 (publicación) y US-05 (candidaturas).
- **Dependencias:** Bloquea US-04, US-05, US-06, US-08, US-09, US-10, US-12, US-13.
- **Coste (M):** Formulario + máquina de estados (Borrador → Aprobada). Bien delimitado.
- **Riesgo (bajo):** El principal riesgo (R1) es la ambigüedad sobre quién puede aprobar. Resolver antes del sprint mediante spike de definición de permisos.
- **Obstáculo potencial:** El flujo de aprobación puede añadir complejidad; para el MVP se puede simplificar a auto-aprobación o aprobación por cualquier Responsable del tenant.

---

##### Posición 4 · US-05 — Envío de candidatura por el candidato
**Score: 21** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Es la acción central del candidato y el input que alimenta todo el motor de IA y el proceso posterior.
- **Urgencia (5/5):** Sin candidaturas no hay nada que procesar, evaluar ni contratar.
- **Dependencias:** Bloquea US-06 (cribado), US-08 (evaluaciones), US-09 (entrevistas), US-11 (portal), US-12 (oferta).
- **Coste (M):** Formulario + upload a S3 + creación de registros en BD. Implementación estándar.
- **Riesgo (bajo):** El riesgo R2 (impacto del cierre de publicación en candidaturas en curso) debe aclararse en los ACs antes de desarrollar. Gestión de ficheros en S3 es tecnología madura.
- **Madurez tecnológica:** Alta. El modelo de datos CANDIDATO/CANDIDATURA/CURRICULUM está completamente definido en el PRD.

---

##### Posición 5 · US-04 — Publicación multicanal de ofertas
**Score: 16** · Complejidad: L

**Justificación de prioridad:**
- **Valor de negocio (4/5):** Sin publicación no llegan candidatos. Es esencial para el MVP aunque puede arrancar con 1-2 canales.
- **Urgencia (5/5):** Sin oferta visible, US-05 no puede ocurrir desde canales externos.
- **Dependencias:** Habilita la atribución de fuente (`publicacion_origen_id`) en US-05.
- **Coste (L):** La integración con múltiples APIs externas (LinkedIn, Indeed) tiene coste elevado. **Estrategia MVP:** implementar primero solo el portal corporativo (internal) y añadir LinkedIn/Indeed en iteraciones posteriores.
- **Riesgo (medio):** Las APIs de portales externos tienen procesos de aprobación y cuotas. Riesgo de bloqueo externo. El portal corporativo es riesgo bajo y cubre el MVP.
- **Madurez tecnológica:** Media. APIs de LinkedIn Jobs/Indeed bien documentadas pero con requisitos de partnership.

---

#### Sprint 2 — Motor inteligente (diferenciadores estratégicos clave)

**Objetivo:** Activar el cribado automático con IA y el portal del candidato. Al final de este sprint, LTI ya demuestra su diferenciador principal frente a ATS heredados.

---

##### Posición 6 · US-06 — Cribado automático de candidatos con IA
**Score: 14** · Complejidad: L

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Es el diferenciador estratégico #1 del producto según el PRD. Sin él, LTI es un ATS ordinario.
- **Urgencia (4/5):** Puede arrancar el MVP sin él (candidaturas se reciben igualmente), pero no habría propuesta de valor diferencial.
- **Dependencias:** Alimenta US-07 (espacio colaborativo) y US-14 (analítica).
- **Coste (L):** Es el componente más complejo internamente (Motor de IA con C4 Level 3-4 documentado). Pipeline asíncrono, integración con LLM externo, fallback heurístico.
- **Riesgo (alto):** Tres riesgos activos: (1) calidad de los prompts requiere iteración, (2) coste de API de LLM puede dispararse con volumen, (3) CVs no procesables (R3, R4). Mitigar con límites de gasto y cuota por tenant en MVP.
- **Madurez tecnológica:** Media-alta. LLMs (OpenAI/Anthropic) son APIs estables, pero la ingeniería de prompts y la validación de outputs requieren trabajo específico.
- **Obstáculo potencial:** Latencia de procesamiento. El diseño asíncrono del PRD mitiga el impacto en UX, pero el SLA de procesamiento debe definirse (recomendado: < 2 min por CV).

---

##### Posición 7 · US-11 — Portal del candidato (seguimiento de estado)
**Score: 15** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (4/5):** Protege la marca empleadora del cliente y reduce el churn de candidatos. Es un argumento de venta clave frente a competidores.
- **Urgencia (4/5):** Sin portal, los candidatos están "a ciegas" y la experiencia de LTI es incompleta para el comprador.
- **Dependencias:** Recibe datos de US-05 (candidatura) y US-12 (oferta). Puede implementarse en paralelo con US-06.
- **Coste (M):** Aplicación Next.js ligera con consulta de estado. La parte de notificaciones por email reutiliza el Servicio de Notificaciones del PRD.
- **Riesgo (bajo):** El aislamiento de tenant (R2-relacionado) debe verificarse para que el candidato solo vea su candidatura del tenant correcto.
- **Madurez tecnológica:** Alta. Next.js, lectura de BD, envío de emails son tecnologías maduras.

---

#### Sprint 3 — Proceso de evaluación y entrevistas

**Objetivo:** Completar el flujo de evaluación estructurada: entrevistas programadas, scorecards y decisión final. Al final de este sprint, LTI cubre el ciclo completo hasta la decisión de contratación.

---

##### Posición 8 · US-10 — Scorecard estructurada del entrevistador
**Score: 16** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (4/5):** Aporta datos estructurados para la decisión y elimina el feedback disperso en emails/Slack.
- **Urgencia (4/5):** Prerequisito directo de US-07 (espacio colaborativo). Sin scorecards, la vista de consenso no tiene datos.
- **Dependencias:** Bloquea US-07 en su capacidad más valiosa (vista de consenso).
- **Coste (M):** Formulario con competencias + persistencia en SCORECARD/ELEMENTO_SCORECARD. El modelo de datos está completamente definido.
- **Riesgo (bajo-medio):** El riesgo R6 (transición "Confirmada → Realizada" de entrevista) debe resolverse antes de este sprint.
- **Madurez tecnológica:** Alta. No hay dependencias técnicas externas.

---

##### Posición 9 · US-09 — Programación automatizada de entrevistas
**Score: 12** · Complejidad: L

**Justificación de prioridad:**
- **Valor de negocio (4/5):** Elimina el mayor punto de fricción operativa de los reclutadores. Argumento de venta potente.
- **Urgencia (4/5):** Habilita US-10 de forma fluida, aunque las entrevistas pueden coordinarse manualmente en MVP inicial.
- **Dependencias:** Habilita US-10. Sin ella, la entrevista puede crearse manualmente como fallback.
- **Coste (L):** La integración con Google Calendar y Outlook requiere OAuth2 y gestión de tokens de refresco. Algoritmo de cálculo de franjas con múltiples participantes añade complejidad.
- **Riesgo (alto):** Tres riesgos activos: (1) OAuth con calendarios corporativos puede requerir aprobación del administrador de Google Workspace/Azure AD del cliente, (2) gestión de zonas horarias, (3) panelistas sin calendario integrado (R6). **Estrategia MVP:** permitir la creación manual de entrevistas como fallback si la integración de calendario no está conectada.
- **Madurez tecnológica:** Media. APIs de Google Calendar y Outlook Calendar son estables pero la integración OAuth2 empresarial tiene fricción de configuración.

---

##### Posición 10 · US-07 — Espacio de trabajo colaborativo
**Score: 13** · Complejidad: L

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Es el segundo diferenciador estratégico del PRD. La colaboración en tiempo real es lo que distingue LTI de los ATS heredados.
- **Urgencia (3/5):** Requiere US-06 y US-10 con datos reales para tener valor completo.
- **Dependencias:** Consume datos de US-06 (score IA), US-08 (evaluaciones) y US-10 (scorecards).
- **Coste (L):** La sincronización en tiempo real via WebSocket, los comentarios con @menciones y la vista de consenso son componentes no triviales.
- **Riesgo (medio):** El riesgo R4 (fórmula de consenso) y R7 (acción "Avanzar a Oferta" no definida) deben resolverse antes del sprint.
- **Madurez tecnológica:** Media. WebSockets son tecnología madura pero la sincronización concurrente (condiciones de carrera en comentarios simultáneos) requiere diseño cuidadoso.

---

#### Sprint 4 — Cierre del ciclo y datos

**Objetivo:** Completar el ciclo end-to-end con la gestión de ofertas, y añadir visibilidad analítica para los decision makers.

---

##### Posición 11 · US-12 — Gestión y envío de oferta digital
**Score: 16** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (5/5):** Cierra el ciclo de contratación dentro de la plataforma. Sin oferta digital, el proceso acaba fuera de LTI (email, Word, PDF manual), rompiendo la promesa end-to-end.
- **Urgencia (4/5):** Puede demorarse hasta Sprint 4 porque los sprints anteriores deben generar candidatos listos para recibir ofertas.
- **Dependencias:** Depende de US-07 (decisión tomada) y alimenta US-11 (aceptación en portal).
- **Coste (M):** Generación de documento + estados de OFERTA + integración con DocuSign (opcional en MVP; puede sustituirse por descarga PDF en MVP v1).
- **Riesgo (bajo-medio):** El riesgo R7 (bisagra US-07 → US-12) debe resolverse. La integración con firma electrónica (DocuSign) puede dejarse para MVP+1 reduciendo el coste y riesgo.
- **Madurez tecnológica:** Alta (sin DocuSign). Media (con DocuSign).

---

##### Posición 12 · US-14 — Dashboard de analítica e informes
**Score: 11** · Complejidad: M

**Justificación de prioridad:**
- **Valor de negocio (4/5):** Las métricas son un argumento de venta crítico para compradores de nivel directivo (CHRO, VP de RR.HH.) y una herramienta de mejora continua.
- **Urgencia (3/5):** Solo tiene valor cuando hay datos suficientes generados por los sprints anteriores. No tiene sentido en un tenant recién activado.
- **Dependencias:** Consume datos de todas las US anteriores. No bloquea a ninguna.
- **Coste (M):** Queries sobre el modelo de datos existente + visualización. Se puede arrancar con 5 KPIs predefinidos sin constructor personalizado.
- **Riesgo (bajo):** El riesgo R3 (flag HEURISTIC_FALLBACK en analítica) y R9 (métrica de precisión de IA) deben incluirse en los ACs antes del sprint.
- **Madurez tecnológica:** Alta. Queries SQL + librería de charting (recharts, Chart.js).

---

#### Sprint 5 — Funcionalidades avanzadas (post-MVP core)

**Objetivo:** Añadir las funcionalidades de mayor complejidad técnica que amplifican el valor del producto pero no son bloqueantes para el MVP.

---

##### Posición 13 · US-08 — Asignación de evaluaciones en línea
**Score: 7** · Complejidad: L

**Justificación de prioridad:**
- **Valor de negocio (3/5):** Aporta datos objetivos de competencia, pero los procesos pueden funcionar sin evaluaciones formales en el MVP inicial.
- **Urgencia (3/5):** No bloquea ninguna otra US del MVP core. Es una mejora del flujo de evaluación, no un prerequisito.
- **Dependencias:** Ninguna US del MVP core depende de ella.
- **Coste (L):** La integración con proveedores externos (HackerRank, Codility) tiene complejidad de integración + gestión de webhooks de resultado. Para MVP podría limitarse a evaluaciones nativas básicas.
- **Riesgo (alto):** Tres riesgos activos: (1) cada proveedor externo tiene su propia API y contrato de webhooks, (2) el recálculo del score IA (R5) requiere definición previa, (3) gestión de expiración de asignaciones.
- **Madurez tecnológica:** Media. Los proveedores (HackerRank, TestGorilla) tienen APIs documentadas pero la integración bidireccional (asignación + webhook de resultado) requiere trabajo de integración específico.

---

##### Posición 14 · US-13 — Constructor de automatizaciones no-code
**Score: 2** · Complejidad: L

**Justificación de prioridad:**
- **Valor de negocio (3/5):** Elimina trabajo administrativo repetitivo y es un argumento de venta para equipos de RR.HH. maduros. Sin embargo, no es bloqueante para operar LTI.
- **Urgencia (2/5):** Los procesos pueden operarse manualmente en el MVP. La automatización es una mejora de eficiencia, no un prerequisito funcional.
- **Dependencias:** Ninguna US del MVP core la requiere. Es ella quien consume los eventos de otras US.
- **Coste (L):** Motor de evaluación de reglas + UI de construcción drag-and-drop + gestión de bucles y errores. Es la US de mayor coste de implementación del backlog.
- **Riesgo (muy alto):** El riesgo R8 (event catalog no definido) es un bloqueante técnico que debe resolverse antes de comenzar. El riesgo de bucles de automatización (AC de US-13) requiere diseño de salvaguardas cuidadoso.
- **Madurez tecnológica:** Media-baja para la capa no-code. El motor de reglas es un problema bien conocido, pero el builder visual (drag-and-drop) requiere inversión de UX significativa.
- **Recomendación:** Diferir al MVP+1. En el MVP puede sustituirse por reglas de automatización predefinidas configurables (templates fijos) sin el builder visual, reduciendo el coste a M.

---

### Resumen del roadmap por sprint

| Sprint | US incluidas | Objetivo |
|---|---|---|
| **Sprint 0** | US-01, US-02 | Fundaciones: auth y configuración del tenant |
| **Sprint 1** | US-03, US-05, US-04 | Flujo de captación: requisición → oferta → candidatura |
| **Sprint 2** | US-06, US-11 | Motor IA + portal del candidato (diferenciadores visibles) |
| **Sprint 3** | US-10, US-09, US-07 | Evaluación estructurada: scorecards + entrevistas + colaboración |
| **Sprint 4** | US-12, US-14 | Cierre del ciclo: oferta digital + analítica |
| **Sprint 5** | US-08, US-13 | Funcionalidades avanzadas: evaluaciones externas + automatizaciones |

---

## Bloque 3 — Matriz resumen del Backlog MVP

| # | Item | Impacto usuario / negocio | Urgencia | Complejidad | Riesgos / Dependencias |
|:---:|---|---|:---:|:---:|---|
| 1 | **US-01** · Autenticación y acceso | 🔴 Crítico — sin auth no hay producto | Máxima | M | Bajo · Prerequisito de todas las US · Decidir SSO ahora o en fase 2 |
| 2 | **US-02** · Config. inicial del tenant | 🔴 Crítico — sin tenant no hay operación | Máxima | M | Bajo · Bloquea US-03 a US-14 · Validar routing multiinquilino con infra |
| 3 | **US-03** · Crear requisición | 🔴 Alto — punto de entrada del ciclo | Muy alta | M | Bajo-medio · Bloquea 8 US · R1: definir roles de aprobación antes del sprint |
| 4 | **US-05** · Envío de candidatura | 🔴 Alto — input de todo el motor de datos | Muy alta | M | Bajo · Bloquea US-06/08/09/11/12 · R2: aclarar impacto del cierre de publicación |
| 5 | **US-04** · Publicación multicanal | 🟠 Alto — sin oferta no hay candidatos | Alta | L | Medio · Depende de US-03 · APIs de portales con procesos de partnership · Empezar por portal corporativo |
| 6 | **US-11** · Portal del candidato | 🟠 Alto — experiencia de marca empleadora | Alta | M | Bajo · Depende de US-05 · Verificar aislamiento de tenant |
| 7 | **US-06** · Cribado IA | 🔴 Muy alto — diferenciador estratégico #1 | Alta | L | Alto · Depende de US-05 · R3/R4: coste LLM, calidad de prompts, CVs no procesables · SLA < 2 min por CV |
| 8 | **US-10** · Scorecard del entrevistador | 🟠 Alto — datos estructurados para decisión | Alta | M | Bajo · Depende de US-09 · R6: resolver transición "Confirmada → Realizada" antes del sprint |
| 9 | **US-09** · Programación de entrevistas | 🟠 Alto — elimina fricción de coordinación | Alta | L | Alto · Depende de US-03/05 · OAuth2 corporativo con Google/Azure AD · Proveer fallback manual obligatorio |
| 10 | **US-07** · Espacio colaborativo | 🔴 Muy alto — diferenciador estratégico #2 | Media | L | Medio · Depende de US-06/10 · R4: definir fórmula de consenso · R7: añadir AC "Avanzar a Oferta" |
| 11 | **US-12** · Gestión de oferta digital | 🟠 Alto — cierra el ciclo end-to-end | Media-alta | M | Bajo-medio · Depende de US-07 · R7: bisagra US-07→US-12 · DocuSign diferible al MVP+1 |
| 12 | **US-14** · Dashboard de analítica | 🟡 Medio-alto — decisiones basadas en datos | Media | M | Bajo · Consume datos de todas las US · R3/R9: incluir métricas de IA y precisión de cribado |
| 13 | **US-08** · Asignación de evaluaciones | 🟡 Medio — datos objetivos de competencia | Baja | L | Alto · No bloquea otras US · R5: definir recálculo score IA · Integración webhooks por proveedor |
| 14 | **US-13** · Automatizaciones no-code | 🟡 Medio — eficiencia operativa avanzada | Muy baja | L | Muy alto · No bloquea otras US · R8: event catalog como prerequisito · Riesgo de bucles · Diferir a MVP+1 |

### Leyenda de iconos

| Icono | Significado |
|---|---|
| 🔴 | Impacto crítico o muy alto |
| 🟠 | Impacto alto |
| 🟡 | Impacto medio |

---

---

## Bloque 4 — Tickets de desarrollo por Sprint y User Story

> **Convención de identificadores:** `TKT-[US]-[NN]`  
> **Escala de puntos de historia (Story Points):** S = 1–2 SP · M = 3–5 SP · L = 8–13 SP  
> **Tipos de ticket:** `[BE]` Backend · `[FE]` Frontend · `[INFRA]` Infraestructura · `[QA]` Testing · `[SPIKE]` Investigación técnica

---

### Sprint 0 — Fundaciones

#### US-01 · Autenticación y gestión de acceso

---

##### TKT-01-01 · [SPIKE] Definir estrategia de autenticación para el MVP
- **US:** US-01 — Autenticación y gestión de acceso
- **Tipo:** SPIKE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Investigar y documentar la estrategia de autenticación para el MVP: email + contraseña con JWT. Definir la librería a utilizar (p.ej. Passport.js), la duración del token, la política de refresco y la estrategia de almacenamiento de contraseñas (bcrypt). Dejar documentado el plan de SSO para fase 2 sin implementarlo. Documentar la decisión en un ADR (Architecture Decision Record).

**Criterios de aceptación:**
- ADR creado y revisado por el equipo técnico.
- Decisión: email/password + JWT HS256 con expiración de 8 horas.
- Estrategia de refresh token documentada (opcional MVP, requerida fase 2).
- SSO explícitamente marcado como out-of-scope para el MVP.

**Dependencias previas:** Ninguna.  
**Bloquea:** TKT-01-02, TKT-01-03.

---

##### TKT-01-02 · [BE] Endpoint de login con JWT
- **US:** US-01 — Autenticación y gestión de acceso
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar el endpoint `POST /auth/login` que reciba `email` y `password`, valide las credenciales contra la tabla USUARIO, y devuelva un JWT firmado en caso de éxito.

**Acceptance Criteria:**

```gherkin
# Happy path
Scenario: Credenciales válidas → JWT emitido
  Given el usuario existe en USUARIO con activo = true
  When POST /auth/login con email y password correctos
  Then responde 200 con { token, expires_in, rol }
  And registra evento en REGISTRO_AUDITORIA (accion = "login_ok")

# Error — contraseña incorrecta
Scenario: Contraseña errónea acumula intentos fallidos
  Given el usuario existe en USUARIO
  When POST /auth/login con password incorrecto 3 veces consecutivas
  Then en el tercer intento responde 401 y bloquea la cuenta 15 minutos
  And registra evento "login_failed" en REGISTRO_AUDITORIA

# Edge case — usuario inactivo
Scenario: Usuario con activo = false intenta login
  Given el usuario tiene activo = false
  When POST /auth/login
  Then responde 403 con mensaje "Cuenta desactivada"
  And no genera token
```

**Notas técnicas:**
- Usar `bcrypt.compare` para verificar contraseña.
- El token debe incluir: `user_id`, `empresa_id`, `rol`, `exp`.
- Campo `intentos_fallidos` + `bloqueado_hasta` en tabla USUARIO.

**Dependencias previas:** TKT-01-01.  
**Bloquea:** TKT-01-04, TKT-01-05.

---

##### TKT-01-03 · [BE] Middleware de autenticación y autorización por rol
- **US:** US-01 — Autenticación y gestión de acceso
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar el middleware que valide el JWT en cada petición protegida, extraiga el `rol` del token y rechace peticiones sin permisos suficientes. Definir la matriz de permisos por rol (Admin, Reclutador, Responsable, Entrevistador, Candidato).

**Acceptance Criteria:**

```gherkin
# Happy path
Scenario: Token válido con rol correcto
  Given petición con Authorization: Bearer <token_válido>
  And el endpoint requiere rol "Reclutador"
  And el token contiene rol = "Reclutador"
  When el middleware valida la petición
  Then permite el paso al controlador

# Edge case — token expirado
Scenario: Token expirado es rechazado
  Given petición con token JWT con exp en el pasado
  When el middleware valida el token
  Then responde 401 con { error: "Sesión expirada" }
  And redirige al cliente a re-autenticarse

# Error — rol insuficiente
Scenario: Entrevistador intenta acceder a endpoint de Admin
  Given token con rol = "Entrevistador"
  And el endpoint requiere rol = "Admin"
  When el middleware evalúa permisos
  Then responde 403 con { error: "Acceso denegado" }
  And registra el intento en REGISTRO_AUDITORIA
```

**Notas técnicas:**
- Matriz de permisos como constante en `config/roles.ts`.
- El middleware debe ser reutilizable por todos los servicios del monolito.

**Dependencias previas:** TKT-01-01.  
**Bloquea:** Todas las rutas protegidas de las US posteriores.

---

##### TKT-01-04 · [BE] Endpoint de recuperación de contraseña
- **US:** US-01 — Autenticación y gestión de acceso
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar `POST /auth/forgot-password` (genera token de reset y envía email) y `POST /auth/reset-password` (valida token y actualiza contraseña). El token de reset debe tener expiración de 1 hora y uso único.

**Acceptance Criteria:**

```gherkin
# Happy path
Scenario: Usuario solicita reset y completa el proceso
  Given usuario existe con email "ana@empresa.com"
  When POST /auth/forgot-password con { email }
  Then se genera token_reset de un solo uso con exp 1h
  And se envía email con enlace que contiene el token
  When usuario POST /auth/reset-password con token válido y nueva contraseña
  Then contraseña actualizada, token invalidado
  And responde 200

# Edge case — token ya usado
Scenario: Token de reset usado por segunda vez
  Given el token ya fue utilizado una vez
  When POST /auth/reset-password con el mismo token
  Then responde 400 con "Enlace expirado o ya utilizado"
```

**Dependencias previas:** TKT-01-02.

---

##### TKT-01-05 · [FE] Pantalla de login y recuperación de contraseña
- **US:** US-01 — Autenticación y gestión de acceso
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar en la React SPA las pantallas de login (email + password) y recuperación de contraseña. Gestionar el almacenamiento del JWT en memoria (no localStorage por seguridad) y el flujo de redirección post-login al dashboard según rol.

**Acceptance Criteria:**
- Formulario de login con validación de campos vacíos y formato de email.
- Mensaje de error claro ante credenciales incorrectas (sin revelar si el email existe o no).
- Contador de bloqueo visible cuando la cuenta está bloqueada temporalmente.
- Redirección post-login al dashboard del rol correspondiente.
- Pantalla de "Recuperar contraseña" con formulario de email y confirmación de envío.
- Token JWT almacenado en memoria (React context/state), no en localStorage.

**Dependencias previas:** TKT-01-02, TKT-01-04.

---

##### TKT-01-06 · [QA] Tests de integración de autenticación
- **US:** US-01 — Autenticación y gestión de acceso
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Escribir tests de integración que cubran los escenarios principales del flujo de auth: login exitoso, login fallido con bloqueo, token expirado, usuario inactivo y reset de contraseña.

**Criterios de cobertura mínima:**
- Login exitoso → token válido con claims correctos.
- 3 fallos consecutivos → bloqueo de cuenta.
- Token expirado → 401 en endpoint protegido.
- Usuario `activo = false` → 403.
- Reset de contraseña happy path + token expirado.

**Dependencias previas:** TKT-01-02, TKT-01-03, TKT-01-04.

---

#### US-02 · Configuración inicial del tenant

---

##### TKT-02-01 · [INFRA] Configurar routing multiinquilino por subdominio
- **US:** US-02 — Configuración inicial del tenant
- **Tipo:** INFRA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Configurar el API Gateway para resolver el `empresa_id` a partir del subdominio de la petición (p.ej. `acme.lti.app → empresa_id = uuid`). Implementar la capa de resolución de tenant en el middleware para que todas las peticiones tengan `req.tenant` disponible.

**Acceptance Criteria:**
- El middleware extrae el subdominio de `Host` header y lo resuelve contra la tabla EMPRESA.
- Si el subdominio no existe, responde 404 con mensaje claro.
- El `empresa_id` del tenant resuelto se inyecta en todas las queries de BD posteriores.
- Tests unitarios del resolver de tenant.

**Dependencias previas:** TKT-01-03.  
**Bloquea:** TKT-02-02 y todas las US siguientes que accedan a datos de empresa.

---

##### TKT-02-02 · [BE] CRUD de empresa y configuración inicial
- **US:** US-02 — Configuración inicial del tenant
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar los endpoints para crear y actualizar la entidad EMPRESA: `POST /admin/empresa` (alta inicial) y `PUT /admin/empresa` (actualización de datos). Incluir validación de unicidad de subdominio y restricción de dominio corporativo de email.

**Acceptance Criteria:**

```gherkin
# Happy path
Scenario: Admin crea la empresa con subdominio disponible
  Given POST /admin/empresa con { nombre, sector, nivel_tamaño, subdominio }
  When el subdominio no existe en BD
  Then se crea EMPRESA en BD y responde 201 con el id del tenant

# Error — subdominio duplicado
Scenario: Subdominio ya en uso
  Given el subdominio "acme" ya existe en EMPRESA
  When POST /admin/empresa con subdominio "acme"
  Then responde 409 con "Subdominio no disponible"

# Edge case — subdominio con caracteres inválidos
Scenario: Subdominio con espacios o mayúsculas
  Given POST /admin/empresa con subdominio "Mi Empresa"
  Then responde 400 con "El subdominio solo puede contener letras minúsculas, números y guiones"
```

**Dependencias previas:** TKT-02-01.

---

##### TKT-02-03 · [BE] CRUD de usuarios e invitación por email
- **US:** US-02 — Configuración inicial del tenant
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar los endpoints de gestión de usuarios: `POST /admin/usuarios/invitar`, `PUT /admin/usuarios/:id` (editar rol/estado), `DELETE /admin/usuarios/:id` (desactivar). El flujo de invitación genera un token de activación de un solo uso, enviado por email, con expiración de 48 horas.

**Acceptance Criteria:**

```gherkin
# Happy path
Scenario: Admin invita a nuevo usuario
  Given POST /admin/usuarios/invitar con { email, rol }
  When el email no existe previamente en el tenant
  Then se crea USUARIO con activo = false y token de activación
  And se envía email al usuario con enlace de activación

# Happy path — activación de cuenta
Scenario: Usuario activa su cuenta
  Given usuario con token de activación válido
  When POST /auth/activate con { token, password }
  Then activo = true, contraseña establecida, token invalidado

# Edge case — invitación a dominio no corporativo (si restricción configurada)
Scenario: Email fuera del dominio permitido
  Given empresa tiene dominio_permitido = "@empresa.com"
  When Admin invita a "user@gmail.com"
  Then responde 400 con "Email fuera del dominio corporativo permitido"

# Error — rol inválido
Scenario: Admin asigna un rol que no existe
  Given POST /admin/usuarios/invitar con { rol: "SuperAdmin" }
  Then responde 400 con "Rol no válido"
```

**Dependencias previas:** TKT-02-02, TKT-01-02.

---

##### TKT-02-04 · [FE] Panel de administración: empresa y usuarios
- **US:** US-02 — Configuración inicial del tenant
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la React SPA el área de administración (`/admin`) accesible solo para el rol Admin. Incluir: formulario de configuración de empresa, listado de usuarios con su rol y estado, formulario de invitación de usuario y opción de desactivar/reactivar cuentas.

**Acceptance Criteria:**
- El menú `/admin` solo es visible y accesible para usuarios con rol Admin (validado también en backend).
- Formulario de empresa muestra los datos actuales y permite editarlos.
- Listado de usuarios muestra nombre, email, rol y estado (activo/inactivo).
- El botón "Invitar usuario" abre un formulario con campo email + selector de rol.
- Confirmación visual (toast) tras cada acción exitosa.
- Mensaje de error claro cuando el email ya existe o el dominio no está permitido.

**Dependencias previas:** TKT-02-02, TKT-02-03, TKT-01-05.

---

##### TKT-02-05 · [QA] Tests de gestión de tenant y usuarios
- **US:** US-02 — Configuración inicial del tenant
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: creación de empresa con subdominio válido/duplicado, invitación y activación de usuario, desactivación de cuenta, y verificación de aislamiento de tenant (usuario de empresa A no puede ver datos de empresa B).

**Cobertura mínima:**
- Creación de empresa → subdominio único y duplicado.
- Invitación + activación completa.
- Aislamiento: query con `empresa_id` de otro tenant devuelve vacío.

**Dependencias previas:** TKT-02-02, TKT-02-03.

---

### Sprint 1 — Flujo de captación

#### US-03 · Crear requisición de empleo

---

##### TKT-03-01 · [SPIKE] Definir roles con permiso de aprobación de requisiciones (R1)
- **US:** US-03 — Crear requisición de empleo
- **Tipo:** SPIKE · **Complejidad:** S · **Story Points:** 1

**Descripción:**  
Resolver el riesgo R1 identificado en el análisis de consistencia. Documentar qué roles pueden crear requisiciones (Reclutador, Admin) y qué roles pueden aprobarlas (Responsable de Selección, Admin). Definir si en el MVP el flujo de aprobación es simplificado (auto-aprobación disponible para Admin) o siempre requiere segundo actor.

**Criterios de aceptación:**
- Decisión documentada en la matriz de permisos de `config/roles.ts`.
- Para el MVP: creación = Reclutador + Admin; aprobación = Responsable + Admin.
- Auto-aprobación disponible solo para Admin (flag configurable por tenant).

**Dependencias previas:** TKT-01-03.  
**Bloquea:** TKT-03-02.

---

##### TKT-03-02 · [BE] CRUD de requisición de empleo y máquina de estados
- **US:** US-03 — Crear requisición de empleo
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar los endpoints de gestión de requisiciones: `POST /requisiciones` (crear), `GET /requisiciones` (listar por tenant), `GET /requisiciones/:id` (detalle), `PUT /requisiciones/:id` (editar en borrador), `PATCH /requisiciones/:id/estado` (transición de estado). Estados: `Borrador → Pendiente_Aprobacion → Aprobada → Cerrada`.

**Acceptance Criteria:**

```gherkin
# Happy path — crear requisición
Scenario: Reclutador crea requisición con todos los campos
  Given Reclutador autenticado con empresa_id válido
  When POST /requisiciones con { titulo, departamento, ubicacion, tipo_contrato,
       salario_min, salario_max, descripcion, fecha_cierre_objetivo }
  Then se crea REQUISICION_EMPLEO con estado "Borrador" y creado_por = user_id
  And responde 201 con el id de la requisición
  And registra evento en REGISTRO_AUDITORIA

# Happy path — enviar a aprobación
Scenario: Reclutador envía requisición a aprobación
  Given requisición en estado "Borrador"
  When PATCH /requisiciones/:id/estado con { estado: "Pendiente_Aprobacion" }
  And el solicitante tiene rol Reclutador
  Then estado cambia a "Pendiente_Aprobacion"
  And se notifica al Responsable de Selección del tenant

# Happy path — aprobación
Scenario: Responsable aprueba la requisición
  Given requisición en estado "Pendiente_Aprobacion"
  When PATCH /requisiciones/:id/estado con { estado: "Aprobada" }
  And solicitante tiene rol Responsable o Admin
  Then estado cambia a "Aprobada"

# Error — salario incoherente
Scenario: salario_min > salario_max
  Given POST /requisiciones con salario_min = 50000 y salario_max = 30000
  Then responde 400 con "El salario mínimo no puede superar al máximo"

# Edge case — descripción vacía
Scenario: Descripción no enviada
  Given POST /requisiciones sin campo descripcion
  Then responde 400 con "La descripción del puesto es obligatoria"
```

**Dependencias previas:** TKT-03-01, TKT-02-01.

---

##### TKT-03-03 · [BE] Definición de etapas del proceso por requisición
- **US:** US-03 — Crear requisición de empleo
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar los endpoints para gestionar las ETAPA_PROCESO de una requisición: `POST /requisiciones/:id/etapas`, `PUT /requisiciones/:id/etapas/:etapa_id`, `DELETE /requisiciones/:id/etapas/:etapa_id`. Las etapas tienen nombre, orden y tipo (Cribado, Evaluacion, Entrevista, Oferta). Incluir etapas por defecto al crear una requisición.

**Acceptance Criteria:**
- Al crear una requisición se generan automáticamente 4 etapas por defecto: `Cribado (1) → Evaluación (2) → Entrevista (3) → Oferta (4)`.
- El reclutador puede añadir, renombrar, reordenar o eliminar etapas mientras la requisición está en "Borrador".
- No se pueden eliminar etapas si ya existen candidaturas en ellas.
- El orden de etapas es validado para no permitir duplicados de `indice_orden`.

**Dependencias previas:** TKT-03-02.

---

##### TKT-03-04 · [FE] Formulario de creación y edición de requisición
- **US:** US-03 — Crear requisición de empleo
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la React SPA el formulario de nueva requisición (`/requisiciones/nueva`) y la vista de edición (`/requisiciones/:id/editar`). Incluir todos los campos del modelo, validaciones frontend, editor de etapas del proceso y botón de "Enviar para aprobación".

**Acceptance Criteria:**
- Todos los campos obligatorios muestran error inline si están vacíos al intentar guardar.
- El selector de tipo de contrato es un dropdown con los valores del enum.
- Los campos de salario validan que min ≤ max en tiempo real.
- La sección de etapas permite reordenar por drag-and-drop y añadir/eliminar etapas personalizadas.
- El botón "Enviar para aprobación" solo está activo cuando todos los campos obligatorios están completos.
- El estado actual de la requisición es visible en un badge en la cabecera del formulario.

**Dependencias previas:** TKT-03-02, TKT-03-03.

---

##### TKT-03-05 · [FE] Listado y detalle de requisiciones
- **US:** US-03 — Crear requisición de empleo
- **Tipo:** FE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar la vista de listado de requisiciones (`/requisiciones`) con filtros por estado, departamento y fecha, y la vista de detalle (`/requisiciones/:id`) con toda la información de la requisición, sus etapas y el historial de cambios de estado.

**Acceptance Criteria:**
- El listado muestra título, estado (con badge de color), departamento y fecha de apertura.
- El listado es filtrable por estado (Borrador, Pendiente, Aprobada, Cerrada).
- El Responsable ve el botón "Aprobar" en las requisiciones en estado "Pendiente_Aprobacion".
- El historial de cambios de estado se muestra en orden cronológico.

**Dependencias previas:** TKT-03-04.

---

##### TKT-03-06 · [QA] Tests de requisición y flujo de aprobación
- **US:** US-03 — Crear requisición de empleo
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración que cubran: creación con campos válidos e inválidos, transiciones de estado permitidas y denegadas, y verificación de permisos por rol en cada transición.

**Dependencias previas:** TKT-03-02, TKT-03-03.

---

#### US-05 · Envío de candidatura por el candidato

---

##### TKT-05-01 · [INFRA] Configurar almacenamiento S3 para CVs
- **US:** US-05 — Envío de candidatura
- **Tipo:** INFRA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Configurar el bucket S3 (o compatible) para el almacenamiento de CVs. Definir la estructura de rutas (`/{empresa_id}/cvs/{candidatura_id}/{filename}`), las políticas de acceso (privado, solo accesible desde el backend), el límite de tamaño de archivo (máx. 5 MB) y los tipos MIME permitidos (PDF, DOCX).

**Acceptance Criteria:**
- Bucket configurado con acceso privado y política de solo lectura/escritura desde el servicio backend.
- Validación de tipos MIME en el presigned URL o en la capa de API.
- Presigned URLs generadas con expiración de 1 hora para lectura.

**Dependencias previas:** TKT-02-01.  
**Bloquea:** TKT-05-02.

---

##### TKT-05-02 · [BE] Endpoint de envío de candidatura y upload de CV
- **US:** US-05 — Envío de candidatura
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar el endpoint público `POST /candidaturas` (no requiere auth de usuario interno). Crea o reutiliza el registro CANDIDATO, crea la CANDIDATURA, sube el CV a S3 y crea el registro CURRICULUM. Encola el trabajo de cribado en la cola de trabajos del Motor de IA.

**Acceptance Criteria:**

```gherkin
# Happy path
Scenario: Candidato envía candidatura completa
  Given POST /candidaturas con { nombre, email, telefono, requisicion_id,
        publicacion_id, archivo_cv }
  When la requisición está en estado "Aprobada" y la publicación en "Activa"
  Then se crea o actualiza CANDIDATO
  And se crea CANDIDATURA con etapa = "Cribado" y estado = "Recibida"
  And el CV se sube a S3 y se crea CURRICULUM con url_archivo
  And se encola trabajo { candidatura_id, requisicion_id } en la cola de cribado
  And se envía email de confirmación al candidato con número de referencia
  And responde 201 con { candidatura_id, referencia }

# Edge case — candidato ya postulado al mismo puesto
Scenario: Email ya tiene candidatura activa para la misma requisición
  Given candidatura activa para email "juan@email.com" y requisicion_id X
  When POST /candidaturas con mismo email y requisicion_id X
  Then responde 409 con "Ya tienes una candidatura activa para este puesto"

# Edge case — oferta cerrada
Scenario: Candidato intenta postular a publicación cerrada
  Given PUBLICACION_EMPLEO con estado "Cerrada"
  When POST /candidaturas referenciando esa publicación
  Then responde 410 con "Esta oferta ya no está disponible"

# Error — formato CV inválido
Scenario: CV en formato .pages
  Given archivo CV con MIME type no permitido
  When POST /candidaturas
  Then responde 400 con "Formato de archivo no admitido. Se aceptan PDF y DOCX."
```

**Dependencias previas:** TKT-05-01, TKT-03-02.

---

##### TKT-05-03 · [BE] Endpoint de avance manual de candidatura entre etapas
- **US:** US-05 — Envío de candidatura
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar `PATCH /candidaturas/:id/etapa` para que el Reclutador o Responsable pueda mover manualmente una candidatura entre etapas (avanzar, descartar). Incluir registro en REGISTRO_AUDITORIA y notificación automática al candidato en cambios de estado relevantes (Descartada, Avanzada a Oferta).

**Acceptance Criteria:**
- Solo Reclutador y Responsable pueden invocar este endpoint.
- Al pasar a estado "Descartada" se envía email automático al candidato.
- Al pasar a etapa "Oferta" se envía notificación al Responsable.
- Todos los cambios de etapa se registran en REGISTRO_AUDITORIA con el user_id que realizó la acción.

**Dependencias previas:** TKT-05-02, TKT-03-03.

---

##### TKT-05-04 · [FE] Formulario público de candidatura en el portal
- **US:** US-05 — Envío de candidatura
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la aplicación Next.js del Portal del Candidato el formulario de candidatura (`/ofertas/:publicacion_id/aplicar`). El formulario incluye nombre, email, teléfono, URL de LinkedIn (opcional) y upload de CV. Debe ser responsive y accesible (WCAG AA).

**Acceptance Criteria:**
- Formulario accesible sin autenticación (público).
- Validación inline de campos obligatorios y formato de email.
- El upload de CV muestra nombre del archivo y tamaño; rechaza archivos > 5 MB con mensaje claro.
- Al enviar con éxito, muestra pantalla de confirmación con número de referencia y mensaje de que recibirá un email.
- Si la oferta está cerrada, el formulario muestra banner "Esta oferta ya no está disponible" y deshabilita el envío.

**Dependencias previas:** TKT-05-02.

---

##### TKT-05-05 · [QA] Tests de envío de candidatura
- **US:** US-05 — Envío de candidatura
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: envío exitoso con upload real a S3 mock, candidatura duplicada, oferta cerrada, formato de CV inválido, y verificación de que el trabajo de cribado se encola correctamente.

**Dependencias previas:** TKT-05-02.

---

#### US-04 · Publicación multicanal de ofertas

---

##### TKT-04-01 · [SPIKE] Evaluar APIs de portales de empleo para MVP
- **US:** US-04 — Publicación multicanal
- **Tipo:** SPIKE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Investigar los requisitos de acceso a las APIs de LinkedIn Jobs y Indeed para publicación programática. Evaluar si los procesos de partnership/aprobación son viables en el timeline del MVP. Decisión documentada: si la aprobación de partnership tarda más de 2 semanas, el MVP arranca solo con el portal corporativo interno y un canal "Manual" (genera URL pública de la oferta). LinkedIn e Indeed se añaden en Sprint 1.1.

**Criterios de aceptación:**
- Decisión documentada con timeline de aprobación estimado para cada portal.
- Para el MVP: Portal Corporativo (obligatorio) + canal "Enlace externo manual" (como fallback).

**Dependencias previas:** TKT-03-02.  
**Bloquea:** TKT-04-03.

---

##### TKT-04-02 · [BE] Endpoint de publicación y gestión de PUBLICACION_EMPLEO
- **US:** US-04 — Publicación multicanal
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `POST /requisiciones/:id/publicaciones` para crear una publicación en un canal, `GET /requisiciones/:id/publicaciones` para listar publicaciones y su estado, y `PATCH /publicaciones/:id/estado` para cerrar publicaciones. Una requisición debe estar en estado "Aprobada" para poder publicar.

**Acceptance Criteria:**

```gherkin
# Happy path — publicar en portal corporativo
Scenario: Reclutador publica en portal corporativo
  Given requisición en estado "Aprobada"
  When POST /requisiciones/:id/publicaciones con { canal: "Portal_Corporativo" }
  Then se crea PUBLICACION_EMPLEO con estado "Activa" y publicado_en = now
  And la oferta es accesible en la URL pública del portal

# Error — publicar oferta no aprobada
Scenario: Requisición en estado "Borrador"
  Given requisición en estado "Borrador"
  When POST /requisiciones/:id/publicaciones
  Then responde 422 con "Solo se pueden publicar requisiciones aprobadas"

# Edge case — cerrar publicación con candidaturas en curso
Scenario: Reclutador cierra una publicación que tiene candidaturas activas
  Given publicación "Activa" con 15 candidaturas asociadas
  When PATCH /publicaciones/:id/estado con { estado: "Cerrada" }
  Then la publicación cambia a estado "Cerrada"
  And las candidaturas existentes NO se ven afectadas (R2 resuelto)
  And el endpoint responde 200 con warning "Las candidaturas en curso continúan activas"
```

**Dependencias previas:** TKT-04-01, TKT-03-02.

---

##### TKT-04-03 · [BE] Integración con portal corporativo (página pública de ofertas)
- **US:** US-04 — Publicación multicanal
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar el endpoint público `GET /portal/:subdominio/ofertas` que devuelva las ofertas activas del tenant. Este endpoint es consumido por el Portal del Candidato (Next.js) para mostrar las ofertas disponibles con la marca corporativa.

**Acceptance Criteria:**
- Devuelve solo PUBLICACION_EMPLEO con estado "Activa" del tenant identificado por subdominio.
- El response incluye: título, departamento, ubicación, tipo de contrato, descripción (truncada a 500 chars) y URL de candidatura.
- Paginado (máx. 20 por página).
- Caché de 5 minutos en Redis para reducir carga de BD.

**Dependencias previas:** TKT-04-02, TKT-02-01.

---

##### TKT-04-04 · [FE] Panel de publicaciones en el dashboard del reclutador
- **US:** US-04 — Publicación multicanal
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar en la React SPA la sección de publicaciones dentro del detalle de una requisición. Incluir: listado de canales disponibles con checkboxes, estado de cada publicación (Activa/Cerrada), indicador de "credenciales no configuradas" para canales externos pendientes de integración, y botón de "Cerrar publicación".

**Acceptance Criteria:**
- Los canales disponibles se cargan dinámicamente de la configuración del tenant.
- Los canales sin credenciales configuradas muestran tooltip "Requiere configuración en Ajustes".
- El estado de cada publicación se actualiza en tiempo real (polling de 30s o WebSocket).
- Confirmación modal antes de cerrar una publicación con candidaturas activas.

**Dependencias previas:** TKT-04-02, TKT-04-03.

---

##### TKT-04-05 · [FE] Página pública de ofertas en el Portal del Candidato
- **US:** US-04 — Publicación multicanal
- **Tipo:** FE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar en el Portal del Candidato (Next.js) la página de listado de ofertas de la empresa (`/{subdominio}/ofertas`) y la página de detalle de cada oferta con el botón "Solicitar empleo". Aplicar la marca corporativa del tenant (logo, colores).

**Acceptance Criteria:**
- La página muestra el logo y nombre de la empresa del tenant.
- Las ofertas se listan con título, departamento, ubicación y tipo de contrato.
- El botón "Solicitar empleo" navega al formulario de candidatura (TKT-05-04).
- Si no hay ofertas activas, se muestra el mensaje "No hay ofertas disponibles en este momento."
- La página está optimizada para SEO (server-side rendering con Next.js).

**Dependencias previas:** TKT-04-03, TKT-05-04.

---

##### TKT-04-06 · [QA] Tests de publicación multicanal
- **US:** US-04 — Publicación multicanal
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: publicación en portal corporativo, intento de publicar requisición no aprobada, cierre de publicación con verificación de que candidaturas activas no se ven afectadas (R2), y acceso público al listado de ofertas por subdominio.

**Dependencias previas:** TKT-04-02, TKT-04-03.

---

### Sprint 2 — Motor inteligente

#### US-06 · Cribado automático de candidatos con IA

---

##### TKT-06-01 · [SPIKE] Diseñar prompt de cribado y validar calidad con datos reales
- **US:** US-06 — Cribado automático con IA
- **Tipo:** SPIKE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Diseñar el prompt de cribado inicial para el LLM (OpenAI GPT-4o o Anthropic Claude). Validar la calidad del output con al menos 10 CVs de prueba reales (anonimizados) frente a 2 descripciones de puesto distintas. Documentar el esquema JSON de respuesta esperado. Definir el SLA de procesamiento objetivo (< 2 min por CV). Estimar el coste de API por CV procesado y definir el presupuesto máximo por tenant/mes.

**Criterios de aceptación:**
- Prompt documentado con estructura: system prompt + user prompt + schema de respuesta JSON.
- Tasa de respuestas JSON válidas ≥ 90 % en los 10 CVs de prueba.
- Coste estimado por CV documentado y validado con el equipo de producto.
- SLA definido: < 2 min p95.

**Dependencias previas:** TKT-05-02.  
**Bloquea:** TKT-06-03.

---

##### TKT-06-02 · [INFRA] Configurar cola de trabajos asíncrona (Redis Streams)
- **US:** US-06 — Cribado automático con IA
- **Tipo:** INFRA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Configurar Redis Streams como cola de trabajos para el Motor de Cribado. Implementar el productor (en el Servicio de Candidaturas) y el consumidor (en el Motor de IA). Incluir dead-letter queue para trabajos fallidos tras 3 reintentos y monitorización de profundidad de cola.

**Acceptance Criteria:**
- El productor encola `{ candidatura_id, requisicion_id }` correctamente tras cada candidatura recibida.
- El consumidor procesa trabajos en orden FIFO.
- Trabajos fallidos tras 3 reintentos van a la dead-letter stream.
- La profundidad de cola es visible como métrica de Prometheus.

**Dependencias previas:** TKT-05-02.  
**Bloquea:** TKT-06-03.

---

##### TKT-06-03 · [BE] Motor de cribado IA — pipeline completo
- **US:** US-06 — Cribado automático con IA
- **Tipo:** BE · **Complejidad:** L · **Story Points:** 13

**Descripción:**  
Implementar el pipeline completo del Motor de Cribado (Python/FastAPI) según el diseño C4 del PRD: JobConsumer → DocumentFetcher → CVParser → PromptBuilder → LLMClient (con retry + fallback HeuristicScorer) → ResponseParser → ScoreNormalizer → ResultWriter → EventPublisher. Cada componente según las responsabilidades del diagrama de componentes C4 nivel 3.

**Acceptance Criteria:**

```gherkin
# Happy path — cribado exitoso
Scenario: Pipeline procesa candidatura con CV PDF legible
  Given trabajo en cola con candidatura_id y requisicion_id válidos
  When el JobConsumer consume el trabajo
  Then el CVParser extrae el texto del PDF con éxito
  And el PromptBuilder construye el prompt con CV + JD + competencias
  And el LLMClient llama a la API del LLM y recibe JSON válido
  And el ResponseParser valida el schema JSON
  And el ScoreNormalizer normaliza puntuaciones a escala 0-100
  And el ResultWriter persiste puntuacion_adecuacion_ia y resumen_ia en CANDIDATURA
  And el EventPublisher publica "cribado_completado" en el bus de eventos
  And el procesamiento completo toma < 2 minutos (SLA p95)

# Edge case — API LLM no disponible
Scenario: LLM devuelve 503 tras 3 reintentos con backoff exponencial
  Given el LLM externo está caído
  When el LLMClient agota los reintentos
  Then invoca el HeuristicScorer como fallback
  And persiste la puntuación con source = "HEURISTIC_FALLBACK"
  And el campo resumen_ia incluye "[Puntuación estimada — sin IA]"

# Edge case — CV es imagen escaneada sin texto
Scenario: CVParser no puede extraer texto del PDF
  Given PDF sin capa de texto OCR
  When CVParser intenta extracción
  Then lanza ParseError y el pipeline marca la candidatura con flag "revision_manual_requerida"
  And notifica al Reclutador vía Servicio de Notificaciones
  And NO encola a dead-letter (es un fallo esperado, no un error del sistema)

# Error — respuesta JSON malformada del LLM
Scenario: LLM devuelve texto libre en lugar de JSON
  Given respuesta del LLM sin estructura JSON válida
  When ResponseParser intenta deserializar
  Then encola el trabajo en dead-letter stream
  And registra métricas de fallo en Prometheus (record_failure)
  And NO escribe datos en CANDIDATURA
```

**Notas técnicas:** Implementar `TokenBudgetGuard` para truncar CVs que superen el presupuesto de tokens. El campo `ResponseSource` debe persistirse para trazabilidad en la analítica (R3).

**Dependencias previas:** TKT-06-01, TKT-06-02.

---

##### TKT-06-04 · [FE] Vista de lista reducida de candidatos con puntuación IA
- **US:** US-06 — Cribado automático con IA
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la React SPA la vista de candidatos de una requisición (`/requisiciones/:id/candidatos`) con la lista ordenada por `puntuacion_adecuacion_ia`, codificación de color (verde ≥ 70, amarillo 40–69, rojo < 40), resumen en lenguaje natural expandible y badge diferenciador "Puntuación estimada" para puntuaciones de fallback heurístico. Actualización en tiempo real vía WebSocket al recibir el evento "cribado_completado".

**Acceptance Criteria:**
- La lista muestra candidatos ordenados de mayor a menor puntuación.
- El color del badge de puntuación sigue la escala definida.
- Al expandir un candidato se muestra el resumen IA completo.
- Los candidatos con `source = HEURISTIC_FALLBACK` muestran badge amarillo con tooltip explicativo.
- Los candidatos con `revision_manual_requerida` muestran icono de advertencia con mensaje.
- Las nuevas puntuaciones aparecen en tiempo real sin recargar la página.

**Dependencias previas:** TKT-06-03.

---

##### TKT-06-05 · [QA] Tests del Motor de Cribado IA
- **US:** US-06 — Cribado automático con IA
- **Tipo:** QA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Tests unitarios de cada componente del pipeline con mocks del LLM. Tests de integración del pipeline completo con LLM mockeado. Tests del escenario de fallback heurístico y del escenario de CV no parseable.

**Cobertura mínima:**
- CVParser: PDF válido, PDF imagen, DOCX.
- LLMClient: éxito, 429 con retry, 503 con fallback.
- ResponseParser: JSON válido, JSON malformado → dead-letter.
- Pipeline e2e: candidatura → puntuación persistida.

**Dependencias previas:** TKT-06-03.

---

#### US-11 · Portal del candidato — seguimiento de estado

---

##### TKT-11-01 · [BE] Endpoints de consulta de estado de candidatura para candidatos
- **US:** US-11 — Portal del candidato
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar el endpoint `GET /portal/candidaturas/:referencia` (autenticado por token de candidato) que devuelva el estado actual, etapa y última actualización de la candidatura. Implementar también `POST /portal/auth/token` para que el candidato autentique con su email y el número de referencia recibido en el email de confirmación.

**Acceptance Criteria:**
- El candidato se autentica con `{ email, referencia }` y recibe un token de sesión ligero (exp 24h).
- `GET /portal/candidaturas/:referencia` devuelve solo datos del candidato autenticado (validación de aislamiento).
- Si la candidatura pertenece a un tenant diferente al del subdominio actual, responde 404 (aislamiento multiinquilino).
- Los cambios de estado se reflejan en tiempo real si el candidato tiene la sesión abierta (SSE o polling de 30s).

**Dependencias previas:** TKT-05-02.

---

##### TKT-11-02 · [BE] Notificaciones automáticas de cambio de estado al candidato
- **US:** US-11 — Portal del candidato
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar en el Servicio de Notificaciones el envío automático de emails al candidato en los siguientes eventos: candidatura recibida (confirmación), avance de etapa, descarte y oferta enviada. Los emails deben usar plantillas con la marca corporativa del tenant y enlace al portal.

**Acceptance Criteria:**
- Candidatura recibida → email con referencia y enlace al portal (< 2 min de latencia).
- Cambio de etapa (avance) → email "¡Tu candidatura ha avanzado!".
- Descarte → email de comunicación de no continuidad (sin revelar motivo interno).
- Oferta enviada → email con enlace directo a la oferta en el portal.
- Todos los emails incluyen el logo y nombre de la empresa del tenant.
- Emails no enviados si el candidato tiene email inválido → log de error, no excepción del sistema.

**Dependencias previas:** TKT-05-03, TKT-05-02.

---

##### TKT-11-03 · [FE] Vista de seguimiento de candidatura en el portal
- **US:** US-11 — Portal del candidato
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en el Portal del Candidato (Next.js) la vista de seguimiento (`/mis-candidaturas/:referencia`) con: stepper visual de progreso de etapas, fecha de último cambio, mensajes del reclutador y acceso a la aceptación de oferta cuando esté disponible.

**Acceptance Criteria:**
- El stepper muestra todas las etapas del proceso de la requisición, resaltando la etapa actual.
- Las etapas futuras se muestran en gris; la actual en color primario; las pasadas con check.
- Si la candidatura está en estado "Descartada", el stepper muestra mensaje "Proceso finalizado" sin revelar motivos internos.
- Si hay una oferta en estado "Enviada", se muestra un CTA prominente "Ver y responder oferta".
- La vista se actualiza automáticamente cada 30 segundos (polling ligero).
- La página es responsive y funciona correctamente en móvil.

**Dependencias previas:** TKT-11-01, TKT-11-02.

---

##### TKT-11-04 · [QA] Tests del portal del candidato
- **US:** US-11 — Portal del candidato
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: autenticación con referencia válida e inválida, aislamiento multiinquilino (candidato de empresa A no ve datos de empresa B), envío de email de confirmación y actualización de estado en tiempo real.

**Dependencias previas:** TKT-11-01, TKT-11-02.

---

### Sprint 3 — Proceso de evaluación y entrevistas

#### US-10 · Scorecard estructurada del entrevistador

---

##### TKT-10-01 · [BE] CRUD de competencias por empresa
- **US:** US-10 — Scorecard del entrevistador
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar los endpoints para gestionar el catálogo de COMPETENCIAS del tenant: `POST /competencias`, `GET /competencias`, `PUT /competencias/:id`, `DELETE /competencias/:id`. Las competencias tienen nombre, descripción y categoría (Técnica, Comportamental, Cultural). Incluir 5 competencias genéricas por defecto al activar el tenant.

**Acceptance Criteria:**
- Solo Admin y Reclutador pueden gestionar competencias.
- Al crear el tenant se generan 5 competencias genéricas por defecto (Comunicación, Resolución de problemas, Trabajo en equipo, Adaptabilidad, Orientación a resultados).
- Una competencia no puede eliminarse si está referenciada en ELEMENTO_SCORECARD existente.

**Dependencias previas:** TKT-02-02.

---

##### TKT-10-02 · [BE] Gestión del estado de entrevista (Confirmada → Realizada) — Resuelve R6
- **US:** US-10 — Scorecard del entrevistador
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar la lógica que resuelve el riesgo R6: la transición de estado de ENTREVISTA de "Confirmada" a "Realizada". Dos mecanismos: (1) automático — job programado que marca como "Realizada" las entrevistas cuya `programada_en + duracion_minutos` ya haya pasado; (2) manual — `PATCH /entrevistas/:id/estado` para que el Reclutador la marque manualmente.

**Acceptance Criteria:**
- Job programado ejecuta cada 15 minutos y transiciona entrevistas pasadas a "Realizada".
- El reclutador puede marcar manualmente una entrevista como "Realizada" si el job aún no ha corrido.
- Al pasar a "Realizada", se notifica a todos los panelistas: "Ya puedes enviar tu scorecard para [candidato]".
- No se pueden enviar scorecards para entrevistas en estado distinto a "Realizada".

**Dependencias previas:** TKT-03-02, TKT-05-02.  
**Bloquea:** TKT-10-03.

---

##### TKT-10-03 · [BE] Endpoints de scorecard y elementos de competencia
- **US:** US-10 — Scorecard del entrevistador
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `POST /entrevistas/:id/scorecards` para crear una scorecard con sus elementos de competencia. La scorecard pertenece al usuario autenticado (panelista). Implementar `GET /candidaturas/:id/scorecards` para que el Responsable consulte todas las scorecards de una candidatura con su agregación de puntuaciones.

**Acceptance Criteria:**

```gherkin
# Happy path — scorecard enviada
Scenario: Entrevistador envía scorecard completa
  Given entrevista en estado "Realizada"
  And el usuario autenticado es panelista de la entrevista
  When POST /entrevistas/:id/scorecards con { puntuaciones_competencias[],
       comentario_general, recomendacion }
  Then se crea SCORECARD con enviado_en = now
  And se crean ELEMENTO_SCORECARD por cada competencia puntuada
  And el Responsable recibe notificación in-app "Nueva scorecard disponible"
  And responde 201

# Error — competencia sin puntuar
Scenario: Payload incompleto con competencia sin puntuación
  Given la requisición tiene 4 competencias obligatorias
  When POST /scorecards con solo 3 competencias en el array
  Then responde 400 con "Debes puntuar todas las competencias obligatorias"

# Error — usuario no es panelista
Scenario: Usuario no asignado como panelista intenta enviar scorecard
  Given el usuario no existe en PANELISTA_ENTREVISTA para esa entrevista
  When POST /entrevistas/:id/scorecards
  Then responde 403 con "No tienes permiso para evaluar esta entrevista"

# Edge case — scorecard ya enviada por el mismo panelista
Scenario: Panelista intenta enviar segunda scorecard para la misma entrevista
  Given ya existe una SCORECARD del mismo panelista para esa entrevista
  When POST /entrevistas/:id/scorecards
  Then responde 409 con "Ya has enviado tu evaluación para esta entrevista"
```

**Dependencias previas:** TKT-10-01, TKT-10-02.

---

##### TKT-10-04 · [FE] Formulario de scorecard para el entrevistador
- **US:** US-10 — Scorecard del entrevistador
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la React SPA el formulario de scorecard (`/entrevistas/:id/scorecard`). Muestra las competencias del kit de entrevista de la requisición, controles de puntuación (escala Likert 1–5 con etiquetas), campo de comentario por competencia (opcional), comentario general (obligatorio) y selector de recomendación (No avanzar / Dudoso / Avanzar).

**Acceptance Criteria:**
- El formulario carga las competencias del kit de entrevista de la requisición; si no hay kit, usa las competencias genéricas del tenant con aviso visual.
- Cada competencia tiene control de puntuación 1–5 con etiquetas descriptivas (1=Muy por debajo / 5=Supera expectativas).
- El botón "Enviar" permanece deshabilitado hasta que todas las competencias estén puntuadas y el comentario general completado.
- Tras enviar con éxito, muestra confirmación "Tu evaluación ha sido registrada" y deshabilita el formulario.
- Si la entrevista no está en estado "Realizada", muestra mensaje "Aún no puedes enviar la evaluación" con la fecha programada.

**Dependencias previas:** TKT-10-03.

---

##### TKT-10-05 · [QA] Tests de scorecard y competencias
- **US:** US-10 — Scorecard del entrevistador
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: envío exitoso de scorecard completa, intento de envío con competencias incompletas, intento de envío por usuario no panelista, doble envío por el mismo panelista, y transición automática de entrevista a "Realizada".

**Dependencias previas:** TKT-10-03, TKT-10-02.

---

#### US-09 · Programación automatizada de entrevistas

---

##### TKT-09-01 · [SPIKE] Evaluar OAuth2 con Google Calendar y Outlook para entornos corporativos
- **US:** US-09 — Programación de entrevistas
- **Tipo:** SPIKE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Investigar los requisitos de aprobación OAuth2 de Google Workspace y Microsoft Azure AD para aplicaciones empresariales. Evaluar si el flujo puede completarse sin aprobación de marketplace (aplicación privada de tenant). Documentar el flujo de autorización por panelista. Definir el fallback de programación manual como mecanismo garantizado para el MVP si la integración de calendario no está disponible.

**Criterios de aceptación:**
- Decisión documentada: ¿app pública o privada de Google Workspace?
- Fallback manual diseñado: endpoint para crear ENTREVISTA con fecha/hora manual sin integración de calendario.
- Gestión de zonas horarias documentada (almacenar en UTC, mostrar en zona local del usuario).

**Dependencias previas:** TKT-03-02.  
**Bloquea:** TKT-09-02, TKT-09-03.

---

##### TKT-09-02 · [BE] Integración OAuth2 con Google Calendar y Outlook
- **US:** US-09 — Programación de entrevistas
- **Tipo:** BE · **Complejidad:** L · **Story Points:** 8

**Descripción:**  
Implementar el flujo OAuth2 para que cada panelista (usuario de LTI) autorice a LTI a leer su disponibilidad de calendario. Almacenar tokens de acceso y refresco de forma segura (cifrados en BD). Implementar `GET /calendarios/:user_id/disponibilidad` que devuelva los slots libres del usuario en un rango de fechas dado.

**Acceptance Criteria:**
- Flujo OAuth2 completo con Google (redirect → callback → almacenamiento de tokens).
- Flujo OAuth2 completo con Outlook/Microsoft (idem).
- Los tokens de acceso se refrescan automáticamente antes de su expiración.
- `GET /disponibilidad` devuelve slots libres considerando todos los calendarios del usuario (no solo el primario).
- Si un usuario no tiene calendario integrado, el endpoint devuelve `{ disponibilidad: null, integrado: false }` sin error.
- Todos los tokens se almacenan cifrados con AES-256 en BD.

**Dependencias previas:** TKT-09-01.

---

##### TKT-09-03 · [BE] Algoritmo de cálculo de franjas y programación de entrevista
- **US:** US-09 — Programación de entrevistas
- **Tipo:** BE · **Complejidad:** L · **Story Points:** 8

**Descripción:**  
Implementar `POST /candidaturas/:id/entrevistas/programar` que: recibe el panel de entrevistadores y la duración, lee la disponibilidad de cada panelista, calcula las intersecciones libres en los próximos 5 días laborables (ampliable a 10 si no hay coincidencias), crea la ENTREVISTA con estado "Propuesta" y envía las opciones al candidato vía portal. Implementar `PATCH /entrevistas/:id/confirmar-franja` para cuando el candidato elija.

**Acceptance Criteria:**

```gherkin
# Happy path — programación exitosa
Scenario: 3 panelistas con coincidencia en los próximos 5 días
  Given 3 panelistas con calendarios integrados y disponibilidad coincidente
  When POST /candidaturas/:id/entrevistas/programar con { panelistas[], duracion_minutos }
  Then calcula 3 franjas disponibles en los próximos 5 días laborables
  And crea ENTREVISTA con estado "Propuesta" y las 3 franjas como opciones
  And envía al candidato email con enlace para seleccionar franja

# Happy path — candidato selecciona franja
Scenario: Candidato elige franja horaria
  Given ENTREVISTA en estado "Propuesta" con 3 opciones
  When PATCH /entrevistas/:id/confirmar-franja con { franja_seleccionada }
  Then estado → "Confirmada" con programada_en = franja seleccionada
  And crea evento en Google Calendar / Outlook para todos los panelistas y candidato
  And envía invitaciones de calendario y confirmaciones por email
  And programa recordatorios automáticos 24h y 1h antes

# Edge case — sin coincidencias en 5 días
Scenario: Agendas llenas en ventana de 5 días laborables
  Given los 3 panelistas no tienen coincidencias en 5 días
  When el algoritmo amplía la búsqueda a 10 días
  And sigue sin coincidencias
  Then notifica al Reclutador: "No se encontraron franjas disponibles. Considera reducir el panel."
  And habilita el modo de programación manual como fallback

# Edge case — panelista sin calendario integrado
Scenario: Uno de los panelistas no tiene calendario conectado
  Given el panelista "Pedro" tiene integrado = false
  When el Reclutador intenta programar incluyendo a Pedro
  Then LTI advierte en la respuesta: { panelistas_sin_calendario: ["Pedro"] }
  And permite continuar con programación manual para ese panelista
```

**Dependencias previas:** TKT-09-02.

---

##### TKT-09-04 · [BE] Fallback: creación manual de entrevista sin calendario
- **US:** US-09 — Programación de entrevistas
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar `POST /candidaturas/:id/entrevistas/manual` para que el Reclutador cree una entrevista directamente con fecha, hora y panelistas sin necesidad de integración de calendario. Este endpoint es el fallback garantizado del MVP cuando la integración de calendario no está disponible.

**Acceptance Criteria:**
- Crea ENTREVISTA con estado "Confirmada" directamente.
- Envía email de confirmación al candidato y a los panelistas con los datos de la entrevista.
- Si el panelista tiene calendario integrado, también crea el evento en su calendario.
- No requiere que ningún panelista tenga calendario integrado.

**Dependencias previas:** TKT-03-02, TKT-05-02.

---

##### TKT-09-05 · [FE] Panel de programación de entrevistas
- **US:** US-09 — Programación de entrevistas
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la React SPA el panel de programación de entrevistas accesible desde el perfil del candidato. Incluir: selector de panelistas, selector de duración, visualización de disponibilidad de cada panelista, lista de franjas calculadas y confirmación de programación. Incluir toggle "Programación manual" como fallback visible.

**Acceptance Criteria:**
- El selector de panelistas muestra para cada usuario si tiene el calendario integrado (icono verde/rojo).
- Tras seleccionar panelistas y duración, el botón "Calcular franjas" ejecuta el algoritmo y muestra las opciones.
- Si no hay franjas disponibles, muestra el mensaje del sistema con opción de ampliar la ventana o cambiar panelistas.
- El toggle "Programación manual" permite introducir fecha y hora directamente.
- El estado de la entrevista (Propuesta/Confirmada) es visible en el perfil del candidato.

**Dependencias previas:** TKT-09-03, TKT-09-04.

---

##### TKT-09-06 · [FE] Selección de franja horaria en el Portal del Candidato
- **US:** US-09 — Programación de entrevistas
- **Tipo:** FE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar en el Portal del Candidato la vista de selección de franja horaria (`/portal/entrevistas/:token/elegir-franja`). Muestra las 3 opciones de horario con fecha, hora y duración. El candidato selecciona una y recibe confirmación con enlace de reunión (si aplica).

**Acceptance Criteria:**
- Las franjas se muestran en la zona horaria del candidato (detectada por browser o solicitada).
- Al seleccionar una franja, se muestra confirmación inmediata con los datos de la entrevista.
- Si la entrevista ya fue confirmada por otro medio, la página muestra "Esta entrevista ya ha sido programada".
- El enlace de reunión (Zoom/Teams) se muestra en la confirmación si está disponible.

**Dependencias previas:** TKT-09-03, TKT-11-03.

---

##### TKT-09-07 · [QA] Tests de programación de entrevistas
- **US:** US-09 — Programación de entrevistas
- **Tipo:** QA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Tests de integración con calendarios mockeados para: cálculo de franjas con 3 panelistas disponibles, expansión de ventana sin coincidencias, fallback manual, confirmación de franja por el candidato y creación de evento en calendario.

**Dependencias previas:** TKT-09-03, TKT-09-04.

---

#### US-07 · Espacio de trabajo colaborativo

---

##### TKT-07-01 · [BE] Endpoint de perfil consolidado del candidato con datos en tiempo real
- **US:** US-07 — Espacio colaborativo
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `GET /candidaturas/:id/perfil-completo` que devuelva en una sola respuesta: datos del candidato, CV (presigned URL), puntuación IA y resumen, historial de etapas, lista de entrevistas con sus scorecards y puntuaciones agregadas por competencia (media ponderada — resuelve R4), y la última oferta si existe. Implementar también el canal WebSocket `candidatura:{id}` para push de actualizaciones en tiempo real.

**Acceptance Criteria:**
- El endpoint devuelve todos los datos consolidados en una sola llamada (evitar N+1 queries).
- La `puntuacion_consenso` se calcula como media simple de todas las puntuaciones por competencia de todas las scorecards enviadas (fórmula de consenso documentada — R4 resuelto).
- Cuando se publica el evento "scorecard_enviada" o "cribado_completado", el servidor emite la actualización por WebSocket al canal `candidatura:{id}`.
- Solo usuarios del mismo tenant con roles Reclutador, Responsable o Entrevistador (de esa candidatura) pueden acceder.

**Dependencias previas:** TKT-10-03, TKT-06-03.

---

##### TKT-07-02 · [BE] Sistema de comentarios con @menciones
- **US:** US-07 — Espacio colaborativo
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `POST /candidaturas/:id/comentarios`, `GET /candidaturas/:id/comentarios` y `POST /candidaturas/:id/comentarios/:comentario_id/respuestas`. Los comentarios soportan @menciones (formato `@user_id`). Al detectar una mención, el Servicio de Notificaciones envía notificación in-app y email al usuario mencionado.

**Acceptance Criteria:**

```gherkin
# Happy path — comentario con @mención
Scenario: Responsable menciona a un entrevistador
  Given POST /candidaturas/:id/comentarios con { texto: "@uuid_ana ¿puedes ampliar tu valoración?" }
  When se persiste el comentario
  Then se detecta la mención a uuid_ana
  And se envía notificación in-app a Ana con texto y enlace al perfil del candidato
  And el comentario es visible para todo el equipo del tenant en tiempo real (WebSocket)

# Edge case — comentarios simultáneos
Scenario: Dos usuarios comentan al mismo tiempo
  Given dos peticiones POST simultáneas de usuarios distintos
  When ambas se persisten en BD
  Then ambos comentarios tienen timestamps distintos y se muestran en orden cronológico
  And no hay pérdida de datos (sin condición de carrera)

# Error — mención a usuario de otro tenant
Scenario: El texto contiene @uuid de usuario de empresa diferente
  Given @uuid_externo no pertenece al tenant de la candidatura
  When se procesa la mención
  Then la mención se ignora silenciosamente (no se envía notificación)
  And el comentario se guarda sin la mención activa
```

**Dependencias previas:** TKT-07-01.

---

##### TKT-07-03 · [BE] Acción "Tomar decisión final" — Resuelve R7
- **US:** US-07 — Espacio colaborativo
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar `POST /candidaturas/:id/decision` con `{ decision: "Avanzar_a_Oferta" | "Descartar" }`. Esta acción, exclusiva del Responsable de Selección, cambia la etapa de la candidatura y activa las notificaciones correspondientes. Resuelve el riesgo R7 identificado en el análisis de consistencia.

**Acceptance Criteria:**
- Solo el rol Responsable de Selección puede invocar este endpoint.
- `Avanzar_a_Oferta` → cambia la etapa de CANDIDATURA a "Oferta" y notifica al Reclutador.
- `Descartar` → cambia el estado de CANDIDATURA a "Descartada", envía email al candidato y notifica al equipo.
- Ambas acciones quedan registradas en REGISTRO_AUDITORIA con user_id y timestamp.
- No puede tomarse decisión si no hay al menos una scorecard enviada (validación de negocio).

**Dependencias previas:** TKT-07-01, TKT-10-03.  
**Bloquea:** TKT-12-02 (creación de oferta requiere etapa "Oferta").

---

##### TKT-07-04 · [FE] Vista de perfil consolidado del candidato con colaboración en tiempo real
- **US:** US-07 — Espacio colaborativo
- **Tipo:** FE · **Complejidad:** L · **Story Points:** 8

**Descripción:**  
Implementar en la React SPA la vista de perfil del candidato (`/candidaturas/:id`) con: sección de datos y CV, puntuación IA con badge de fuente, vista de consenso con radar chart de competencias por puntuaciones agregadas, lista de scorecards por entrevistador expandibles, sección de comentarios con @menciones y botones de decisión final (solo visible para Responsable).

**Acceptance Criteria:**
- Todos los datos del perfil se cargan en la carga inicial desde `GET /perfil-completo`.
- El radar chart de competencias muestra la puntuación media por competencia de todas las scorecards.
- Los comentarios se actualizan en tiempo real vía WebSocket sin recargar la página.
- El input de comentario detecta `@` y muestra un autocomplete con los miembros del equipo del tenant.
- Los botones "Avanzar a Oferta" y "Descartar" solo son visibles para usuarios con rol Responsable.
- Si no hay scorecards, el radar chart muestra un placeholder "Aún no hay evaluaciones".

**Dependencias previas:** TKT-07-01, TKT-07-02, TKT-07-03.

---

##### TKT-07-05 · [QA] Tests del espacio colaborativo
- **US:** US-07 — Espacio colaborativo
- **Tipo:** QA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Tests de integración para: carga del perfil consolidado, publicación de comentario con mención y verificación de notificación, concurrencia de comentarios simultáneos, acción de decisión final con verificación de cambio de etapa, y validación de que no se puede decidir sin scorecards.

**Dependencias previas:** TKT-07-01, TKT-07-02, TKT-07-03.

---

### Sprint 4 — Cierre del ciclo y datos

#### US-12 · Gestión y envío de oferta digital

---

##### TKT-12-01 · [BE] Generación de documento de oferta en PDF
- **US:** US-12 — Gestión de oferta digital
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar la generación de documentos PDF de carta de oferta a partir de una plantilla configurable por tenant. La plantilla incluye: logo de la empresa, nombre del candidato, título del puesto, compensación, fecha de inicio y cláusulas estándar. El PDF generado se almacena en S3.

**Acceptance Criteria:**
- La plantilla es editable por el Admin del tenant (texto libre con variables: `{{candidato_nombre}}`, `{{titulo}}`, `{{compensacion}}`, `{{fecha_inicio}}`).
- El PDF generado se almacena en `/{empresa_id}/ofertas/{oferta_id}/oferta.pdf`.
- La URL del PDF en S3 se persiste en `OFERTA.url_documento`.
- El PDF generado es accesible mediante presigned URL con expiración de 7 días.

**Dependencias previas:** TKT-07-03, TKT-05-01.

---

##### TKT-12-02 · [BE] Endpoints de gestión del ciclo de vida de la oferta
- **US:** US-12 — Gestión de oferta digital
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar los endpoints del ciclo de vida de OFERTA: `POST /candidaturas/:id/ofertas` (crear y enviar), `GET /candidaturas/:id/ofertas/:oferta_id` (detalle), `PATCH /ofertas/:id/responder` (candidato acepta/rechaza). Estados: `Borrador → Enviada → Aceptada | Rechazada | Expirada`. Incluir job de expiración automática.

**Acceptance Criteria:**

```gherkin
# Happy path — crear y enviar oferta
Scenario: Reclutador crea y envía oferta
  Given candidatura en etapa "Oferta"
  When POST /candidaturas/:id/ofertas con { titulo, compensacion, fecha_inicio }
  Then se genera el PDF y se crea OFERTA con estado "Enviada" y enviado_en = now
  And se envía email al candidato con enlace al portal para ver la oferta
  And el Responsable recibe notificación "Oferta enviada a [candidato]"

# Happy path — aceptación
Scenario: Candidato acepta la oferta
  Given OFERTA en estado "Enviada"
  When PATCH /ofertas/:id/responder con { respuesta: "Aceptada" }
  And el usuario autenticado es el candidato propietario de la oferta
  Then OFERTA.estado → "Aceptada" con respondido_en = now
  And CANDIDATURA.estado → "Contratado"
  And se activa el traspaso al HRIS (evento "candidato_contratado" en bus de eventos)
  And Reclutador y Responsable reciben notificación de aceptación

# Edge case — oferta expirada sin respuesta
Scenario: Job de expiración marca oferta como expirada
  Given OFERTA con enviado_en hace 5 días y estado "Enviada"
  When el job de expiración ejecuta
  Then OFERTA.estado → "Expirada"
  And el Reclutador recibe notificación "La oferta para [candidato] ha expirado"

# Error — segunda oferta con una ya activa
Scenario: Intento de crear segunda oferta activa para la misma candidatura
  Given ya existe OFERTA en estado "Enviada" para la candidatura
  When POST /candidaturas/:id/ofertas
  Then responde 409 con "Ya existe una oferta activa para este candidato"
```

**Dependencias previas:** TKT-12-01, TKT-07-03.

---

##### TKT-12-03 · [FE] Panel de gestión de oferta para el reclutador
- **US:** US-12 — Gestión de oferta digital
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar en la React SPA el panel de gestión de oferta accesible desde el perfil del candidato (visible cuando la etapa es "Oferta"). Incluye: formulario de creación de oferta con campos de compensación y fecha de inicio, previsualización del PDF generado, estado actual de la oferta y historial de respuestas.

**Acceptance Criteria:**
- El panel de oferta solo es visible cuando la candidatura está en etapa "Oferta".
- El formulario de creación muestra una previsualización del PDF antes de enviar.
- El estado de la oferta (Enviada/Aceptada/Rechazada/Expirada) se actualiza en tiempo real.
- Si la oferta está "Aceptada", se muestra un banner de celebración y el estado del traspaso al HRIS.
- Si la oferta está "Rechazada", se muestra opción de "Reabrir proceso" (vuelve la candidatura a etapa Entrevista).

**Dependencias previas:** TKT-12-02.

---

##### TKT-12-04 · [FE] Vista de oferta y aceptación en el Portal del Candidato
- **US:** US-12 — Gestión de oferta digital
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar en el Portal del Candidato la vista de oferta (`/portal/ofertas/:token`). Muestra el PDF de la oferta embebido, los datos principales (título, compensación, fecha de inicio) y los botones "Aceptar oferta" y "Rechazar oferta" con confirmación modal.

**Acceptance Criteria:**
- El PDF se renderiza embebido en la página (sin necesidad de descarga).
- Los botones de respuesta solo están activos si la oferta está en estado "Enviada".
- "Aceptar oferta" muestra modal de confirmación: "¿Confirmas que aceptas esta oferta?".
- "Rechazar oferta" muestra modal con campo de texto opcional para indicar el motivo (no visible para el reclutador internamente, solo para métricas).
- Tras responder, la pantalla muestra mensaje de confirmación y el CTA del portal cambia a "Ver tu candidatura".

**Dependencias previas:** TKT-12-02, TKT-11-03.

---

##### TKT-12-05 · [QA] Tests de gestión de oferta
- **US:** US-12 — Gestión de oferta digital
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: creación y envío de oferta, aceptación y cambio de estado a "Contratado", rechazo con reapertura de proceso, expiración automática y validación de que no pueden crearse dos ofertas activas para la misma candidatura.

**Dependencias previas:** TKT-12-02.

---

#### US-14 · Dashboard de analítica e informes

---

##### TKT-14-01 · [BE] Queries y endpoints de métricas core del dashboard
- **US:** US-14 — Dashboard de analítica
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `GET /analytics/dashboard` que devuelva los 5 KPIs core: (1) tiempo medio hasta la contratación en días, (2) tasa de conversión por etapa (%), (3) top fuentes de candidatos por volumen, (4) tasa de aceptación de ofertas (%), (5) posiciones abiertas/cerradas en el período. El endpoint acepta filtros: `departamento`, `fecha_inicio`, `fecha_fin`. Incluir la métrica de precisión del cribado IA (R9): % de candidatos con `puntuacion_adecuacion_ia ≥ 70` que llegaron a etapa Oferta.

**Acceptance Criteria:**
- El endpoint devuelve los 5 KPIs en < 2 segundos con índices de BD adecuados.
- Los filtros son opcionales; sin filtros devuelve datos del último trimestre por defecto.
- Candidatos con `source = HEURISTIC_FALLBACK` son excluidos del cálculo de precisión de IA (R3).
- Si hay menos de 5 candidaturas para un KPI, el campo devuelve `null` con `{ insuficiente_datos: true }`.
- Los resultados se cachean 10 minutos en Redis para reducir carga de BD.

**Dependencias previas:** TKT-05-02, TKT-06-03, TKT-12-02.

---

##### TKT-14-02 · [BE] Endpoint de exportación a CSV
- **US:** US-14 — Dashboard de analítica
- **Tipo:** BE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Implementar `GET /analytics/dashboard/export?format=csv` que devuelva los mismos datos del dashboard en formato CSV con cabeceras descriptivas en castellano. Aplicar los mismos filtros que el endpoint de dashboard.

**Acceptance Criteria:**
- El CSV incluye una fila de cabecera con nombres descriptivos en castellano.
- Los datos numéricos usan coma como separador decimal (locale es-ES).
- El archivo se descarga con nombre `lti-analytics-{fecha}.csv`.
- El endpoint tiene rate limiting (máx. 10 exportaciones por usuario por hora).

**Dependencias previas:** TKT-14-01.

---

##### TKT-14-03 · [FE] Dashboard de analítica con widgets KPI y filtros
- **US:** US-14 — Dashboard de analítica
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en la React SPA la sección de analítica (`/analytics`) accesible para Reclutador, Responsable y Admin. Incluir 5 widgets KPI, filtros por departamento y rango de fechas, y botón de exportación a CSV.

**Acceptance Criteria:**
- Cada KPI se muestra en un widget con el valor, tendencia (vs período anterior) e ícono.
- Los widgets con datos insuficientes muestran el valor "–" con tooltip explicativo.
- El filtro de departamento carga los departamentos disponibles del tenant.
- El filtro de rango de fechas usa un datepicker con rango predefinido (Último mes / Último trimestre / Último año / Personalizado).
- El botón "Exportar CSV" descarga el archivo con los filtros activos.
- Los usuarios con rol Entrevistador ven el mensaje "No tienes acceso a esta sección" (R9 permisos).

**Dependencias previas:** TKT-14-01, TKT-14-02.

---

##### TKT-14-04 · [QA] Tests de analítica
- **US:** US-14 — Dashboard de analítica
- **Tipo:** QA · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Tests de integración para: cálculo correcto de los 5 KPIs con dataset de prueba conocido, exclusión de candidatos HEURISTIC_FALLBACK de la métrica de precisión IA, respuesta con datos insuficientes (< 5 candidaturas), exportación CSV con filtros, y acceso denegado para rol Entrevistador.

**Dependencias previas:** TKT-14-01, TKT-14-02.

---

### Sprint 5 — Funcionalidades avanzadas

#### US-08 · Asignación de evaluaciones en línea

---

##### TKT-08-01 · [SPIKE] Definir recálculo de score IA tras resultado de evaluación (R5)
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** SPIKE · **Complejidad:** S · **Story Points:** 2

**Descripción:**  
Resolver el riesgo R5: definir si el recálculo de `puntuacion_adecuacion_ia` tras recibir un RESULTADO_EVALUACION es (a) un nuevo trabajo en la cola del Motor de IA, (b) una actualización ponderada directa de la puntuación existente, o (c) un campo separado `puntuacion_con_evaluacion`. Documentar la decisión con la fórmula de ponderación.

**Criterios de aceptación:**
- Decisión documentada: opción recomendada (b) — actualización ponderada: `nueva_puntuacion = (puntuacion_ia * 0.6) + (puntuacion_evaluacion * 0.4)`.
- Los pesos (0.6 / 0.4) son configurables por requisición.
- El campo `puntuacion_adecuacion_ia` siempre refleja la puntuación más reciente y completa.

**Dependencias previas:** TKT-06-03.  
**Bloquea:** TKT-08-03.

---

##### TKT-08-02 · [BE] CRUD de plantillas de evaluación nativa
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar los endpoints para gestionar EVALUACION: `POST /evaluaciones`, `GET /evaluaciones`, `PUT /evaluaciones/:id`, `DELETE /evaluaciones/:id`. Una evaluación tiene título, tipo (Competencias, Técnica, Personalidad), descripción, límite de tiempo, puntuación de aprobado y proveedor (Nativa | HackerRank | Codility | TestGorilla). Para el MVP, implementar completamente las evaluaciones de tipo Nativa (preguntas abiertas y múltiple opción).

**Acceptance Criteria:**
- Solo Admin y Reclutador pueden gestionar plantillas de evaluación.
- Las evaluaciones de tipo "Nativa" permiten añadir preguntas con tipos: texto libre, opción múltiple, puntuación numérica.
- Las evaluaciones de proveedores externos tienen los campos `proveedor` e `id_externo` (la integración de API se implementa en TKT-08-04).
- Una evaluación con `activo = false` no puede asignarse a nuevos candidatos.

**Dependencias previas:** TKT-02-02.

---

##### TKT-08-03 · [BE] Endpoint de asignación de evaluación y gestión del ciclo de vida
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `POST /candidaturas/:id/evaluaciones` (asignar evaluación a candidato), `GET /candidaturas/:id/evaluaciones` (listar asignaciones), y el job de expiración de asignaciones. Al completarse una evaluación, actualizar la `puntuacion_adecuacion_ia` de la candidatura según la fórmula definida en TKT-08-01.

**Acceptance Criteria:**

```gherkin
# Happy path — asignación exitosa
Scenario: Reclutador asigna evaluación a candidato
  Given evaluación activa y candidatura activa
  When POST /candidaturas/:id/evaluaciones con { evaluacion_id, dias_expiracion: 5 }
  Then se crea ASIGNACION_EVALUACION con estado "Pendiente"
  And expira_en = now + 5 días
  And se genera url_acceso_candidato única
  And el candidato recibe email con la URL de acceso

# Happy path — resultado registrado y recálculo de score
Scenario: Candidato completa la evaluación nativa
  Given ASIGNACION_EVALUACION en estado "Pendiente"
  When el candidato completa la evaluación
  Then se crea RESULTADO_EVALUACION con puntuacion_bruta y aprobado
  And se recalcula puntuacion_adecuacion_ia = (ia_actual * 0.6) + (eval_normalizada * 0.4)
  And la ASIGNACION_EVALUACION cambia a estado "Completada"
  And el Reclutador recibe notificación con el resultado

# Edge case — evaluación expirada
Scenario: Asignación supera la fecha de expiración
  Given ASIGNACION_EVALUACION con expira_en en el pasado y estado "Pendiente"
  When el job de expiración ejecuta
  Then estado → "Expirada"
  And el Reclutador recibe notificación "La evaluación de [candidato] ha expirado"
```

**Dependencias previas:** TKT-08-01, TKT-08-02, TKT-06-03.

---

##### TKT-08-04 · [BE] Integración con proveedor externo HackerRank (webhook)
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** BE · **Complejidad:** L · **Story Points:** 8

**Descripción:**  
Implementar la integración bidireccional con HackerRank: (1) creación de invitación de prueba via API de HackerRank al asignar la evaluación, (2) recepción del webhook de resultado cuando el candidato completa la prueba, (3) transformación del resultado al modelo RESULTADO_EVALUACION de LTI.

**Acceptance Criteria:**
- Al asignar una evaluación con proveedor "HackerRank", se llama a la API de HackerRank para crear la invitación y obtener la URL de acceso del candidato.
- El endpoint de webhook `POST /webhooks/hackerrank/resultado` valida la firma HMAC del payload.
- Al recibir el webhook, se crea RESULTADO_EVALUACION con los datos normalizados.
- Si el webhook falla 3 veces, el trabajo va a dead-letter queue para revisión manual.
- Todos los errores de la integración se registran con suficiente contexto para depuración.

**Dependencias previas:** TKT-08-03.

---

##### TKT-08-05 · [FE] Panel de evaluaciones en el perfil del candidato
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Implementar en la React SPA la sección de evaluaciones dentro del perfil del candidato. Incluir: listado de evaluaciones asignadas con estado y fecha de expiración, botón de asignar nueva evaluación (con selector de plantillas), resultado de evaluaciones completadas (puntuación y badge aprobado/no aprobado).

**Acceptance Criteria:**
- El listado de evaluaciones muestra: nombre, tipo, estado (Pendiente/Completada/Expirada), fecha de expiración y resultado si está completada.
- El badge "Aprobado/No aprobado" usa colores verde/rojo según el campo `aprobado` del resultado.
- El botón "Asignar evaluación" abre un modal con el listado de plantillas activas del tenant filtrable por tipo.
- Si ya hay una evaluación activa del mismo tipo, muestra aviso pero no bloquea la asignación.

**Dependencias previas:** TKT-08-03.

---

##### TKT-08-06 · [FE] Vista de evaluación nativa en el Portal del Candidato
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** FE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar en el Portal del Candidato la vista de realización de evaluación nativa (`/portal/evaluaciones/:token`). Incluye: temporizador regresivo si tiene límite de tiempo, presentación de preguntas una a una o en listado, y pantalla de confirmación de envío.

**Acceptance Criteria:**
- La evaluación solo es accesible con el `url_acceso_candidato` único.
- Si tiene límite de tiempo, muestra un temporizador visible y envía automáticamente al llegar a 0.
- Las respuestas se guardan automáticamente cada 30 segundos para evitar pérdida de datos.
- Al completar y enviar, muestra "Evaluación enviada correctamente. El equipo revisará tus resultados."
- Si la evaluación ha expirado, muestra "Este enlace ha expirado" y no permite acceder.

**Dependencias previas:** TKT-08-03, TKT-11-03.

---

##### TKT-08-07 · [QA] Tests de asignación y resultado de evaluaciones
- **US:** US-08 — Asignación de evaluaciones
- **Tipo:** QA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Tests de integración para: asignación exitosa con generación de URL, recepción de webhook de HackerRank con verificación de firma, recálculo de puntuación IA tras resultado, expiración automática, y acceso denegado a evaluación expirada.

**Dependencias previas:** TKT-08-03, TKT-08-04.

---

#### US-13 · Constructor de automatizaciones no-code

---

##### TKT-13-01 · [SPIKE] Definir event catalog del sistema (R8)
- **US:** US-13 — Constructor de automatizaciones
- **Tipo:** SPIKE · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Resolver el riesgo R8: documentar el catálogo de eventos del sistema que el Motor de Automatización puede escuchar. Para cada evento: nombre, payload, versión y servicio productor. Eventos mínimos para el MVP de automatizaciones: `candidatura_recibida`, `cribado_completado`, `evaluacion_completada`, `scorecard_enviada`, `entrevista_confirmada`, `decision_tomada`, `oferta_enviada`, `oferta_aceptada`.

**Criterios de aceptación:**
- Event catalog publicado como documento técnico y accesible al equipo.
- Cada evento tiene schema JSON definido y validado.
- El bus de eventos (Redis Streams) implementa el schema definido en todos los productores de los sprints anteriores.

**Dependencias previas:** TKT-06-03, TKT-08-03, TKT-10-03, TKT-12-02.  
**Bloquea:** TKT-13-02.

---

##### TKT-13-02 · [BE] Motor de evaluación de reglas de automatización
- **US:** US-13 — Constructor de automatizaciones
- **Tipo:** BE · **Complejidad:** L · **Story Points:** 13

**Descripción:**  
Implementar el Motor de Automatización (servicio Node.js) que: (1) consume eventos del bus, (2) evalúa las REGLA_AUTOMATIZACION activas del tenant, (3) ejecuta las acciones configuradas. Acciones disponibles en MVP: `mover_candidatura_a_etapa`, `enviar_notificacion_email`, `notificar_usuario_interno`. Incluir detección de bucles (máx. 3 iteraciones de la misma regla por evento) y logging de ejecuciones.

**Acceptance Criteria:**

```gherkin
# Happy path — ejecución de regla
Scenario: Regla mueve candidato automáticamente al superar evaluación
  Given REGLA: CUANDO "evaluacion_completada" Y "puntuacion_normalizada >= 70"
               ENTONCES "mover_candidatura_a_etapa(Entrevista técnica)"
  When se publica evento "evaluacion_completada" con puntuacion = 85
  Then el motor evalúa la condición (85 >= 70) → verdadera
  And ejecuta la acción: candidatura movida a etapa "Entrevista técnica"
  And registra la ejecución en el log de la regla con timestamp

# Error — regla con etapa destino inexistente
Scenario: La etapa destino fue eliminada
  Given la regla apunta a etapa_id que no existe en BD
  When el motor intenta ejecutar la acción
  Then registra error "Etapa destino no encontrada" en el log de la regla
  And marca la regla con estado "Error_Configuracion"
  And NO modifica la candidatura

# Edge case — detección de bucle
Scenario: Dos reglas se activan mutuamente
  Given regla A: "si etapa=X → mover a Y" y regla B: "si etapa=Y → mover a X"
  When se activa la regla A por primera vez
  Then el motor permite la ejecución
  When la regla B se activa y vuelve a activar la regla A
  Then en la 3ª iteración el motor detecta el ciclo
  And detiene la ejecución con evento "automation_loop_detected"
  And notifica al Admin del tenant
```

**Dependencias previas:** TKT-13-01, TKT-06-02.

---

##### TKT-13-03 · [BE] Endpoints CRUD de reglas de automatización
- **US:** US-13 — Constructor de automatizaciones
- **Tipo:** BE · **Complejidad:** M · **Story Points:** 5

**Descripción:**  
Implementar `POST /requisiciones/:id/automatizaciones`, `GET /requisiciones/:id/automatizaciones`, `PUT /automatizaciones/:id`, `PATCH /automatizaciones/:id/activar-desactivar`, `DELETE /automatizaciones/:id`. Una regla tiene nombre, `condiciones_disparo` (JSON con evento + condiciones) y `acciones` (JSON con lista de acciones).

**Acceptance Criteria:**
- Solo Reclutador y Admin pueden gestionar reglas.
- Al crear una regla, el backend valida que los eventos referenciados existen en el event catalog.
- Al crear una regla, el backend valida que las etapas destino existen en la requisición.
- Una regla desactivada (`activo = false`) no se evalúa por el motor pero se conserva su configuración.
- No se pueden crear más de 20 reglas activas por requisición (límite MVP).

**Dependencias previas:** TKT-13-02.

---

##### TKT-13-04 · [FE] Constructor visual de reglas de automatización
- **US:** US-13 — Constructor de automatizaciones
- **Tipo:** FE · **Complejidad:** L · **Story Points:** 8

**Descripción:**  
Implementar en la React SPA el constructor de reglas de automatización (`/requisiciones/:id/automatizaciones`). Incluir: listado de reglas activas e inactivas, formulario de creación de regla con selector de evento disparador, condiciones adicionales (campo + operador + valor) y selector de acciones. Mostrar el estado de ejecución de cada regla (última ejecución, errores de configuración).

**Acceptance Criteria:**
- El formulario de creación tiene tres secciones: "Cuando ocurra" (evento), "Si se cumple" (condiciones opcionales) y "Entonces" (acciones).
- Los selectores de evento, campo de condición y acción son dropdowns dinámicos que cargan las opciones disponibles.
- Las reglas con estado "Error_Configuracion" se muestran con badge rojo y el mensaje de error.
- El toggle de activar/desactivar regla es visible en el listado sin necesidad de abrir el editor.
- Al guardar una regla, se muestra el resultado de la validación del backend (errores de configuración inline).

**Dependencias previas:** TKT-13-03.

---

##### TKT-13-05 · [QA] Tests del motor de automatización
- **US:** US-13 — Constructor de automatizaciones
- **Tipo:** QA · **Complejidad:** M · **Story Points:** 3

**Descripción:**  
Tests de integración para: evaluación de regla con condición verdadera y ejecución de acción, evaluación de regla con condición falsa (no ejecuta), regla con etapa destino inexistente (error de configuración), detección de bucle entre dos reglas, y verificación de que reglas inactivas no se evalúan.

**Dependencias previas:** TKT-13-02, TKT-13-03.

---

### Tabla de roadmap con desglose de tickets

| Sprint | US | Tickets | SP totales US | Objetivo del sprint |
|---|---|---|:---:|---|
| **Sprint 0** | **US-01** | TKT-01-01 (2) · TKT-01-02 (3) · TKT-01-03 (3) · TKT-01-04 (2) · TKT-01-05 (3) · TKT-01-06 (2) | **15 SP** | Fundaciones: auth y acceso |
| **Sprint 0** | **US-02** | TKT-02-01 (3) · TKT-02-02 (3) · TKT-02-03 (5) · TKT-02-04 (5) · TKT-02-05 (2) | **18 SP** | Fundaciones: tenant y usuarios |
| | | | **Sprint 0 total: 33 SP** | |
| **Sprint 1** | **US-03** | TKT-03-01 (1) · TKT-03-02 (5) · TKT-03-03 (3) · TKT-03-04 (5) · TKT-03-05 (2) · TKT-03-06 (2) | **18 SP** | Creación de requisición |
| **Sprint 1** | **US-05** | TKT-05-01 (2) · TKT-05-02 (5) · TKT-05-03 (2) · TKT-05-04 (5) · TKT-05-05 (2) | **16 SP** | Envío de candidatura |
| **Sprint 1** | **US-04** | TKT-04-01 (2) · TKT-04-02 (5) · TKT-04-03 (3) · TKT-04-04 (3) · TKT-04-05 (2) · TKT-04-06 (2) | **17 SP** | Publicación multicanal |
| | | | **Sprint 1 total: 51 SP** | |
| **Sprint 2** | **US-06** | TKT-06-01 (3) · TKT-06-02 (3) · TKT-06-03 (13) · TKT-06-04 (5) · TKT-06-05 (3) | **27 SP** | Motor IA de cribado |
| **Sprint 2** | **US-11** | TKT-11-01 (3) · TKT-11-02 (3) · TKT-11-03 (5) · TKT-11-04 (2) | **13 SP** | Portal del candidato |
| | | | **Sprint 2 total: 40 SP** | |
| **Sprint 3** | **US-10** | TKT-10-01 (2) · TKT-10-02 (2) · TKT-10-03 (5) · TKT-10-04 (5) · TKT-10-05 (2) | **16 SP** | Scorecards estructuradas |
| **Sprint 3** | **US-09** | TKT-09-01 (3) · TKT-09-02 (8) · TKT-09-03 (8) · TKT-09-04 (2) · TKT-09-05 (5) · TKT-09-06 (2) · TKT-09-07 (3) | **31 SP** | Programación de entrevistas |
| **Sprint 3** | **US-07** | TKT-07-01 (5) · TKT-07-02 (5) · TKT-07-03 (2) · TKT-07-04 (8) · TKT-07-05 (3) | **23 SP** | Espacio colaborativo |
| | | | **Sprint 3 total: 70 SP** | |
| **Sprint 4** | **US-12** | TKT-12-01 (3) · TKT-12-02 (5) · TKT-12-03 (3) · TKT-12-04 (3) · TKT-12-05 (2) | **16 SP** | Oferta digital |
| **Sprint 4** | **US-14** | TKT-14-01 (5) · TKT-14-02 (2) · TKT-14-03 (5) · TKT-14-04 (2) | **14 SP** | Dashboard de analítica |
| | | | **Sprint 4 total: 30 SP** | |
| **Sprint 5** | **US-08** | TKT-08-01 (2) · TKT-08-02 (5) · TKT-08-03 (5) · TKT-08-04 (8) · TKT-08-05 (3) · TKT-08-06 (5) · TKT-08-07 (3) | **31 SP** | Evaluaciones en línea |
| **Sprint 5** | **US-13** | TKT-13-01 (3) · TKT-13-02 (13) · TKT-13-03 (5) · TKT-13-04 (8) · TKT-13-05 (3) | **32 SP** | Motor de automatizaciones |
| | | | **Sprint 5 total: 63 SP** | |
| | | | **TOTAL MVP: 287 SP** | |

> **Nota de planificación:** Sprint 3 acumula 70 SP debido a la complejidad de las integraciones de calendario (US-09). Se recomienda evaluar en la reunión de refinamiento si US-09 se distribuye en dos sub-sprints o si parte del trabajo de TKT-09-02 y TKT-09-03 puede iniciarse en paralelo durante el Sprint 2 como trabajo preparatorio.

---

*Fin del documento — LTI ATS User Stories MVP v1.0*
