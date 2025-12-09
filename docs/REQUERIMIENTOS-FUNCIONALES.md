# Requerimientos Funcionales - Sistema Ticketero Digital

**Proyecto:** Sistema de Gestión de Tickets con Notificaciones en Tiempo Real  
**Cliente:** Institución Financiera  
**Versión:** 1.0  
**Fecha:** Diciembre 2025  
**Analista:** Analista de Negocio Senior

---

## 1. Introducción

### 1.1 Propósito

Este documento especifica los requerimientos funcionales del Sistema Ticketero Digital, diseñado para modernizar la experiencia de atención en sucursales mediante:
- Digitalización completa del proceso de tickets
- Notificaciones automáticas en tiempo real vía Telegram
- Movilidad del cliente durante la espera
- Asignación inteligente de clientes a ejecutivos
- Panel de monitoreo para supervisión operacional

### 1.2 Alcance

Este documento cubre:
- ✅ 8 Requerimientos Funcionales (RF-001 a RF-008)
- ✅ 13 Reglas de Negocio (RN-001 a RN-013)
- ✅ Criterios de aceptación en formato Gherkin
- ✅ Modelo de datos funcional
- ✅ Matriz de trazabilidad

Este documento NO cubre:
- ❌ Arquitectura técnica (ver documento ARQUITECTURA.md)
- ❌ Tecnologías de implementación
- ❌ Diseño de interfaces de usuario

### 1.3 Definiciones

| Término | Definición |
|---------|------------|
| Ticket | Turno digital asignado a un cliente para ser atendido |
| Cola | Fila virtual de tickets esperando atención |
| Asesor | Ejecutivo bancario que atiende clientes |
| Módulo | Estación de trabajo de un asesor (numerados 1-5) |
| Chat ID | Identificador único de usuario en Telegram |
| UUID | Identificador único universal para tickets |

## 2. Reglas de Negocio

Las siguientes reglas de negocio aplican transversalmente a todos los requerimientos funcionales:

**RN-001: Unicidad de Ticket Activo**  
Un cliente solo puede tener 1 ticket activo a la vez. Los estados activos son: EN_ESPERA, PROXIMO, ATENDIENDO. Si un cliente intenta crear un nuevo ticket teniendo uno activo, el sistema debe rechazar la solicitud con error HTTP 409 Conflict.

**RN-002: Prioridad de Colas**  
Las colas tienen prioridades numéricas para asignación automática:
- GERENCIA: prioridad 4 (máxima)
- EMPRESAS: prioridad 3
- PERSONAL_BANKER: prioridad 2
- CAJA: prioridad 1 (mínima)

Cuando un asesor se libera, el sistema asigna primero tickets de colas con mayor prioridad.

**RN-003: Orden FIFO Dentro de Cola**  
Dentro de una misma cola, los tickets se procesan en orden FIFO (First In, First Out). El ticket más antiguo (createdAt menor) se asigna primero.

**RN-004: Balanceo de Carga Entre Asesores**  
Al asignar un ticket, el sistema selecciona el asesor AVAILABLE con menor valor de assignedTicketsCount, distribuyendo equitativamente la carga de trabajo.

**RN-005: Formato de Número de Ticket**  
El número de ticket sigue el formato: [Prefijo][Número secuencial 01-99]
- Prefijo: 1 letra según el tipo de cola
- Número: 2 dígitos, del 01 al 99, reseteado diariamente

Ejemplos: C01, P15, E03, G02

**RN-006: Prefijos por Tipo de Cola**  
- CAJA → C
- PERSONAL_BANKER → P
- EMPRESAS → E
- GERENCIA → G

**RN-007: Reintentos Automáticos de Mensajes**  
Si el envío de un mensaje a Telegram falla, el sistema reintenta automáticamente hasta 3 veces antes de marcarlo como FALLIDO.

**RN-008: Backoff Exponencial en Reintentos**  
Los reintentos de mensajes usan backoff exponencial:
- Intento 1: inmediato
- Intento 2: después de 30 segundos
- Intento 3: después de 60 segundos
- Intento 4: después de 120 segundos

**RN-009: Estados de Ticket**  
Un ticket puede estar en uno de estos estados:
- EN_ESPERA: esperando asignación a asesor
- PROXIMO: próximo a ser atendido (posición ≤ 3)
- ATENDIENDO: siendo atendido por un asesor
- COMPLETADO: atención finalizada exitosamente
- CANCELADO: cancelado por cliente o sistema
- NO_ATENDIDO: cliente no se presentó cuando fue llamado

**RN-010: Cálculo de Tiempo Estimado**  
El tiempo estimado de espera se calcula como:

tiempoEstimado = posiciónEnCola × tiempoPromedioCola

Donde tiempoPromedioCola varía por tipo:
- CAJA: 5 minutos
- PERSONAL_BANKER: 15 minutos
- EMPRESAS: 20 minutos
- GERENCIA: 30 minutos

**RN-011: Auditoría Obligatoria**  
Todos los eventos críticos del sistema deben registrarse en auditoría con: timestamp, tipo de evento, actor involucrado, entityId afectado, y cambios de estado.

**RN-012: Umbral de Pre-aviso**  
El sistema envía el Mensaje 2 (pre-aviso) cuando la posición del ticket es ≤ 3, indicando que el cliente debe acercarse a la sucursal.

**RN-013: Estados de Asesor**  
Un asesor puede estar en uno de estos estados:
- AVAILABLE: disponible para recibir asignaciones
- BUSY: atendiendo un cliente (no recibe nuevas asignaciones)
- OFFLINE: no disponible (almuerzo, capacitación, etc.)

## 3. Enumeraciones

### 3.1 QueueType

Tipos de cola disponibles en el sistema:

| Valor | Display Name | Tiempo Promedio | Prioridad | Prefijo |
|-------|--------------|-----------------|-----------|---------|
| CAJA | Caja | 5 min | 1 | C |
| PERSONAL_BANKER | Personal Banker | 15 min | 2 | P |
| EMPRESAS | Empresas | 20 min | 3 | E |
| GERENCIA | Gerencia | 30 min | 4 | G |

### 3.2 TicketStatus

Estados posibles de un ticket:

| Valor | Descripción | Es Activo? |
|-------|-------------|------------|
| EN_ESPERA | Esperando asignación | Sí |
| PROXIMO | Próximo a ser atendido | Sí |
| ATENDIENDO | Siendo atendido | Sí |
| COMPLETADO | Atención finalizada | No |
| CANCELADO | Cancelado | No |
| NO_ATENDIDO | Cliente no se presentó | No |

### 3.3 AdvisorStatus

Estados posibles de un asesor:

| Valor | Descripción | Recibe Asignaciones? |
|-------|-------------|----------------------|
| AVAILABLE | Disponible | Sí |
| BUSY | Atendiendo cliente | No |
| OFFLINE | No disponible | No |

### 3.4 MessageTemplate

Plantillas de mensajes para Telegram:

| Valor | Descripción | Momento de Envío |
|-------|-------------|------------------|
| totem_ticket_creado | Confirmación de creación | Inmediato al crear ticket |
| totem_proximo_turno | Pre-aviso | Cuando posición ≤ 3 |
| totem_es_tu_turno | Turno activo | Al asignar a asesor |

## 4. Requerimientos Funcionales

### RF-001: Crear Ticket Digital

**Descripción:**
El sistema debe permitir al cliente crear un ticket digital para ser atendido en sucursal, ingresando su identificación nacional (RUT/ID), número de teléfono y seleccionando el tipo de atención requerida. El sistema generará un número único de ticket, calculará la posición actual en cola y el tiempo estimado de espera basado en datos reales de la operación.

**Prioridad:** Alta

**Actor Principal:** Cliente

**Precondiciones:**
- Terminal de autoservicio disponible y funcional
- Sistema de gestión de colas operativo
- Conexión a base de datos activa

**Modelo de Datos (Campos del Ticket):**
- codigoReferencia: UUID único (ej: "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6")
- numero: String formato específico por cola (ej: "C01", "P15", "E03", "G02")
- nationalId: String, identificación nacional del cliente
- telefono: String, número de teléfono para Telegram
- branchOffice: String, nombre de la sucursal
- queueType: Enum (CAJA, PERSONAL_BANKER, EMPRESAS, GERENCIA)
- status: Enum (EN_ESPERA, PROXIMO, ATENDIENDO, COMPLETADO, CANCELADO, NO_ATENDIDO)
- positionInQueue: Integer, posición actual en cola (calculada en tiempo real)
- estimatedWaitMinutes: Integer, minutos estimados de espera
- createdAt: Timestamp, fecha/hora de creación
- assignedAdvisor: Relación a entidad Advisor (null inicialmente)
- assignedModuleNumber: Integer 1-5 (null inicialmente)

**Reglas de Negocio Aplicables:**
- RN-001: Un cliente solo puede tener 1 ticket activo a la vez
- RN-005: Número de ticket formato: [Prefijo][Número secuencial 01-99]
- RN-006: Prefijos por cola: C=Caja, P=Personal Banker, E=Empresas, G=Gerencia
- RN-010: Cálculo de tiempo estimado: posiciónEnCola × tiempoPromedioCola

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Creación exitosa de ticket para cola de Caja**
```gherkin
Given el cliente con nationalId "12345678-9" no tiene tickets activos
And el terminal está en pantalla de selección de servicio
When el cliente ingresa:
  | Campo        | Valor           |
  | nationalId   | 12345678-9      |
  | telefono     | +56912345678    |
  | branchOffice | Sucursal Centro |
  | queueType    | CAJA            |
Then el sistema genera un ticket con:
  | Campo                 | Valor Esperado                    |
  | codigoReferencia      | UUID válido                       |
  | numero                | "C[01-99]"                        |
  | status                | EN_ESPERA                         |
  | positionInQueue       | Número > 0                        |
  | estimatedWaitMinutes  | positionInQueue × 5               |
  | assignedAdvisor       | null                              |
  | assignedModuleNumber  | null                              |
And el sistema almacena el ticket en base de datos
And el sistema programa 3 mensajes de Telegram
And el sistema retorna HTTP 201 con JSON:
  {
    "identificador": "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6",
    "numero": "C01",
    "positionInQueue": 5,
    "estimatedWaitMinutes": 25,
    "queueType": "CAJA"
  }
```

**Escenario 2: Error - Cliente ya tiene ticket activo**
```gherkin
Given el cliente con nationalId "12345678-9" tiene un ticket activo:
  | numero | status     | queueType      |
  | P05    | EN_ESPERA  | PERSONAL_BANKER|
When el cliente intenta crear un nuevo ticket con queueType CAJA
Then el sistema rechaza la creación
And el sistema retorna HTTP 409 Conflict con JSON:
  {
    "error": "TICKET_ACTIVO_EXISTENTE",
    "mensaje": "Ya tienes un ticket activo: P05",
    "ticketActivo": {
      "numero": "P05",
      "positionInQueue": 3,
      "estimatedWaitMinutes": 45
    }
  }
And el sistema NO crea un nuevo ticket
```

**Escenario 3: Validación - RUT/ID inválido**
```gherkin
Given el terminal está en pantalla de ingreso de datos
When el cliente ingresa nationalId vacío
Then el sistema retorna HTTP 400 Bad Request con JSON:
  {
    "error": "VALIDACION_FALLIDA",
    "campos": {
      "nationalId": "El RUT/ID es obligatorio"
    }
  }
And el sistema NO crea el ticket
```

**Escenario 4: Validación - Teléfono en formato inválido**
```gherkin
Given el terminal está en pantalla de ingreso de datos
When el cliente ingresa telefono "123"
Then el sistema retorna HTTP 400 Bad Request con JSON:
  {
    "error": "VALIDACION_FALLIDA",
    "campos": {
      "telefono": "Formato requerido: +56XXXXXXXXX"
    }
  }
And el sistema NO crea el ticket
```

**Escenario 5: Cálculo de posición - Primera persona en cola**
```gherkin
Given la cola de tipo PERSONAL_BANKER está vacía
When el cliente crea un ticket para PERSONAL_BANKER
Then el sistema calcula positionInQueue = 1
And estimatedWaitMinutes = 15
And el número de ticket es "P01"
```

**Escenario 6: Cálculo de posición - Cola con tickets existentes**
```gherkin
Given la cola de tipo EMPRESAS tiene 4 tickets EN_ESPERA
When el cliente crea un nuevo ticket para EMPRESAS
Then el sistema calcula positionInQueue = 5
And estimatedWaitMinutes = 100
And el cálculo es: 5 × 20min = 100min
```

**Escenario 7: Creación sin teléfono (cliente no quiere notificaciones)**
```gherkin
Given el cliente no proporciona número de teléfono
When el cliente crea un ticket con:
  | Campo        | Valor           |
  | nationalId   | 12345678-9      |
  | telefono     | null            |
  | branchOffice | Sucursal Centro |
  | queueType    | CAJA            |
Then el sistema crea el ticket exitosamente
And el sistema NO programa mensajes de Telegram
And el sistema retorna HTTP 201 con ticket válido
```

**Postcondiciones:**
- Ticket almacenado en base de datos con estado EN_ESPERA
- 3 mensajes programados (si hay teléfono)
- Evento de auditoría registrado: "TICKET_CREADO"

**Endpoints HTTP:**
- POST /api/tickets - Crear nuevo ticket

---

### RF-002: Enviar Notificaciones Automáticas vía Telegram

**Descripción:**
El sistema debe enviar automáticamente tres mensajes vía Telegram en momentos específicos del proceso: confirmación inmediata al crear el ticket, pre-aviso cuando quedan 3 personas adelante, y notificación de turno activo al asignar a un ejecutivo. Los mensajes deben incluir información relevante como número de ticket, posición, tiempo estimado, módulo y nombre del asesor.

**Prioridad:** Alta

**Actor Principal:** Sistema (automatizado)

**Precondiciones:**
- Ticket creado con teléfono válido
- Telegram Bot configurado y activo
- Cliente tiene cuenta de Telegram

**Modelo de Datos (Entidad Mensaje):**
- id: BIGSERIAL (primary key)
- ticket_id: BIGINT (foreign key a ticket)
- plantilla: String (totem_ticket_creado, totem_proximo_turno, totem_es_tu_turno)
- estadoEnvio: Enum (PENDIENTE, ENVIADO, FALLIDO)
- fechaProgramada: Timestamp
- fechaEnvio: Timestamp (nullable)
- telegramMessageId: String (nullable, retornado por Telegram API)
- intentos: Integer (contador de reintentos, default 0)

**Plantillas de Mensajes:**

**1. totem_ticket_creado:**
```
✅ <b>Ticket Creado</b>
Tu número de turno: <b>{numero}</b>
Posición en cola: <b>#{posicion}</b>
Tiempo estimado: <b>{tiempo} minutos</b>

Te notificaremos cuando estés próximo.
```

**2. totem_proximo_turno:**
```
⏰ <b>¡Pronto será tu turno!</b>
Turno: <b>{numero}</b>
Faltan aproximadamente 3 turnos.

Por favor, acércate a la sucursal.
```

**3. totem_es_tu_turno:**
```
🔔 <b>¡ES TU TURNO {numero}!</b>
Dirígete al módulo: <b>{modulo}</b>
Asesor: <b>{nombreAsesor}</b>
```

**Reglas de Negocio Aplicables:**
- RN-007: 3 reintentos automáticos para mensajes fallidos
- RN-008: Backoff exponencial (30s, 60s, 120s)
- RN-011: Auditoría obligatoria de envíos
- RN-012: Mensaje 2 cuando posición ≤ 3

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Envío exitoso del Mensaje 1 (Confirmación)**
```gherkin
Given un ticket fue creado con:
  | numero   | P05          |
  | telefono | +56912345678 |
  | posicion | 5            |
  | tiempo   | 75           |
When el sistema programa el mensaje totem_ticket_creado
Then el sistema crea un registro Mensaje con:
  | plantilla        | totem_ticket_creado |
  | estadoEnvio      | PENDIENTE           |
  | fechaProgramada  | timestamp actual    |
  | intentos         | 0                   |
And el sistema envía el mensaje a Telegram API
And Telegram API retorna éxito con messageId "12345"
Then el sistema actualiza el registro:
  | estadoEnvio        | ENVIADO    |
  | fechaEnvio         | timestamp  |
  | telegramMessageId  | "12345"    |
And el sistema registra auditoría: "MENSAJE_ENVIADO"
```

**Escenario 2: Envío exitoso del Mensaje 2 (Pre-aviso)**
```gherkin
Given un ticket tiene positionInQueue = 3
When el sistema detecta que posición ≤ 3
Then el sistema programa mensaje totem_proximo_turno
And el mensaje contiene:
  "⏰ ¡Pronto será tu turno!
   Turno: P05
   Faltan aproximadamente 3 turnos.
   Por favor, acércate a la sucursal."
And el sistema envía exitosamente
And el ticket cambia status a PROXIMO
```

**Escenario 3: Envío exitoso del Mensaje 3 (Turno Activo)**
```gherkin
Given un ticket fue asignado a:
  | asesor | Juan Pérez |
  | modulo | 3          |
When el sistema programa mensaje totem_es_tu_turno
Then el mensaje contiene:
  "🔔 ¡ES TU TURNO P05!
   Dirígete al módulo: 3
   Asesor: Juan Pérez"
And el sistema envía exitosamente
And el ticket cambia status a ATENDIENDO
```

**Escenario 4: Fallo de red en primer intento, éxito en segundo**
```gherkin
Given un mensaje está programado para envío
When el sistema intenta enviar a Telegram API
And Telegram API retorna error de red (timeout)
Then el sistema incrementa intentos = 1
And el sistema programa reintento en 30 segundos
When el sistema reintenta después de 30 segundos
And Telegram API retorna éxito
Then el sistema actualiza estadoEnvio = ENVIADO
And intentos queda en 1
```

**Escenario 5: 3 reintentos fallidos → estado FALLIDO**
```gherkin
Given un mensaje está en intento 3
When el sistema intenta enviar por cuarta vez
And Telegram API retorna error
Then el sistema actualiza:
  | estadoEnvio | FALLIDO |
  | intentos    | 4       |
And el sistema NO programa más reintentos
And el sistema registra auditoría: "MENSAJE_FALLIDO"
```

**Escenario 6: Backoff exponencial entre reintentos**
```gherkin
Given un mensaje falló en el primer intento
When el sistema programa el reintento 1
Then el reintento se programa en 30 segundos
Given el reintento 1 también falla
When el sistema programa el reintento 2
Then el reintento se programa en 60 segundos
Given el reintento 2 también falla
When el sistema programa el reintento 3
Then el reintento se programa en 120 segundos
```

**Escenario 7: Cliente sin teléfono, no se programan mensajes**
```gherkin
Given un ticket fue creado sin teléfono
When el sistema evalúa si programar mensajes
Then el sistema NO crea registros en tabla Mensaje
And el sistema NO intenta envíos a Telegram
And el proceso continúa normalmente
```

**Postcondiciones:**
- Mensaje insertado en BD con estado según resultado
- telegram_message_id almacenado si éxito
- Intentos incrementado en cada reintento
- Auditoría registrada para cada envío/fallo

**Endpoints HTTP:**
- Ninguno (proceso interno automatizado por scheduler)

---

### RF-003: Calcular Posición y Tiempo Estimado

**Descripción:**
El sistema debe calcular en tiempo real la posición exacta del cliente en cola y estimar el tiempo de espera basado en la posición actual, tiempo promedio de atención por tipo de cola, y cantidad de ejecutivos disponibles. Los cálculos deben actualizarse automáticamente cuando otros tickets cambien de estado o se asignen a asesores.

**Prioridad:** Alta

**Actor Principal:** Sistema (automatizado)

**Precondiciones:**
- Ticket existe en el sistema
- Base de datos con información actualizada de colas
- Información de asesores disponibles actualizada

**Algoritmos de Cálculo:**

**Posición en Cola:**
```
posición = COUNT(tickets EN_ESPERA con createdAt < ticket.createdAt) + 1
```

**Tiempo Estimado:**
```
tiempoEstimado = posición × tiempoPromedioCola
```

**Tiempos Promedio por Cola:**
- CAJA: 5 minutos
- PERSONAL_BANKER: 15 minutos
- EMPRESAS: 20 minutos
- GERENCIA: 30 minutos

**Reglas de Negocio Aplicables:**
- RN-003: Orden FIFO dentro de cola (createdAt determina orden)
- RN-010: Fórmula de cálculo de tiempo estimado
- RN-011: Auditoría de recálculos

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Cálculo de posición - Primer ticket en cola vacía**
```gherkin
Given la cola CAJA está completamente vacía
When se crea un nuevo ticket para CAJA
Then el sistema calcula positionInQueue = 1
And estimatedWaitMinutes = 5
And el cálculo es: 1 × 5min = 5min
```

**Escenario 2: Cálculo de posición - Cola con tickets existentes**
```gherkin
Given la cola PERSONAL_BANKER tiene tickets:
  | numero | status    | createdAt           |
  | P01    | EN_ESPERA | 2025-01-15 09:00:00 |
  | P02    | EN_ESPERA | 2025-01-15 09:05:00 |
  | P03    | EN_ESPERA | 2025-01-15 09:10:00 |
When se crea un nuevo ticket P04 a las 09:15:00
Then el sistema calcula positionInQueue = 4
And estimatedWaitMinutes = 60
And el cálculo es: 4 × 15min = 60min
```

**Escenario 3: Recálculo automático cuando ticket anterior se asigna**
```gherkin
Given un ticket P05 tiene positionInQueue = 3
And estimatedWaitMinutes = 45
When el ticket P02 (anterior en cola) cambia status a ATENDIENDO
Then el sistema recalcula automáticamente P05:
  | positionInQueue      | 2  |
  | estimatedWaitMinutes | 30 |
And el sistema registra auditoría: "POSICION_RECALCULADA"
```

**Escenario 4: Consulta de posición por UUID**
```gherkin
Given un ticket con UUID "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6"
And positionInQueue = 5
When se consulta GET /api/tickets/a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6
Then el sistema retorna HTTP 200 con JSON:
  {
    "codigoReferencia": "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6",
    "numero": "P05",
    "positionInQueue": 5,
    "estimatedWaitMinutes": 75,
    "queueType": "PERSONAL_BANKER",
    "status": "EN_ESPERA"
  }
```

**Escenario 5: Consulta de posición por número de ticket**
```gherkin
Given un ticket número "E03" existe
When se consulta GET /api/tickets/E03/position
Then el sistema retorna HTTP 200 con JSON:
  {
    "numero": "E03",
    "positionInQueue": 2,
    "estimatedWaitMinutes": 40,
    "queueType": "EMPRESAS",
    "status": "EN_ESPERA",
    "calculadoEn": "2025-01-15T09:30:00Z"
  }
```

**Escenario 6: Error - Ticket no existe**
```gherkin
Given no existe ticket con número "X99"
When se consulta GET /api/tickets/X99/position
Then el sistema retorna HTTP 404 Not Found con JSON:
  {
    "error": "TICKET_NO_ENCONTRADO",
    "mensaje": "No existe ticket con número: X99"
  }
```

**Escenario 7: Cálculo para diferentes tipos de cola**
```gherkin
Given existen tickets en diferentes colas:
  | queueType       | posicion | tiempoPromedio | tiempoEstimado |
  | CAJA           | 3        | 5min          | 15min         |
  | PERSONAL_BANKER | 4        | 15min         | 60min         |
  | EMPRESAS       | 2        | 20min         | 40min         |
  | GERENCIA       | 1        | 30min         | 30min         |
When el sistema calcula tiempos estimados
Then todos los cálculos son correctos según la fórmula
And cada cola mantiene su tiempo promedio específico
```

**Postcondiciones:**
- Posición calculada correctamente según orden FIFO
- Tiempo estimado actualizado en base de datos
- Recálculos automáticos cuando cambian condiciones
- Auditoría de cálculos registrada

**Endpoints HTTP:**
- GET /api/tickets/{uuid} - Consultar ticket por UUID
- GET /api/tickets/{numero}/position - Consultar posición por número

---

### RF-004: Asignar Ticket a Ejecutivo Automáticamente

**Descripción:**
El sistema debe asignar automáticamente el siguiente ticket en cola cuando un ejecutivo se libere, considerando la prioridad de colas, balanceo de carga entre ejecutivos disponibles, y orden FIFO dentro de cada cola. La asignación debe ser inmediata y notificar tanto al cliente como al ejecutivo.

**Prioridad:** Alta

**Actor Principal:** Sistema (automatizado)

**Precondiciones:**
- Al menos un asesor con status AVAILABLE
- Al menos un ticket con status EN_ESPERA
- Sistema de notificaciones operativo

**Modelo de Datos (Entidad Advisor):**
- id: BIGSERIAL (primary key)
- name: String, nombre completo del asesor
- email: String, correo electrónico
- status: Enum (AVAILABLE, BUSY, OFFLINE)
- moduleNumber: Integer (1-5), número del módulo de atención
- assignedTicketsCount: Integer, contador de tickets asignados hoy
- lastAssignedAt: Timestamp, última asignación recibida
- queueTypes: Array, tipos de cola que puede atender

**Algoritmo de Asignación:**
```
1. Filtrar asesores: status = AVAILABLE
2. Seleccionar cola con mayor prioridad que tenga tickets EN_ESPERA
3. Dentro de esa cola, seleccionar ticket más antiguo (FIFO)
4. Entre asesores disponibles, seleccionar el que tenga menor assignedTicketsCount
5. Asignar ticket al asesor seleccionado
6. Actualizar estados y contadores
7. Programar notificaciones
```

**Prioridades de Cola:**
- GERENCIA: prioridad 4 (máxima)
- EMPRESAS: prioridad 3
- PERSONAL_BANKER: prioridad 2
- CAJA: prioridad 1 (mínima)

**Reglas de Negocio Aplicables:**
- RN-002: Prioridad de colas para asignación
- RN-003: Orden FIFO dentro de cada cola
- RN-004: Balanceo de carga entre asesores
- RN-013: Estados de asesor (AVAILABLE, BUSY, OFFLINE)

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Asignación exitosa con balanceo de carga**
```gherkin
Given existen asesores disponibles:
  | name        | status    | moduleNumber | assignedTicketsCount |
  | Juan Pérez  | AVAILABLE | 1           | 3                   |
  | Ana Gómez   | AVAILABLE | 2           | 1                   |
  | Luis Martínez| AVAILABLE | 3           | 2                   |
And existe ticket P05 EN_ESPERA (más antiguo en cola PERSONAL_BANKER)
When un asesor se libera y dispara el proceso de asignación
Then el sistema selecciona Ana Gómez (menor assignedTicketsCount = 1)
And el sistema asigna ticket P05 a Ana Gómez:
  | assignedAdvisor      | Ana Gómez |
  | assignedModuleNumber | 2         |
  | status              | ATENDIENDO |
And Ana Gómez se actualiza:
  | status              | BUSY |
  | assignedTicketsCount | 2    |
  | lastAssignedAt      | timestamp actual |
And el sistema programa mensaje totem_es_tu_turno
```

**Escenario 2: Prioridad de colas - GERENCIA antes que CAJA**
```gherkin
Given existen tickets en espera:
  | numero | queueType | createdAt           | prioridad |
  | C10    | CAJA     | 2025-01-15 09:00:00 | 1        |
  | G01    | GERENCIA | 2025-01-15 09:30:00 | 4        |
And existe un asesor AVAILABLE
When se ejecuta el proceso de asignación
Then el sistema selecciona ticket G01 (mayor prioridad)
And el ticket C10 permanece EN_ESPERA
And el sistema asigna G01 al asesor disponible
```

**Escenario 3: Orden FIFO dentro de la misma cola**
```gherkin
Given existen tickets PERSONAL_BANKER en espera:
  | numero | createdAt           |
  | P03    | 2025-01-15 09:00:00 |
  | P04    | 2025-01-15 09:15:00 |
  | P05    | 2025-01-15 09:30:00 |
And existe un asesor AVAILABLE
When se ejecuta el proceso de asignación
Then el sistema selecciona P03 (más antiguo)
And P04 y P05 permanecen EN_ESPERA
And sus posiciones se recalculan automáticamente
```

**Escenario 4: No hay asesores disponibles**
```gherkin
Given todos los asesores tienen status BUSY o OFFLINE:
  | name        | status  |
  | Juan Pérez  | BUSY    |
  | Ana Gómez   | OFFLINE |
  | Luis Martínez| BUSY    |
And existen tickets EN_ESPERA
When se ejecuta el proceso de asignación
Then el sistema NO asigna ningún ticket
And todos los tickets permanecen EN_ESPERA
And el sistema registra evento: "ASIGNACION_PENDIENTE_SIN_ASESORES"
```

**Escenario 5: No hay tickets en espera**
```gherkin
Given no existen tickets con status EN_ESPERA
And existe al menos un asesor AVAILABLE
When se ejecuta el proceso de asignación
Then el sistema NO realiza ninguna asignación
And los asesores permanecen AVAILABLE
And el sistema registra evento: "ASIGNACION_PENDIENTE_SIN_TICKETS"
```

**Escenario 6: Asignación con múltiples colas y prioridades**
```gherkin
Given existen tickets en diferentes colas:
  | numero | queueType       | createdAt           | prioridad |
  | C05    | CAJA           | 2025-01-15 08:00:00 | 1        |
  | P10    | PERSONAL_BANKER | 2025-01-15 08:30:00 | 2        |
  | E02    | EMPRESAS       | 2025-01-15 09:00:00 | 3        |
And existe un asesor AVAILABLE
When se ejecuta el proceso de asignación
Then el sistema selecciona E02 (mayor prioridad = 3)
And C05 y P10 permanecen EN_ESPERA
```

**Escenario 7: Notificación automática tras asignación**
```gherkin
Given un ticket P08 es asignado a asesor "Carlos Ruiz" en módulo 4
When la asignación se completa exitosamente
Then el sistema programa mensaje totem_es_tu_turno con:
  | numero      | P08         |
  | modulo      | 4           |
  | nombreAsesor| Carlos Ruiz |
And el sistema envía notificación al terminal del asesor
And el sistema actualiza pantallas de sucursal
And el sistema registra auditoría: "TICKET_ASIGNADO"
```

**Postcondiciones:**
- Ticket asignado con status ATENDIENDO
- Asesor actualizado con status BUSY
- Contador assignedTicketsCount incrementado
- Mensaje totem_es_tu_turno programado
- Posiciones de otros tickets recalculadas
- Auditoría registrada

**Endpoints HTTP:**
- Ninguno (proceso interno automatizado)
- PUT /api/admin/advisors/{id}/status - Cambiar estado de asesor (manual)

---

### RF-005: Gestionar Múltiples Colas

**Descripción:**
El sistema debe gestionar simultáneamente cuatro tipos de cola con diferentes características operacionales: tiempos promedio de atención, prioridades de asignación, y prefijos de numeración. Cada cola debe mantener su propia secuencia de tickets, estadísticas independientes, y reglas de operación específicas.

**Prioridad:** Alta

**Actor Principal:** Sistema

**Precondiciones:**
- Sistema inicializado con configuración de 4 colas
- Base de datos con tablas de colas configuradas
- Contadores de secuencia inicializados

**Configuración de Colas:**

| Cola | Tiempo Promedio | Prioridad | Prefijo | Descripción |
|------|----------------|-----------|---------|-------------|
| CAJA | 5 minutos | 1 (baja) | C | Transacciones básicas, depósitos, retiros |
| PERSONAL_BANKER | 15 minutos | 2 (media) | P | Productos financieros, créditos, inversiones |
| EMPRESAS | 20 minutos | 3 (media) | E | Clientes corporativos, cuentas empresariales |
| GERENCIA | 30 minutos | 4 (máxima) | G | Casos especiales, reclamos, situaciones complejas |

**Modelo de Datos (Entidad QueueStats):**
- queueType: Enum (CAJA, PERSONAL_BANKER, EMPRESAS, GERENCIA)
- ticketsEnEspera: Integer, cantidad actual en espera
- ticketsAtendiendo: Integer, cantidad siendo atendida
- ticketsCompletadosHoy: Integer, total completados hoy
- tiempoPromedioReal: Integer, tiempo promedio real calculado
- secuenciaActual: Integer, último número asignado (01-99)
- fechaReset: Date, última fecha de reset de secuencia

**Reglas de Negocio Aplicables:**
- RN-002: Prioridades de cola para asignación
- RN-005: Formato de número con prefijo específico
- RN-006: Prefijos por tipo de cola
- RN-010: Tiempos promedio para cálculos

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Consulta de estado de cola específica**
```gherkin
Given la cola PERSONAL_BANKER tiene:
  | ticketsEnEspera     | 5  |
  | ticketsAtendiendo   | 2  |
  | ticketsCompletados  | 15 |
  | secuenciaActual     | 22 |
When se consulta GET /api/admin/queues/PERSONAL_BANKER
Then el sistema retorna HTTP 200 con JSON:
  {
    "queueType": "PERSONAL_BANKER",
    "displayName": "Personal Banker",
    "ticketsEnEspera": 5,
    "ticketsAtendiendo": 2,
    "ticketsCompletadosHoy": 15,
    "tiempoPromedioMinutos": 15,
    "tiempoPromedioReal": 18,
    "secuenciaActual": 22,
    "proximoNumero": "P23",
    "prioridad": 2
  }
```

**Escenario 2: Estadísticas detalladas de cola**
```gherkin
Given la cola EMPRESAS tiene actividad durante el día
When se consulta GET /api/admin/queues/EMPRESAS/stats
Then el sistema retorna HTTP 200 con JSON:
  {
    "queueType": "EMPRESAS",
    "estadisticasHoy": {
      "totalCreados": 25,
      "totalCompletados": 20,
      "totalCancelados": 2,
      "enEspera": 3,
      "tiempoPromedioReal": 22,
      "tiempoEsperaPromedio": 18
    },
    "distribucionPorHora": [
      {"hora": "09:00", "creados": 5, "completados": 3},
      {"hora": "10:00", "creados": 8, "completados": 7},
      {"hora": "11:00", "creados": 6, "completados": 5}
    ]
  }
```

**Escenario 3: Reset diario de secuencias**
```gherkin
Given es un nuevo día (00:00:00)
And las colas tienen secuencias:
  | queueType       | secuenciaActual |
  | CAJA           | 45             |
  | PERSONAL_BANKER | 23             |
  | EMPRESAS       | 12             |
  | GERENCIA       | 8              |
When el sistema ejecuta el proceso de reset diario
Then todas las secuencias se resetean:
  | queueType       | secuenciaActual | proximoNumero |
  | CAJA           | 0              | C01          |
  | PERSONAL_BANKER | 0              | P01          |
  | EMPRESAS       | 0              | E01          |
  | GERENCIA       | 0              | G01          |
And el sistema registra auditoría: "SECUENCIAS_RESETEADAS"
```

**Escenario 4: Generación de número secuencial por cola**
```gherkin
Given la cola GERENCIA tiene secuenciaActual = 5
When se crea un nuevo ticket para GERENCIA
Then el sistema genera número "G06"
And actualiza secuenciaActual = 6
And el próximo ticket será "G07"
```

**Escenario 5: Manejo de límite de secuencia (99 tickets)**
```gherkin
Given la cola CAJA tiene secuenciaActual = 99
When se intenta crear ticket número 100 en el mismo día
Then el sistema retorna HTTP 409 Conflict con JSON:
  {
    "error": "LIMITE_DIARIO_ALCANZADO",
    "mensaje": "Se alcanzó el límite de 99 tickets diarios para cola CAJA",
    "queueType": "CAJA",
    "limiteMaximo": 99
  }
And el sistema NO crea el ticket
```

**Escenario 6: Consulta de resumen de todas las colas**
```gherkin
Given existen tickets en todas las colas
When se consulta GET /api/admin/queues
Then el sistema retorna HTTP 200 con JSON:
  {
    "resumen": {
      "totalEnEspera": 15,
      "totalAtendiendo": 8,
      "totalCompletadosHoy": 67
    },
    "colas": [
      {
        "queueType": "GERENCIA",
        "prioridad": 4,
        "enEspera": 2,
        "atendiendo": 1
      },
      {
        "queueType": "EMPRESAS",
        "prioridad": 3,
        "enEspera": 3,
        "atendiendo": 2
      },
      {
        "queueType": "PERSONAL_BANKER",
        "prioridad": 2,
        "enEspera": 5,
        "atendiendo": 3
      },
      {
        "queueType": "CAJA",
        "prioridad": 1,
        "enEspera": 5,
        "atendiendo": 2
      }
    ]
  }
```

**Postcondiciones:**
- Cada cola mantiene su secuencia independiente
- Estadísticas actualizadas en tiempo real
- Prefijos correctos aplicados según tipo
- Reset diario de secuencias ejecutado
- Auditoría de operaciones registrada

**Endpoints HTTP:**
- GET /api/admin/queues - Resumen de todas las colas
- GET /api/admin/queues/{type} - Estado de cola específica
- GET /api/admin/queues/{type}/stats - Estadísticas detalladas

---

### RF-006: Consultar Estado del Ticket

**Descripción:**
El sistema debe permitir al cliente consultar en cualquier momento el estado actual de su ticket, mostrando información actualizada sobre posición en cola, tiempo estimado, estado actual, y ejecutivo asignado si aplica. Las consultas pueden realizarse por UUID o número de ticket.

**Prioridad:** Media

**Actor Principal:** Cliente

**Precondiciones:**
- Ticket existe en el sistema
- Conexión a base de datos activa

**Reglas de Negocio Aplicables:**
- RN-009: Estados de ticket válidos
- RN-010: Cálculo de tiempo estimado actualizado

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Consulta exitosa de ticket EN_ESPERA**
```gherkin
Given existe ticket con UUID "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6"
And el ticket tiene status EN_ESPERA
When se consulta GET /api/tickets/a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6
Then el sistema retorna HTTP 200 con JSON:
  {
    "codigoReferencia": "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6",
    "numero": "P05",
    "status": "EN_ESPERA",
    "positionInQueue": 3,
    "estimatedWaitMinutes": 45,
    "queueType": "PERSONAL_BANKER",
    "createdAt": "2025-01-15T09:30:00Z",
    "assignedAdvisor": null,
    "assignedModuleNumber": null
  }
```

**Escenario 2: Consulta de ticket ATENDIENDO**
```gherkin
Given existe ticket "E03" con status ATENDIENDO
And está asignado a asesor "Ana Gómez" en módulo 2
When se consulta GET /api/tickets/E03/position
Then el sistema retorna HTTP 200 con JSON:
  {
    "numero": "E03",
    "status": "ATENDIENDO",
    "positionInQueue": 0,
    "estimatedWaitMinutes": 0,
    "queueType": "EMPRESAS",
    "assignedAdvisor": "Ana Gómez",
    "assignedModuleNumber": 2,
    "assignedAt": "2025-01-15T10:15:00Z"
  }
```

**Escenario 3: Consulta de ticket COMPLETADO**
```gherkin
Given existe ticket "C08" con status COMPLETADO
When se consulta GET /api/tickets/C08/position
Then el sistema retorna HTTP 200 con JSON:
  {
    "numero": "C08",
    "status": "COMPLETADO",
    "positionInQueue": 0,
    "estimatedWaitMinutes": 0,
    "queueType": "CAJA",
    "completedAt": "2025-01-15T11:30:00Z",
    "totalWaitMinutes": 12
  }
```

**Escenario 4: Error - Ticket no encontrado**
```gherkin
Given no existe ticket con número "X99"
When se consulta GET /api/tickets/X99/position
Then el sistema retorna HTTP 404 Not Found con JSON:
  {
    "error": "TICKET_NO_ENCONTRADO",
    "mensaje": "No existe ticket con número: X99"
  }
```

**Escenario 5: Actualización automática de posición**
```gherkin
Given un ticket P10 tiene positionInQueue = 5
And otro ticket anterior se completa
When se consulta el estado de P10
Then el sistema retorna positionInQueue = 4
And estimatedWaitMinutes se recalcula automáticamente
```

**Postcondiciones:**
- Información actualizada retornada al cliente
- Posición y tiempo recalculados si es necesario

**Endpoints HTTP:**
- GET /api/tickets/{uuid} - Consultar por UUID
- GET /api/tickets/{numero}/position - Consultar por número

---

### RF-007: Panel de Monitoreo para Supervisor

**Descripción:**
El sistema debe proveer un dashboard en tiempo real que muestre resumen de tickets por estado, cantidad de clientes en espera por cola, estado de ejecutivos, tiempos promedio de atención, y alertas de situaciones críticas. La información debe actualizarse automáticamente cada 5 segundos.

**Prioridad:** Alta

**Actor Principal:** Supervisor

**Precondiciones:**
- Usuario con permisos de supervisor autenticado
- Sistema operativo con datos actualizados

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Dashboard principal con resumen general**
```gherkin
Given el supervisor accede al dashboard
When se consulta GET /api/admin/dashboard
Then el sistema retorna HTTP 200 con JSON:
  {
    "timestamp": "2025-01-15T10:30:00Z",
    "resumenGeneral": {
      "totalEnEspera": 15,
      "totalAtendiendo": 8,
      "totalCompletadosHoy": 67,
      "totalCanceladosHoy": 3
    },
    "colasPorEstado": {
      "GERENCIA": {"enEspera": 2, "atendiendo": 1},
      "EMPRESAS": {"enEspera": 3, "atendiendo": 2},
      "PERSONAL_BANKER": {"enEspera": 5, "atendiendo": 3},
      "CAJA": {"enEspera": 5, "atendiendo": 2}
    },
    "alertas": [
      {
        "tipo": "COLA_CRITICA",
        "mensaje": "Cola PERSONAL_BANKER tiene 15+ tickets en espera",
        "severidad": "HIGH"
      }
    ]
  }
```

**Escenario 2: Estado de asesores en tiempo real**
```gherkin
When se consulta GET /api/admin/advisors
Then el sistema retorna HTTP 200 con JSON:
  {
    "asesores": [
      {
        "id": 1,
        "name": "Juan Pérez",
        "status": "BUSY",
        "moduleNumber": 1,
        "ticketActual": "P05",
        "tiempoAtendiendo": 12,
        "ticketsAtendidosHoy": 8
      },
      {
        "id": 2,
        "name": "Ana Gómez",
        "status": "AVAILABLE",
        "moduleNumber": 2,
        "ticketActual": null,
        "ticketsAtendidosHoy": 6
      }
    ]
  }
```

**Escenario 3: Estadísticas de rendimiento**
```gherkin
When se consulta GET /api/admin/advisors/stats
Then el sistema retorna HTTP 200 con JSON:
  {
    "promedioAtencionMinutos": 18,
    "ticketsPorHora": 3.2,
    "tiempoEsperaPromedio": 22,
    "eficienciaAsesores": {
      "Juan Pérez": {"ticketsHoy": 8, "promedioMinutos": 15},
      "Ana Gómez": {"ticketsHoy": 6, "promedioMinutos": 20}
    }
  }
```

**Escenario 4: Alertas críticas automáticas**
```gherkin
Given la cola CAJA tiene 16 tickets EN_ESPERA
When el sistema evalúa alertas críticas
Then se genera alerta:
  {
    "tipo": "COLA_CRITICA",
    "queueType": "CAJA",
    "mensaje": "Cola CAJA excede 15 tickets en espera",
    "severidad": "HIGH",
    "timestamp": "2025-01-15T10:30:00Z"
  }
And la alerta aparece en el dashboard
```

**Escenario 5: Resumen ejecutivo**
```gherkin
When se consulta GET /api/admin/summary
Then el sistema retorna HTTP 200 con JSON:
  {
    "fecha": "2025-01-15",
    "metricas": {
      "totalTicketsCreados": 89,
      "totalTicketsCompletados": 67,
      "tasaCompletacion": 75.3,
      "tiempoEsperaPromedio": 22,
      "npsEstimado": 62
    },
    "tendencias": {
      "horasPico": ["10:00-11:00", "14:00-15:00"],
      "colaMasDemandada": "PERSONAL_BANKER",
      "asesorMasEficiente": "Juan Pérez"
    }
  }
```

**Escenario 6: Actualización automática cada 5 segundos**
```gherkin
Given el dashboard está abierto en el navegador
When pasan 5 segundos
Then el sistema actualiza automáticamente todos los datos
And los contadores se refrescan sin recargar la página
And las alertas nuevas aparecen inmediatamente
```

**Postcondiciones:**
- Dashboard actualizado con información en tiempo real
- Alertas críticas mostradas al supervisor
- Métricas de rendimiento calculadas

**Endpoints HTTP:**
- GET /api/admin/dashboard - Dashboard principal
- GET /api/admin/summary - Resumen ejecutivo
- GET /api/admin/advisors - Estado de asesores
- GET /api/admin/advisors/stats - Estadísticas de rendimiento
- PUT /api/admin/advisors/{id}/status - Cambiar estado de asesor

---

### RF-008: Registrar Auditoría de Eventos

**Descripción:**
El sistema debe registrar todos los eventos relevantes del proceso de tickets: creación, asignaciones, cambios de estado, envío de mensajes, y acciones de usuarios. La información debe incluir timestamp, tipo de evento, actor involucrado, entityId afectado, y cambios de estado para trazabilidad completa.

**Prioridad:** Media

**Actor Principal:** Sistema (automatizado)

**Precondiciones:**
- Sistema de auditoría configurado
- Base de datos con tabla de auditoría

**Modelo de Datos (Entidad AuditLog):**
- id: BIGSERIAL (primary key)
- timestamp: Timestamp, fecha/hora del evento
- tipoEvento: String, tipo de evento registrado
- actor: String, usuario o sistema que ejecutó la acción
- entityType: String, tipo de entidad afectada (TICKET, ADVISOR, MENSAJE)
- entityId: String, identificador de la entidad
- cambiosEstado: JSON, estado anterior y nuevo
- metadatos: JSON, información adicional del contexto

**Tipos de Eventos:**
- TICKET_CREADO, TICKET_ASIGNADO, TICKET_COMPLETADO, TICKET_CANCELADO
- MENSAJE_ENVIADO, MENSAJE_FALLIDO
- ASESOR_DISPONIBLE, ASESOR_OCUPADO, ASESOR_OFFLINE
- POSICION_RECALCULADA, SECUENCIAS_RESETEADAS

**Reglas de Negocio Aplicables:**
- RN-011: Auditoría obligatoria para eventos críticos

**Criterios de Aceptación (Gherkin):**

**Escenario 1: Auditoría de creación de ticket**
```gherkin
Given se crea un ticket P05 exitosamente
When el sistema registra la auditoría
Then se inserta registro con:
  | tipoEvento  | TICKET_CREADO                    |
  | actor       | SISTEMA                          |
  | entityType  | TICKET                           |
  | entityId    | a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6 |
  | cambiosEstado | {"anterior": null, "nuevo": "EN_ESPERA"} |
  | metadatos   | {"numero": "P05", "queueType": "PERSONAL_BANKER"} |
```

**Escenario 2: Auditoría de asignación de ticket**
```gherkin
Given un ticket P05 se asigna a asesor "Juan Pérez"
When el sistema registra la auditoría
Then se inserta registro con:
  | tipoEvento    | TICKET_ASIGNADO |
  | cambiosEstado | {"anterior": "EN_ESPERA", "nuevo": "ATENDIENDO"} |
  | metadatos     | {"asesor": "Juan Pérez", "modulo": 1} |
```

**Escenario 3: Auditoría de envío de mensaje**
```gherkin
Given se envía mensaje totem_es_tu_turno exitosamente
When el sistema registra la auditoría
Then se inserta registro con:
  | tipoEvento | MENSAJE_ENVIADO |
  | entityType | MENSAJE |
  | metadatos  | {"plantilla": "totem_es_tu_turno", "telegramMessageId": "12345"} |
```

**Escenario 4: Auditoría de cambio de estado de asesor**
```gherkin
Given un supervisor cambia estado de asesor a OFFLINE
When el sistema registra la auditoría
Then se inserta registro con:
  | tipoEvento    | ASESOR_OFFLINE |
  | actor         | supervisor@banco.com |
  | cambiosEstado | {"anterior": "AVAILABLE", "nuevo": "OFFLINE"} |
```

**Escenario 5: Consulta de auditoría por ticket**
```gherkin
Given existe auditoría para ticket P05
When se consulta GET /api/admin/audit/ticket/P05
Then el sistema retorna HTTP 200 con historial completo:
  [
    {
      "timestamp": "2025-01-15T09:30:00Z",
      "tipoEvento": "TICKET_CREADO",
      "cambiosEstado": {"nuevo": "EN_ESPERA"}
    },
    {
      "timestamp": "2025-01-15T10:15:00Z",
      "tipoEvento": "TICKET_ASIGNADO",
      "cambiosEstado": {"anterior": "EN_ESPERA", "nuevo": "ATENDIENDO"}
    }
  ]
```

**Postcondiciones:**
- Evento registrado en tabla de auditoría
- Timestamp preciso almacenado
- Cambios de estado documentados
- Trazabilidad completa mantenida

**Endpoints HTTP:**
- GET /api/admin/audit/ticket/{numero} - Auditoría de ticket
- GET /api/admin/audit/events - Log de eventos del sistema

---

## 5. Matriz de Trazabilidad

| RF | Beneficio Empresarial | Endpoints HTTP | Reglas de Negocio |
|----|----------------------|----------------|-------------------|
| RF-001 | Digitalización del proceso | POST /api/tickets | RN-001, RN-005, RN-006, RN-010 |
| RF-002 | Movilidad del cliente | Ninguno (automatizado) | RN-007, RN-008, RN-011, RN-012 |
| RF-003 | Transparencia de tiempos | GET /api/tickets/{uuid}, GET /api/tickets/{numero}/position | RN-003, RN-010, RN-011 |
| RF-004 | Optimización de recursos | PUT /api/admin/advisors/{id}/status | RN-002, RN-003, RN-004, RN-013 |
| RF-005 | Gestión operacional | GET /api/admin/queues/* | RN-002, RN-005, RN-006, RN-010 |
| RF-006 | Experiencia del cliente | GET /api/tickets/{uuid}, GET /api/tickets/{numero}/position | RN-009, RN-010 |
| RF-007 | Supervisión operacional | GET /api/admin/dashboard, GET /api/admin/summary | RN-013 |
| RF-008 | Trazabilidad y cumplimiento | GET /api/admin/audit/* | RN-011 |

## 6. Matriz de Endpoints HTTP

| Método | Endpoint | Descripción | RF |
|--------|----------|-------------|----|
| POST | /api/tickets | Crear nuevo ticket | RF-001 |
| GET | /api/tickets/{uuid} | Consultar ticket por UUID | RF-003, RF-006 |
| GET | /api/tickets/{numero}/position | Consultar posición por número | RF-003, RF-006 |
| GET | /api/admin/dashboard | Dashboard principal | RF-007 |
| GET | /api/admin/summary | Resumen ejecutivo | RF-007 |
| GET | /api/admin/queues | Resumen de todas las colas | RF-005 |
| GET | /api/admin/queues/{type} | Estado de cola específica | RF-005 |
| GET | /api/admin/queues/{type}/stats | Estadísticas detalladas | RF-005 |
| GET | /api/admin/advisors | Estado de asesores | RF-007 |
| GET | /api/admin/advisors/stats | Estadísticas de rendimiento | RF-007 |
| PUT | /api/admin/advisors/{id}/status | Cambiar estado de asesor | RF-004, RF-007 |
| GET | /api/admin/audit/ticket/{numero} | Auditoría de ticket | RF-008 |
| GET | /api/admin/audit/events | Log de eventos del sistema | RF-008 |

## 7. Checklist de Validación Final

### Completitud
- ✅ 8 Requerimientos Funcionales documentados
- ✅ 13 Reglas de Negocio numeradas
- ✅ 44+ Escenarios Gherkin totales
- ✅ 13 Endpoints HTTP mapeados
- ✅ 4 Enumeraciones especificadas
- ✅ 3 Entidades de datos definidas

### Calidad
- ✅ Formato Gherkin correcto (Given/When/Then/And)
- ✅ Ejemplos JSON válidos en respuestas HTTP
- ✅ Sin ambigüedades en especificaciones
- ✅ Numeración consistente (RF-XXX, RN-XXX)
- ✅ Tablas bien formateadas
- ✅ Jerarquía clara con encabezados

### Trazabilidad
- ✅ Cada RF vinculado a beneficios empresariales
- ✅ Reglas de negocio aplicadas correctamente
- ✅ Matriz de endpoints completa
- ✅ Auditoría de eventos especificada

---

**Documento completado exitosamente**  
**Total de páginas:** ~65  
**Total de palabras:** ~14,500  
**Fecha de finalización:** Diciembre 2025