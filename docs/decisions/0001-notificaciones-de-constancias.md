# Procesamiento asíncrono mediante eventos persistidos para notificaciones de constancias

* status: proposed
* date: 2026-11-03

## Context and Problem Statement

El sistema ConstanciaUV debe permitir a los estudiantes solicitar, generar y consultar constancias digitales, notificándoles por correo electrónico cuando su documento esté disponible. El servicio de correo es un componente externo (RT1) cuya disponibilidad y control de duplicados no están garantizados por el proveedor.

El sistema debe garantizar que la generación y almacenamiento de constancias continúen operando con normalidad aun si el servicio de correo falla (RT2), almacenar los envíos fallidos para reintentarlos posteriormente (RT3) y evitar que se envíen notificaciones duplicadas para una misma constancia (RT4). ¿Cómo debe estructurarse la comunicación entre el núcleo de emisión de constancias y el servicio de notificaciones para desacoplar ambos procesos y asegurar la fiabilidad de las entregas?

## Decision Drivers

* **Resiliencia y disponibilidad del flujo principal (RT2, R2):** La generación y resguardo de la constancia no debe bloquearse, demorarse ni fallar debido a indisponibilidad o latencia del proveedor de correo externo.
* **Garantía de entrega e idempotencia (RT3, RT4):** Toda notificación debe registrarse de forma confiable para su reintento y el sistema debe asegurar que no se procese más de una solicitud de notificación hacia el proveedor para la misma constancia.

## Considered Options

* Alternativa A: Comunicación directa con recuperación de fallos
* Alternativa B: Procesamiento asíncrono mediante eventos persistidos

## Decision Outcome

Chosen option: "Alternativa B: Procesamiento asíncrono mediante eventos persistidos", because desacopla por completo la transacción de negocio de la integración externa. Tras almacenar la constancia, el sistema persiste localmente un evento o registro de notificación con estado pendiente dentro de la misma frontera transaccional. Un proceso independiente despacha estas notificaciones, lo que garantiza que la emisión de constancias nunca se detenga ante fallos de red (RT2), garantiza la retención duradera de pendientes (RT3) y permite controlar mediante restricciones de base de datos la unicidad del envío antes de contactar al proveedor (RT4).

### Consequences

* Good, because la generación de constancias responde de manera inmediata y consistente al estudiante sin depender de tiempos de espera de servicios de terceros.
* Good, because aísla la lógica de reintentos, control de errores y saturación de red en un componente dedicado sin afectar el núcleo del dominio.
* Bad, because introduce mayor complejidad arquitectónica al requerir infraestructura adicional para el procesamiento en segundo plano (workers, cron o cola de tareas).
* Bad, because el estudiante experimenta consistencia eventual: la notificación no es instantánea respecto a la generación del documento.
* Trade-off: Se prioriza la disponibilidad del servicio y la robustez transaccional a costa de requerir un despachador en segundo plano y una entrega asíncrona.

### Confirmation

El cumplimiento de esta decisión se comprobará en etapas posteriores de desarrollo mediante:

1. **Pruebas de tolerancia a fallos:** Configuración de un entorno de prueba donde el cliente hacia el servicio de correo devuelva intencionalmente respuestas erróneas (HTTP 500/503) o presente tiempos de espera infinitos. Se verificará que el flujo de solicitud de constancia concluya con éxito (HTTP 201), el PDF quede almacenado y el registro de notificación se guarde como pendiente sin elevar excepciones al usuario.
2. **Pruebas de concurrencia e idempotencia:** Ejecución de pruebas automatizadas donde múltiples hilos del despachador intenten procesar en paralelo la notificación de una misma constancia. Se verificará mediante aserciones en base de datos y mocks que el correo externo se invoque una sola vez y que restricciones de unicidad o bloqueos transaccionales eviten envíos duplicados.

## Pros and Cons of the Options

### Alternativa A: Comunicación directa con recuperación de fallos

Después de almacenar una constancia, la aplicación web invoca directamente el endpoint del proveedor de correo. Si la llamada falla o excede el tiempo de espera, se captura la excepción y se guarda el registro pendiente en base de datos para un reintento futuro.

* Good, because tiene una arquitectura más simple de implementar y entrega la notificación casi de inmediato en condiciones de red ideales.
* Bad, because la latencia del proveedor externo afecta de forma directa el tiempo de respuesta del estudiante al solicitar su constancia.
* Bad, because un fallo no controlado de red o un timeout prolongado puede interrumpir o revertir incorrectamente la transacción de almacenamiento de la constancia si no se aísla con extrema rigurosidad.
* Bad, because coordinar la prevención de duplicados entre el intento síncrono fallido y los procesos de reintento asíncronos aumenta el riesgo de condiciones de carrera.

### Alternativa B: Procesamiento asíncrono mediante eventos persistidos

Después de almacenar una constancia, la aplicación registra atómicamente un registro de notificación en estado pendiente (patrón Transactional Outbox). Un componente en segundo plano (worker) consulta periódicamente los registros pendientes, gestiona bloqueos, se comunica con el proveedor externo y actualiza su estado.

* Good, because garantiza total independencia entre la disponibilidad de la constancia digital y el estado del servicio de correo externo (cumpliendo RT2).
* Good, because asegura la persistencia atómica del evento pendiente junto con la constancia, imposibilitando la pérdida de notificaciones por caídas repentinas del servidor (cumpliendo RT3).
* Good, because la base de datos local actúa como árbitro central para garantizar idempotencia mediante claves únicas sobre la constancia (`id_constancia`), evitando solicitudes redundantes hacia el exterior (cumpliendo RT4).
* Bad, because agrega complejidad operativa al requerir la configuración, monitoreo y mantenimiento de un despachador en segundo plano.

## More Information

### Revisión 1. Cambio en el proveedor de notificaciones

* **Cambio recibido:** El proveedor externo modificó su interfaz de comunicación (API) y la estructura del contrato para reportar errores de envío (cambios en el cuerpo de las respuestas JSON y nuevos códigos de estado HTTP).
* **Impacto sobre la alternativa seleccionada:** El impacto se encuentra estrictamente acotado al adaptador de salida del worker de notificaciones. La emisión de constancias, su almacenamiento y el esquema de base de datos no experimentan alteración alguna.
* **Vigencia de la decisión:** La decisión se mantiene en estado `proposed` adoptando la Alternativa B. El incidente valida la robustez del desacoplamiento, ya que la modificación no afecta el flujo crítico del estudiante.
* **Consecuencias y comprobaciones actualizadas:**
  * Se identifica el riesgo de reintentos infinitos si un error permanente no se reconoce adecuadamente. Se incorpora una política de reintentos con descarte (*Dead Letter Queue* o estado de notificación `FALLIDO_PERMANENTE`) tras alcanzar un límite preestablecido de intentos.
  * Se incorporan pruebas de contrato (*Contract Tests*) y pruebas unitarias de mapeo de errores para validar exhaustivamente el nuevo esquema de respuestas devuelto por el proveedor de correo.
