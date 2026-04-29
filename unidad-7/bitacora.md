# Unidad 7

---

## Bitácora de proceso de aprendizaje

### Actividad 01 

#### ¿Qué diferencia hay entre un evento musical y un mensaje de control?

Un evento musical es algo que ocurre en un instante del tiempo.
Strudel genera estos eventos con un `timestamp`, el sistema sabe
exactamente cuando debe dispararse la animación. Una vez que el instante pasa,
el evento desaparece de la cola.

Un mensaje de control, en cambio, no tiene timestamp ni momento de disparo.
Cuando Open Stage Control envía un mensaje OSC como `/rgb_1` con valores de
color, ese mensaje está diciendo "a partir de ahora, este parámetro vale esto".

---

#### ¿Qué quiere decir que un parámetro del sistema sea persistente?

Un parámetro persistente es aquel que, una vez modificado, mantiene su valor
mientras no reciba una nueva actualización.

Esto es distinto a una animación, que existe por un tiempo
limitado determinado por `delta` y luego desaparece.

---

#### ¿Qué partes del sistema de la unidad 6 permanecen intactas en este nuevo caso?

La lógica completa de Strudel se mantuvo sin modificar: el `StrudelAdapter`, el
`bridgeServer.js`, la conexión `BridgeClient` al puerto 8081, 
la cola `scheduledEvents`, el ciclo`draw()`, y todas las
funciones de dibujo. La única capa que se extendió fue el `sketch.js` para
recibir un segundo `BridgeClient`, y se agregó el `OpenStageControlAdapter`.

---

#### Si Open Stage Control fuera "el dispositivo", ¿cuál sería su protocolo?

El protocolo de Open Stage Control es OSC (Open Sound Control) sobre UDP. Los
mensajes tienen una dirección que identifica el parámetro y una lista de argumentos 
numéricos (`[255, 120, 30]`). La aplicación lanza su servidor HTTP en el puerto 8086 
y envía los mensajes OSC al destino configurado en `launcherConfig.config`.

---

#### ¿Qué parte de ese protocolo te interesa conservar y cuál te gustaría normalizar?

Me interesa conservar la dirección OSC porque es el identificador del parámetro
que se está controlando — `/rgb_1` dice exactamente qué parámetro cambiar.

---

#### ¿Por qué no conviene procesar un mensaje OSC igual que un mensaje de Strudel?

Porque tienen expresiones distintas. Si tratara un mensaje OSC como
un evento de Strudel, intentaría buscarlo en la cola por `timestamp`, pero los
mensajes OSC no tienen `timestamp` 

---

#### ¿Qué variables del sistema deberían vivir como estado persistente y no como evento efímero?

Todo lo que representa una configuración activa del sistema visual: los colores
asociados a cada familia de sonidos (`oscState.bd`, `oscState.hh`), la velocidad
de las animaciones si se controlara con un slider, el nivel de opacidad del fondo,
o cualquier otro parámetro que el usuario quiera ajustar en tiempo real y que
deba mantenerse entre un golpe y el siguiente.

---

#### ¿Qué componentes de la arquitectura necesitas conservar obligatoriamente?

- El `StrudelAdapter` y su integración en `bridgeServer.js`
- El `BridgeClient` conectado al puerto 8081 para Strudel
- La cola `scheduledEvents` con ordenamiento por `timestamp`
- El ciclo `draw()` con `processScheduledEvents()` y `cleanupAnimations()`
- Todas las funciones de dibujo de la unidad 6
- La `FSMTask` con sus estados `estado_esperando` y `estado_corriendo`

---

#### ¿Qué nuevas estructuras de estado necesitas introducir para soportar control paramétrico?

Se introdujo el objeto `oscState`, que se encuentra en el sketch y actúa
como el estado persistente del sistema:

```javascript
let oscState = {
  bd:    [255, 0,   80],
  hh:    [255, 255, 0],
  sd:    [0,   200, 255],
  cp:    [100, 220, 255],
  other: [200, 200, 200],
};
```

---

## Bitácora de aplicación 

### Actividad 02 

#### ¿Cómo configuré Open Stage Control?

Open Stage Control se lanza con el archivo `launcherConfig.config`:
```json
{"send": ["127.0.0.1:9000"], "port": 8086}
```
Esto indica que envía mensajes OSC por UDP al puerto 9000 y sirve su interfaz
web en el puerto 8086. La interfaz (`OSCUI.json`) tiene dos widgets de color RGB:
- `rgb_1`: controla el color de la familia `bd` (kick)
- `rgb_2`: controla el color de la familia `hh` (hihat)

---

#### ¿Qué widgets decidí usar y por qué?

Se uso widgets de color RGB porque permiten controlar los tres canales de
color simultáneamente con un solo gesto. Elegí controlar el kick y el hihat.
---

#### ¿Qué estructura final de mensaje decidí usar para los controles?

El `OpenStageControlAdapter` normaliza cada mensaje OSC a este formato antes de
entregarlo al `bridgeServer.js`:

```javascript
{
  type: "osc",
  address: "rgb_1",        // sin barra inicial
  args: [255, 120, 30],   // valores redondeados a entero
  t: Date.now()
}
```

---

#### ¿Cómo conecté bridgeClient.js, FSMTask, updateLogic y drawRunning?

El segundo `BridgeClient` conecta al puerto 8082 y se abre automáticamente
sin botón. Cuando llega un mensaje OSC, el `bridgeOsc.onOsc()` lo convierte en
un evento `OSC` que se postea a la `FSMTask`. Dentro de `estado_corriendo`, el
evento `OSC` llama a `updateOsc()` que actualiza `oscState`. La función
`drawRunning()` no sabe nada de OSC, solo lee `activeAnimations` cuyos colores
ya fueron resueltos por `getColorForFamily()` al momento del disparo.

```
Open Stage Control → UDP :9000 → OpenStageControlAdapter
→ bridgeServer (:8082) → BridgeClient → postEvent(OSC)
→ FSMTask.updateOsc() → oscState actualizado
→ getColorForFamily() lo lee al disparar cada animación
```

---

#### ¿Cómo integré ambas fuentes de datos en el mismo frontend?

El `sketch.js` tiene dos `BridgeClient` independientes:

```javascript
let bridge;     // Strudel  — puerto 8081
let bridgeOsc;  // OSC      — puerto 8082
```

Los dos flujos convergen en `PainterTask` sin interferirse:
- Strudel → evento `DATA` → `updateLogic()` → `scheduledEvents` → animación
- OSC → evento `OSC` → `updateOsc()` → `oscState` → color en tiempo real

El único punto donde se encuentran es `getColorForFamily()`: cuando Strudel
dispara un bombo, esa función consulta `oscState.bd` para saber qué color usar
en ese momento. Así el color es siempre el más reciente que configuró el usuario
con Open Stage Control.

---

#### ¿Qué pruebas hice para verificar que el control paramétrico no rompe la sincronización de Strudel?

1. Se arranco Strudel con el patrón corriendo y verifiqué que las animaciones
   aparecen sincronizadas con el audio.
2. Con Strudel corriendo, moví el color picker, el color cambió inmediatamente
   sin interrumpir las animaciones ni desincronizar el ritmo.
3. Detuve y volví a arrancar Strudel — el sistema reconectó y el color que había
   configurado con OSC se mantuvo en `oscState`.
4. Usé `--verbose` en ambas terminales para confirmar que los mensajes llegaban
   correctamente a cada bridge sin interferencia entre ellos.

---

#### ¿Qué problemas encontré y cómo los resolví?

**Problema 1:** El `bridgeServer.js` pasaba `strudelPort` al constructor del
`StrudelAdapter` pero el adapter esperaba `port`. El servidor nunca levantaba en
el puerto correcto. Solución: cambiar `{ strudelPort: STRUDEL_PORT }` por
`{ port: STRUDEL_PORT }`.

**Problema 2:** Los mensajes de Strudel llegaban al navegador sin el campo
`payload`. Con logs de debug se confirmó que el objeto llegaba como
`{ type: "strudel", timestamp: ... }` sin `payload`. Se corrigió el flujo de
broadcast para que el objeto completo llegara intacto.

**Problema 3:** El `BridgeClient` no pasaba los mensajes de tipo `"strudel"` al
callback `onData`. Solo reconocía `"microbit"`. Solución: agregar
`msg.type === "strudel"` a la condición.

**Problema 4:** Open Stage Control envía valores RGB como decimales (`159.02`).
Solución: aplicar `Math.round()` en el `OpenStageControlAdapter`.

---

## Bitácora de reflexión

### Actividad 03 

#### 1. Diagrama del flujo 

[![](https://img.plantuml.biz/plantuml/svg/ZLNHRjem57r7uX-kkeU1j2B2rfKXhMf8qgbILpe2snxsOk8BJHlio7QArcbIFwB_ilTzIhzarquePANQaWY8ppt7n_TUcsDjc3B5Ccisz7KgSgRO4cOikLueMGWUo4mgU35trtO8npahCdCYXQbYF6RlPsBYz1R1UxsD-ah9LSLzuwvjL5yoBbUfMC2SPHgteSgO4gYWtspK84mC4uiCuKUp0J1yieu3Upj8Aewg6kwxomxlMC_F-XH2ycVLv_C-Ua_KX_xXWNVuV-hC9gMKtmfUfwSSB7FfLLoJ6bfkcB85yHdb7EPPSINj3ywCrF1mTmzJaNB6uT0INiB3_HqzS3ADpYTHNBqs6svbUzgeWcFMIP9leZKZDvhI_FiNTAHEMkts7Z0DyMGy3QEpDMMKeu3Oi0L2GFwKp8YQ2eDgpcvXXJFB7_Ix_RSbVHG7DID-w5zCel76oQQCUt5fn-Si9xtEbIPf8TMIPUaj0xn1OTTeS9bBHIDSwifY9LeJiqQTpyhcCwfB59fTeJFwwFXHRyluS7mQVeWWZiQVdx4_KobVO8pgPZtD_Zx7cpH1g19jSxFI5r8Pe0nvvNp3sFjV6IfBNQAtbkZjyXgSdLBLTteTkac2BAMSkjhRLwkT5wYCtBONUFzcZafZ6_BQM50tTX9_tEM6XYTCNrk92-QQoRwIUf5JN-EPNAHEltoboKfrejNKetZYXwDT4z3USjkbwY5IriabyY937jaXxTpJzZFf4kcKNERb39-cqJIgAKM4SWavAfrgGrGaCQeBjPfn2PpnLt1v-GeqVITvfzDPRoD_4jNSZgFpC1hlSiiLMuqB8UX4RBM_YJ4AEc3W9jK5SosK7r1VEDLIeoORZCwZhwVqiQbxjOLXV__eSQ_00Bx4RgDscmrgUGTkMf_WQNW1WtVTRXTQTc_OIHx9DfZwbgAa29bXhROJgSEz6jniM8930alyWhDO9jTed0dFmhqAxZs1NHTPdWsmJJIWQcdZQjSA_OdVjd1l9xmsRySjxcfA4Nzrm2R0bSfwhwfgoIN9ebNP1f0KJ50dqFgm8-IgrOQZzvl-iLctCZNA0ijJ_eGUjaopxsLYZgd0zA_y1m00)](https://editor.plantuml.com/uml/ZLNHRjem57r7uX-kkeU1j2B2rfKXhMf8qgbILpe2snxsOk8BJHlio7QArcbIFwB_ilTzIhzarquePANQaWY8ppt7n_TUcsDjc3B5Ccisz7KgSgRO4cOikLueMGWUo4mgU35trtO8npahCdCYXQbYF6RlPsBYz1R1UxsD-ah9LSLzuwvjL5yoBbUfMC2SPHgteSgO4gYWtspK84mC4uiCuKUp0J1yieu3Upj8Aewg6kwxomxlMC_F-XH2ycVLv_C-Ua_KX_xXWNVuV-hC9gMKtmfUfwSSB7FfLLoJ6bfkcB85yHdb7EPPSINj3ywCrF1mTmzJaNB6uT0INiB3_HqzS3ADpYTHNBqs6svbUzgeWcFMIP9leZKZDvhI_FiNTAHEMkts7Z0DyMGy3QEpDMMKeu3Oi0L2GFwKp8YQ2eDgpcvXXJFB7_Ix_RSbVHG7DID-w5zCel76oQQCUt5fn-Si9xtEbIPf8TMIPUaj0xn1OTTeS9bBHIDSwifY9LeJiqQTpyhcCwfB59fTeJFwwFXHRyluS7mQVeWWZiQVdx4_KobVO8pgPZtD_Zx7cpH1g19jSxFI5r8Pe0nvvNp3sFjV6IfBNQAtbkZjyXgSdLBLTteTkac2BAMSkjhRLwkT5wYCtBONUFzcZafZ6_BQM50tTX9_tEM6XYTCNrk92-QQoRwIUf5JN-EPNAHEltoboKfrejNKetZYXwDT4z3USjkbwY5IriabyY937jaXxTpJzZFf4kcKNERb39-cqJIgAKM4SWavAfrgGrGaCQeBjPfn2PpnLt1v-GeqVITvfzDPRoD_4jNSZgFpC1hlSiiLMuqB8UX4RBM_YJ4AEc3W9jK5SosK7r1VEDLIeoORZCwZhwVqiQbxjOLXV__eSQ_00Bx4RgDscmrgUGTkMf_WQNW1WtVTRXTQTc_OIHx9DfZwbgAa29bXhROJgSEz6jniM8930alyWhDO9jTed0dFmhqAxZs1NHTPdWsmJJIWQcdZQjSA_OdVjd1l9xmsRySjxcfA4Nzrm2R0bSfwhwfgoIN9ebNP1f0KJ50dqFgm8-IgrOQZzvl-iLctCZNA0ijJ_eGUjaopxsLYZgd0zA_y1m00)

---

#### 2. Tabla comparativa: Unidades 4, 5, 6 y 7

| Aspecto | Unidad 4 | Unidad 5 | Unidad 6 | Unidad 7 |
|---|---|---|---|---|
| Fuente de datos | micro:bit serial | micro:bit serial | Strudel (navegador) | Open Stage Control |
| Formato del mensaje | CSV texto ASCII | Binario con framing | JSON por WebSocket | OSC sobre UDP |
| Tipo de dato que produce | Sensor continuo | Sensor continuo | Evento temporizado | Parámetro de control |
| Problema técnico principal | Parsear texto y validar | Sincronizar bytes con framing | Sincronizar con timestamp musical | Mantener estado persistente |
| Lugar donde ocurre la traducción | MicrobitASCIIAdapter | MicrobitBinaryAdapter | StrudelAdapter | OpenStageControlAdapter |
| Papel del tiempo | No aplica | No aplica | Central — timestamp decide cuándo dibujar | No aplica — sin timestamp |
| Relación con el estado del sistema | Actualiza rxData directamente | Actualiza rxData directamente | Alimenta cola de eventos efímeros | Actualiza oscState persistente |

---

#### 3. ¿Por qué Open Stage Control no debe tratarse igual que Strudel dentro de la arquitectura?

Porque producen tipos de datos con expresiones distintas. Strudel
produce eventos con `timestamp`, se procesan y desaparecen. 
Open Stage Control produce actualizaciones constantemente
sin `timestamp`.

---

#### 4. Justificación de los controles elegidos

**Control 1 — rgb_1 (color del kick `bd`):**
El kick es el elemento más prominente visualmente ocupa el centro de la
pantalla con un círculo que crece. 

**Control 2 — rgb_2 (color del hihat `hh`):**
El hihat aparece como cuadrados dispersos por toda la pantalla. 
controla el color de todos lor recuadros en conjunto y es una opcion 
visual llamativa para el usuario

---

#### 5. Si tuvieras que integrar una tercera fuente de control en el futuro, ¿qué conservarías y qué extenderías?

Conservaría toda la arquitectura existente sin modificar: los dos adapters, los
dos `bridgeServer`, los dos `BridgeClient`, la `FSMTask`, la cola de eventos y
las funciones de dibujo.

Lo único que agregaría sería un nuevo adapter. 
---
