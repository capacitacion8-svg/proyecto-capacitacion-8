# **Sistema de Gestión de Tickets para Atención en Sucursales**

**Proyecto:** Ticketero Digital con Notificaciones en Tiempo Real  
**Cliente:** Institución Financiera  
**Versión:** 1.0  
**Fecha:** Diciembre 2025

## **1\. Descripción del Proyecto**

### **1.1 Contexto**

Las instituciones financieras enfrentan desafíos en la atención presencial: los clientes no tienen visibilidad de tiempos de espera, deben permanecer físicamente en sucursal sin poder realizar otras actividades, y existe incertidumbre sobre el progreso de su turno.

### **1.2 Solución Propuesta**

Sistema digital de gestión de tickets que moderniza la experiencia de atención mediante:

* Digitalización del proceso de tickets  
* Notificaciones automáticas en tiempo real vía Telegram  
* Movilidad del cliente durante la espera  
* Asignación automática de clientes a ejecutivos disponibles  
* Panel de monitoreo para supervisión operacional

### **1.3 Beneficios Esperados**

* Mejora de NPS de 45 a 65 puntos  
* Reducción de abandonos de cola de 15% a 5%  
* Incremento de 20% en tickets atendidos por ejecutivo  
* Trazabilidad completa para análisis y mejora continua

## **2\. Requerimientos Funcionales**

### **RF-001: Crear Ticket Digital**

El sistema debe permitir al cliente obtener un ticket digital ingresando su RUT/ID y seleccionando el tipo de atención requerida (Caja, Personal Banker, Empresas, Gerencia). El sistema generará un número único, calculará la posición en cola y el tiempo estimado de espera.

### **RF-002: Enviar Notificaciones Automáticas**

El sistema debe enviar tres mensajes automáticos vía Telegram:

* **Mensaje 1 \- Confirmación:** Inmediatamente al crear el ticket, incluyendo número, posición y tiempo estimado  
* **Mensaje 2 \- Pre-aviso:** Cuando quedan 3 personas adelante, solicitando acercarse a sucursal  
* **Mensaje 3 \- Turno Activo:** Al asignar a un ejecutivo, indicando módulo y nombre del asesor

### **RF-003: Calcular Posición y Tiempo Estimado**

El sistema debe calcular en tiempo real la posición exacta del cliente en cola y estimar el tiempo de espera basado en: posición actual, tiempo promedio de atención por tipo de cola, y cantidad de ejecutivos disponibles.

### **RF-004: Asignar Ticket a Ejecutivo Automáticamente**

El sistema debe asignar automáticamente el siguiente ticket en cola cuando un ejecutivo se libere, considerando: prioridad de colas, balanceo de carga entre ejecutivos, y orden FIFO dentro de cada cola.

### **RF-005: Gestionar Múltiples Colas**

El sistema debe gestionar cuatro tipos de cola con diferentes características:

* **Caja:** Transacciones básicas (5 min promedio, prioridad baja)  
* **Personal Banker:** Productos financieros (15 min promedio, prioridad media)  
* **Empresas:** Clientes corporativos (20 min promedio, prioridad media)  
* **Gerencia:** Casos especiales (30 min promedio, prioridad máxima)

### **RF-006: Consultar Estado del Ticket**

El sistema debe permitir al cliente consultar en cualquier momento el estado de su ticket, mostrando: estado actual, posición en cola, tiempo estimado actualizado, y ejecutivo asignado si aplica.

### **RF-007: Panel de Monitoreo para Supervisor**

El sistema debe proveer un dashboard en tiempo real que muestre: resumen de tickets por estado, cantidad de clientes en espera por cola, estado de ejecutivos, tiempos promedio de atención, y alertas de situaciones críticas. Actualización automática cada 5 segundos.

### **RF-008: Registrar Auditoría de Eventos**

El sistema debe registrar todos los eventos relevantes: creación de tickets, asignaciones, cambios de estado, envío de mensajes, y acciones de usuarios. Información debe incluir timestamp, tipo de evento, actor involucrado y cambios de estado.

## **3\. Flujo Detallado del Proceso**

### **3.1 Emisión de Ticket**

1. Cliente ingresa RUT/ID en terminal  
2. Sistema valida identificación  
3. Cliente selecciona tipo de atención  
4. Sistema genera ticket con número único  
5. Sistema calcula posición (\#5) y tiempo estimado (25 min)  
6. Sistema muestra confirmación en pantalla  
7. Sistema envía Mensaje 1 de confirmación vía Telegram

### **3.2 Notificación de Progreso**

8. Cliente puede salir de sucursal  
9. Sistema monitorea progreso de cola automáticamente  
10. Cuando posición ≤ 3, sistema envía Mensaje 2 (pre-aviso)  
11. Cliente regresa a sucursal

### **3.3 Asignación y Atención**

12. Sistema detecta ejecutivo disponible  
13. Sistema asigna ticket al ejecutivo óptimo (balanceo de carga)  
14. Sistema envía Mensaje 3 con módulo y nombre del asesor  
15. Sistema notifica al ejecutivo en su terminal  
16. Sistema actualiza pantallas de sucursal  
17. Cliente se presenta en módulo indicado  
18. Ejecutivo atiende al cliente  
19. Sistema marca ticket como completado  
20. Sistema libera ejecutivo para siguiente asignación

### **3.4 Supervisión**

* Dashboard actualiza métricas en tiempo real cada 5 segundos  
* Supervisor monitorea estado de colas y ejecutivos  
* Sistema genera alertas si cola crítica (más de 15 esperando)  
* Registros de auditoría disponibles para análisis

## **4\. Requerimientos No Funcionales**

### **RNF-001: Disponibilidad**

* Uptime de 99.5% durante horario de atención  
* Máximo 4 horas de downtime al mes  
* Recovery automático en menos de 5 minutos

### **RNF-002: Performance**

* Creación de ticket: menos de 3 segundos  
* Envío de Mensaje 1: menos de 5 segundos  
* Cálculo de posición: menos de 1 segundo  
* Actualización de dashboard: cada 5 segundos

### **RNF-003: Escalabilidad**

* Fase Piloto: 500-800 tickets/día, 1 sucursal  
* Fase Expansión: 2,500-3,000 tickets/día, 5 sucursales  
* Fase Nacional: 25,000+ tickets/día, 50+ sucursales

### **RNF-004: Confiabilidad**

* 99.9% de mensajes entregados exitosamente  
* Sin pérdida de datos ante fallos  
* 3 reintentos automáticos para envío de mensajes (30s, 60s, 120s)

### **RNF-005: Seguridad**

* Cumplimiento de ley de protección de datos personales  
* Encriptación de datos sensibles (teléfonos, RUT)  
* Acceso controlado al panel administrativo  
* Logs de auditoría de todos los accesos

### **RNF-006: Usabilidad**

* Cliente obtiene ticket en menos de 2 minutos  
* Interfaz intuitiva sin necesidad de capacitación  
* Mensajes claros en español simple  
* Dashboard comprensible a primera vista

### **RNF-007: Mantenibilidad**

* Código documentado  
* Arquitectura modular  
* Logs detallados para diagnóstico  
* Actualizaciones sin interrupción de servicio

---

**Preparado por:** Área de Producto e Innovación  
**Tipo:** Proyecto de Capacitación \- Ciclo Completo de Desarrollo de Software

