# Unidad 6

---

## Bitácora de proceso de aprendizaje

### Actividad 01

#### ¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?

Recibir un mensaje significa que los datos llegaron al sistema — el WebSocket lo
capturo y esta en la memoria. Ejecutarlo significa que el sistema
reacciona a ese dato: dibuja algo, cambia un estado y crea una animación.

---

#### ¿Por qué un sistema audiovisual puede necesitar `timestamp` además de los datos del evento?

Porque el audio y el video tienen relojes diferentes y la red introduce latencia
variable. Sin `timestamp`, el sistema no sabe cuándo debería ocurrir la respuesta
visual — solo sabe qué ocurrió.

Strudel calcula con precisión cuándo debe sonar cada nota y lo incluye en el
mensaje como `timestamp`. Si la aplicación visual usa ese mismo `timestamp` para
decidir cuándo dibujar, la animación queda sincronizada con el audio aunque el
mensaje haya llegado un poco tarde por la red.

---

#### ¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos?

La logica de capas se mantiene completamente. El patron Adapter sigue siendo el
mismo — en las unidades 4 y 5 se crearon `MicrobitASCIIAdapter` y
`MicrobitBinaryAdapter`. En la unidad 6 se creo `StrudelAdapter`. El mecanismo
es identico: extender `BaseAdapter`, implementar `connect()`, `disconnect()` y
`getConnectionDetail()`, y llamar a `this.onData?.()` con datos normalizados.

El `bridgeServer.js` no cambió su estructura. Sigue levantando un servidor
WebSocket, creando un adapter según el `--device` que se le pase, y
retransmitiendo los datos al frontend.

El `BridgeClient` sigue siendo el intermediario entre el bridge y el `sketch.js`.
La `FSMTask` sigue coordinando estados. Lo que cambio es solo el adapter.

---

#### Si Strudel fuera "el dispositivo", ¿cuál sería su protocolo?

El protocolo de Strudel es WebSocket sobre TCP. Cuando se usa `.osc()` en el
patrón, Strudel envía mensajes JSON por WebSocket al puerto 8080.

```json
{
  "address": "/dirt/play",
  "args": ["s", "tr909bd", "delta", 0.5, "cps", 0.5, "cycle", 15.25],
  "timestamp": 1774966984435.2805
}
```

---

#### ¿Qué variables mínimas necesitarías extraer para construir una visualización útil?

Las variables mínimas son tres:

- `s` (sound): el nombre del instrumento, que permite clasificar el tipo de
  animación a mostrar (bombo, caja, hihat, etc.)
- `timestamp`: el momento exacto en que debe ocurrir la animación
- `delta`: la duración del ciclo rítmico, que define cuánto tiempo dura la
  animación antes de desaparecer

---

#### ¿Qué problema resuelve la cola de eventos?

La cola de eventos resuelve el problema de la sincronizacion. Sin ella,
cada evento se ejecutaría en el momento en que llega al fronstend. Con la cola, los eventos se
almacenan con su `timestamp` y se ejecutan exactamente cuando `Date.now()`
alcanza ese valor.

---

#### ¿Por qué esta capa no pertenece al bridge sino al lado que interpreta el evento?

Porque el bridge es una capa de transporte — su única responsabilidad es mover
datos de un lugar a otro. No debe saber qué significan esos datos ni cuándo deben
ejecutarse. 

La cola pertenece al frontend porque es ahí donde se sabe qué hacer con el evento
— cómo dibujarlo, cuánto tiempo mantenerlo visible, qué color usar. El bridge
entrega el evento con su timestamp intacto y el frontend decide cuándo y cómo
reaccionar.

---

#### ¿Qué papel cumple el Adapter en U4 y U5?

En las unidades 4 y 5 el Adapter cumple el papel de traductor entre el protocolo
del dispositivo físico y el codigo que espera el resto del
sistema.

En la unidad 4, `MicrobitASCIIAdapter` lee líneas de texto con formato CSV del
puerto serial y las convierte en objetos `{ x, y, btnA, btnB }`. En la unidad 5,
`MicrobitBinaryAdapter` lee paquetes y hace la misma conversión. 
En ambos casos, el resto del sistema no sabe nada del protocolo
original.

---

#### ¿Qué Adapter necesitas para que los eventos de Strudel no entren "crudos"?

Necesito un `StrudelAdapter` que reciba los mensajes JSON de Strudel, extraiga
los campos relevantes de los `args`, clasifique el sonido en (`bd`, `sd`, `hh`, etc.) 
y entregue al sistema un objeto normalizado con `type`, `timestamp` y `payload`.
Así el resto del sistema nunca ve el formato crudo de Strudel.

---

## Bitácora de aplicación 

### Actividad 02

#### Que se implemento?

**StrudelAdapter.js** adapter que actúa como servidor WebSocket en el puerto
8080. Cuando Strudel se conecta y envía eventos, el adapter los normaliza con
`_parseEvent()` y llama a `this.onData?.()`. La normalización convierte los args
 a un objeto plano y clasifica el sonido en una familia visual con `_getFamily()`.

**bridgeServer.js** se registra el `StrudelAdapter` con `--device strudel`. El
bridge instancia el adapter con `{ port: STRUDEL_PORT }` y cuando llega un evento
de tipo `"strudel"` lo retransmite directamente al frontend con `broadcast(wss, d)`.

**bridgeClient.js** — agregué que los mensajes de tipo `"strudel"` pasen al
callback `onData`, igual que los mensajes de tipo `"microbit"`:
```javascript
if (msg.type === "microbit" || msg.type === "strudel") {
  this._onData?.(msg);
}
```

**sketch.js** se implemento `scheduledEvents` como cola ordenada por `timestamp`.
En `updateLogic()` los eventos entran a la cola. En `processScheduledEvents()`
se procesan cuando `Date.now()` alcanza su timestamp. En `drawRunning()` solo se
lee el estado ya calculado y se dibuja — sin parsear mensajes de red.

**Comando de arranque:**
```bash
node bridgeServer.js --device strudel --wsPort 8081 --strudelPort 8080 --verbose
```

---

## Bitácora de reflexión

### Actividad 03 Consolidación y metacognición

#### 1. Diagrama del flujo de datos del sistema

<img width="491" height="880" alt="Diagrama de flujo sfi1" src="https://github.com/user-attachments/assets/e72774c2-62d0-481c-ac67-42e3e541045f" />

---

#### 2. Tabla comparativa: Unidades 4, 5 y 6

| Aspecto | Unidad 4 | Unidad 5 | Unidad 6 |
|---|---|---|---|
| Fuente de datos | micro:bit por serial | micro:bit por serial | Strudel en el navegador |
| Formato del mensaje | CSV en texto ASCII | Binario con framing (6 bytes) | JSON por WebSocket |
| Problema técnico principal | Parsear texto y validar valores | Sincronizar bytes, detectar inicio de paquete | Sincronizar animación con tiempo musical |
| Mecanismo de validación o control | Validar número de campos y rango numérico | Detectar byte de inicio (0xAA), validar tamaño | Verificar que args sea Array y timestamp sea Number |
| Lugar donde ocurre la traducción | MicrobitASCIIAdapter | MicrobitBinaryAdapter | StrudelAdapter |
| Papel del tiempo o sincronización | No aplica — se procesa al llegar | No aplica — se procesa al llegar | Central — el timestamp decide cuándo se dibuja |

---

#### 3. ¿Por qué esta unidad sigue perteneciendo a la misma arquitectura del curso?

Porque la lógica de diseño no cambió. Sigue existiendo una fuente externa de
datos, un adapter que la traduce, un bridge que transporta, un cliente que recibe
y una capa de render que dibuja. Lo que cambio fue del dato.

---

#### 4. ¿Qué decisiones tomé para traducir eventos musicales en visualidad?

El tiempo de vida de cada animación está determinado por `delta * 1000` ms —
así la animación dura exactamente lo que dura el ciclo rítmico del evento. Esto
hace que la visualización respire al ritmo de la música.
---

#### 5. Si tuvieras que integrar una tercera aplicación en el futuro, ¿qué conservarías y qué cambiarías?

Conservaría toda la arquitectura existente sin modificar: `bridgeServer.js`,
`BridgeClient`, `FSMTask` y la lógica de `draw()`. También conservaría el patrón
de la cola de eventos si la nueva fuente también tiene `timestamp`.

Lo único que agregaría es un nuevo adapter.

---
