---
status: "proposed"
date: 2026-11-03
---

# Comunicación entre la generación de constancias y el servicio externo de notificaciones

## Context and Problem Statement

ConstanciaUV genera y almacena constancias digitales (R2) y debe notificar al estudiante cuando su constancia esté disponible (R4). El envío de correos lo realiza un proveedor externo (RT1) que puede no estar disponible.

El equipo debe decidir cómo organizar la comunicación entre la generación de constancias y ese proveedor, bajo estas restricciones:

- Si el proveedor falla, la generación y el almacenamiento de constancias deben continuar (RT2).
- Las notificaciones pendientes deben conservarse para intentar enviarlas después (RT3).
- Debe prevenirse más de una solicitud de notificación para la misma constancia, sin suponer que el proveedor garantiza una entrega única (RT4).

La pregunta es: ¿intentamos enviar la notificación dentro del mismo flujo que almacena la constancia, o la registramos y la enviamos de forma independiente?

## Decision Drivers

* **D1. Continuidad ante fallos del proveedor (RT2, RT3):** la falla del correo no debe afectar la generación de constancias, y ninguna notificación pendiente debe perderse.
* **D2. Prevención de solicitudes duplicadas (RT4):** debe existir un mecanismo propio, verificable, que impida solicitar dos veces la notificación de una misma constancia.

## Considered Options

* **Alternativa A.** Comunicación directa con recuperación de fallos: tras almacenar la constancia, la aplicación solicita directamente el envío; si el proveedor no responde, registra el envío pendiente para reintentarlo.
* **Alternativa B.** Procesamiento asíncrono mediante eventos persistidos: tras almacenar la constancia, la aplicación registra una notificación pendiente; un componente independiente procesa esos registros y se comunica con el proveedor.

## Decision Outcome

Opción elegida: **"Alternativa B. Procesamiento asíncrono mediante eventos persistidos"**, porque concentra en un solo mecanismo el manejo de fallos, pendientes y duplicados, y desacopla por completo la generación de constancias del proveedor externo.

- **RT2:** el flujo de generación solo escribe en la base de datos propia (constancia y registro de notificación); nunca llama al proveedor, por lo que su caída no puede detenerlo.
- **RT3:** toda notificación nace como registro persistido con estado (`pendiente`, `enviada`, `fallida`); lo pendiente se conserva por construcción y el procesador lo reintenta.
- **RT4:** el registro tiene una restricción de unicidad por constancia, por lo que un segundo intento de crear la solicitud falla en la base de datos, sin depender del proveedor.

### Consequences

* Bueno, porque la generación de constancias es independiente de la disponibilidad del proveedor (D1).
* Bueno, porque existe un solo camino que maneja pendientes y reintentos, lo que reduce la lógica duplicada.
* Bueno, porque el conocimiento del proveedor (formato de solicitud y de errores) queda aislado en el componente procesador.
* Bueno, porque la unicidad por constancia es una regla verificable a nivel de datos (D2).
* Malo, porque se agrega un componente adicional que desplegar, monitorear y mantener.
* Malo, porque la notificación ya no es inmediata: llega después del siguiente ciclo de procesamiento.
* Riesgo: si el procesador envía el correo y falla antes de marcar el registro como enviado, podría reenviarlo. La restricción de unicidad evita duplicar la solicitud, pero no cubre este caso; se mitiga marcando el estado de forma atómica con bloqueo del registro y, si el proveedor lo permite, enviando un identificador de idempotencia. Si no lo permite, este riesgo residual queda documentado y aceptado.
* Riesgo (Revisión 1): la forma en que el procesador interpreta los errores del proveedor (reintentable o definitivo) depende del formato de errores del proveedor; un cambio en ese formato puede provocar reintentos indebidos o abandonar envíos que sí eran recuperables.
* Trade-off: se acepta mayor complejidad operativa y algo de latencia a cambio de continuidad, trazabilidad y un único punto de control de pendientes y duplicados.

### Confirmation

Pruebas y revisiones que podrían realizarse en una implementación futura (no se afirma que se hayan ejecutado):

1. **Fallo del proveedor (RT2):** simular que el proveedor no responde o devuelve error y verificar que la constancia se genera, se almacena y aparece en el historial.
2. **Pendientes (RT3):** con el proveedor caído, comprobar que el registro de notificación permanece en estado `pendiente`; al restablecerse el proveedor, comprobar que se envía y pasa a `enviada`.
3. **Duplicados (RT4):** intentar crear dos solicitudes para la misma constancia y verificar que la segunda es rechazada por la restricción de unicidad.
4. **Doble procesamiento:** ejecutar dos instancias del procesador sobre el mismo registro y verificar que solo una lo envía.
5. **Revisión de diseño:** verificar que ningún código del flujo de generación invoque al proveedor directamente.
6. **Pruebas de contrato del proveedor (agregada en la Revisión 1):** con respuestas de error simuladas en el formato nuevo y en el anterior, verificar que el procesador las clasifica correctamente como reintentables o definitivas, que un error desconocido se trata como reintentable con límite de intentos y que ninguna clasificación genera una segunda solicitud para la misma constancia.

## Pros and Cons of the Options

### Alternativa A. Comunicación directa con recuperación de fallos

* Bueno, porque es más simple de implementar y de entender al inicio.
* Bueno, porque el estudiante recibe la notificación casi de inmediato cuando el proveedor responde.
* Malo, porque el flujo de generación depende del proveedor: un proveedor lento o con timeouts afecta el tiempo de respuesta y exige proteger RT2 con cuidado.
* Malo, porque existen dos caminos que manejan pendientes (el intento directo y el reintento posterior), lo que duplica la lógica de fallos y de duplicados.
* Malo, porque un timeout ambiguo (no se sabe si el proveedor envió o no) puede producir un reintento duplicado, lo que dificulta cumplir RT4.

### Alternativa B. Procesamiento asíncrono mediante eventos persistidos

* Bueno, porque la generación de constancias nunca depende del proveedor (RT2).
* Bueno, porque hay un único camino para pendientes, reintentos y control de duplicados (RT3, RT4).
* Bueno, porque los cambios del proveedor se concentran en un componente.
* Malo, porque agrega un componente independiente que operar y monitorear.
* Malo, porque la notificación es diferida, no inmediata.

## More Information

Este ADR documenta únicamente la decisión sobre la comunicación con el servicio de notificaciones, no toda la arquitectura de ConstanciaUV. Es una propuesta académica (`proposed`) y no una decisión institucional aprobada.

### Revisión 1. Cambio en el proveedor de notificaciones

**Cambio recibido.** El proveedor externo modificó su interfaz de comunicación y cambió la forma en que reporta los errores de envío. No se dispone de resultados de pruebas ni de mediciones: este registro es un análisis de impacto, no un resultado de ejecución.

**Impacto sobre la alternativa seleccionada.** El cambio afecta principalmente al componente independiente que se comunica con el proveedor, en concreto a dos puntos: la construcción de las solicitudes de envío y la interpretación de los errores para decidir si un envío se reintenta o se marca como fallido. No afecta a la generación ni al almacenamiento de constancias (RT2), porque ese flujo no invoca al proveedor, ni al registro de notificaciones pendientes y su restricción de unicidad por constancia (RT3, RT4), que son propios de ConstanciaUV. La Alternativa B permite localizar el cambio en un solo componente; en la Alternativa A habría afectado además al flujo de generación, que habría tenido que interpretar los errores del proveedor.

**Decisión.** La decisión se mantiene como propuesta (`proposed`) y su justificación original no cambia: las razones basadas en RT2, RT3 y RT4 siguen vigentes. La nueva información no contradice esas razones, sino que confirma que el aislamiento del proveedor es valioso. Lo que sí requiere evaluación posterior es la política de clasificación de errores: antes de implementar, habría que revisar la documentación del nuevo formato para definir qué errores son reintentables y cuáles definitivos.

**Consecuencia y comprobación a actualizar.** Se agregó a Consequences el riesgo de clasificación incorrecta de errores, y a Confirmation la comprobación 6 (pruebas de contrato con el formato nuevo y el anterior). Esa comprobación incluye verificar que un error desconocido se trate como reintentable con límite de intentos y que el cambio no abra una vía para duplicar solicitudes.
