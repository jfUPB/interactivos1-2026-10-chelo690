# Unidad 5
## Bitácora de proceso de aprendizaje

### Actividad 1:

#### ¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?

R\ EL formnato binario es menos pesado y utiliza menos caracteres, lo que puede dar mayor velocidad en recogida y envio de datos sin embargo para el ususario es mas dificil utilizarlo

#### Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.)

R\ utilizando la calculadora en modalidad de prgoramador seria asi: XValue en hexadecimal 0x01F4 y en bytes 01 F4 asi mismo yValue seria en Hexadecimal 0x020C y en bytes 02 0C, ahora el aState true es 01 y false 00 por ende seria 01 y bState 00

el resultado quedaria 01 F4 02 0C 01 00

#### ¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización? (Pista: piensa en qué rol cumplía el carácter \n.)

R\ Se delimitaba el programa con \n para cada tramo de informacion

#### ¿Por qué en binario no podemos usar \n como delimitador?

R\ en Ascci es un caracter unico y en binario se puede dar como cualquier otro numero por lo que es imposible diferenciarlos.

## Bitácora de aplicación 

En BridgeServer.js se quitan la mencion //const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter"); y se hace lo mismo en createAdapter con if (DEVICE === "microbit-bin")

luego se implementa nuevamente el codigo adapter con varias modificaciones como string a buffer, en vez de terminar las lineas con \n se termina con 0xAA y tambien la forma de determinar el checksum: (sum % 256) === frame[7]

<DETAILS>
<summary><b>Binary Adapter(con errores):</b></summary>

``` js
	
	const { SerialPort } = require("serialport");
	const BaseAdapter = require("./BaseAdapter");
	
	class MicrobitBinaryAdapter extends BaseAdapter {
	  constructor({ path, baud = 115200, verbose = false } = {}) {
	    super();
	    this.path = path;
	    this.baud = baud;
	    this.verbose = verbose;
	
	    this.port = null;
	    this.buffer = Buffer.alloc(0);
	  }
	
	  async connect() {
	    if (this.connected) return;
	
	    if (!this.path) {
	      throw new Error("serialPort is required");
	    }
	
	    this.port = new SerialPort({
	      path: this.path,
	      baudRate: this.baud,
	      autoOpen: false,
	    });
	
	    await new Promise((res, rej) => {
	      this.port.open(err => err ? rej(err) : res());
	    });
	
	    this.connected = true;
	    this.onConnected?.(`serial ${this.path} @${this.baud}`);
	
	    this.port.on("data", (data) => this._processIncoming(data));
	    this.port.on("error", (e) => this._handleError(e));
	    this.port.on("close", () => this._handleClose());
	  }
	
	  async disconnect() {
	    if (!this.connected) return;
	
	    this.connected = false;
	
	    if (this.port?.isOpen) {
	      await new Promise((res, rej) => {
	        this.port.close(err => err ? rej(err) : res());
	      });
	    }
	
	    this.port = null;
	    this.buffer = Buffer.alloc(0);
	
	    this.onDisconnected?.("serial closed");
	  }
	
	  getConnectionDetail() {
	    return `serial ${this.path}`;
	  }
	
	  //  Entrada principal de datos
	  _processIncoming(chunk) {
	    this.buffer = Buffer.concat([this.buffer, chunk]);
	
	    while (true) {
	      const frame = this._extractFrame();
	      if (!frame) break;
	
	      if (!this._isValid(frame)) {
	        console.warn("Checksum incorrecto");
	        continue;
	      }
	
	      const data = this._decode(frame);
	      this.onData?.(data);
	    }
	  }
	
	  //  Busca un paquete válido en el buffer
	  _extractFrame() {
	    const HEADER = 0xAA;
	
	    let idx = this.buffer.indexOf(HEADER);
	
	    if (idx === -1) {
	      this.buffer = Buffer.alloc(0);
	      return null;
	    }
	
	    if (idx > 0) {
	      this.buffer = this.buffer.slice(idx);
	    }
	
	    if (this.buffer.length < 8) return null;
	
	    const frame = this.buffer.slice(0, 8);
	    this.buffer = this.buffer.slice(8);
	
	    return frame;
	  }
	
	  //  Validar checksum
	  _isValid(frame) {
	    let sum = 0;
	    for (let i = 1; i <= 6; i++) {
	      sum += frame[i];
	    }
	
	    return (sum % 256) === frame[7];
	  }
	
	  //  Convertir bytes a datos útiles
	  _decode(frame) {
	    return {
	      x: frame.readInt16BE(1),
	      y: frame.readInt16BE(3),
	      btnA: frame[5] === 1,
	      btnB: frame[6] === 1
	    };
	  }
	
	  _handleError(err) {
	    this.onError?.(String(err?.message || err));
	    this.disconnect();
	  }
	
	  _handleClose() {
	    if (!this.connected) return;
	
	    this.connected = false;
	    this.port = null;
	    this.buffer = Buffer.alloc(0);
	
	    this.onDisconnected?.("serial closed (event)");
	  }
	
	  async writeLine(line) {
	    if (!this.port?.isOpen) return;
	
	    await new Promise((res, rej) => {
	      this.port.write(line, err => err ? rej(err) : res());
	    });
	  }
	
	  async handleCommand(cmd) {
	    if (cmd?.cmd !== "setLed") return;
	
	    const clamp = (v, min, max) => Math.max(min, Math.min(max, Math.trunc(v)));
	
	    const x = clamp(cmd.x, 0, 4);
	    const y = clamp(cmd.y, 0, 4);
	    const v = clamp(cmd.value, 0, 9);
	
	    await this.writeLine(`LED,${x},${y},${v}\n`);
	  }
	}
```

</DETAILS>



<DETAILS>
<summary><b>Microbit:</b></summary>

``` py
	from microbit import *

uart.init(115200)
display.set_pixel(0, 0, 9)

HEADER = 0xAA

def clamp(value, min_v, max_v):
    return max(min_v, min(max_v, value))

def to_int16_be_bytes(val):
    if val < 0:
        val += 65536
    return [(val >> 8) & 0xFF, val & 0xFF]

def build_packet(x, y, btnA, btnB):
    x_bytes = to_int16_be_bytes(x)
    y_bytes = to_int16_be_bytes(y)

    payload = x_bytes + y_bytes + [btnA, btnB]
    checksum = sum(payload) % 256

    return bytes([HEADER] + payload + [checksum])

while True:
    # Leer sensores
    x = clamp(accelerometer.get_x(), -2048, 2047)
    y = clamp(accelerometer.get_y(), -2048, 2047)

    btnA = 1 if button_a.is_pressed() else 0
    btnB = 1 if button_b.is_pressed() else 0

    # Crear paquete
    packet = build_packet(x, y, btnA, btnB)

    # Enviar por UART
    uart.write(packet)

    sleep(100)
```
</DETAILS>

Al implementar el codigo por primera vez utilizando el microbit, encontre que no funcionaba de manera adecuada ya que estaba pintando sin presionar ningun boton, por lo que se realizaron varios cambios en el checksum y tambien en el export

<DETAILS>
<summary><b>Binary Adapter (corregido):</b></summary>

``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.verbose = verbose;

    this.port = null;
    this.buf = Buffer.alloc(0);
  }

  async connect() {
    if (this.connected) return;

    if (!this.path) {
      throw new Error("serialPort is required");
    }

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((res, rej) => {
      this.port.open(err => err ? rej(err) : res());
    });

    this.connected = true;
    this.onConnected?.(`serial ${this.path} @${this.baud}`);

    this.port.on("data", (data) => this._onChunk(data));
    this.port.on("error", (e) => this._handleError(e));
    this.port.on("close", () => this._handleClose());
  }

  async disconnect() {
    if (!this.connected) return;

    this.connected = false;

    if (this.port?.isOpen) {
      await new Promise((res, rej) => {
        this.port.close(err => err ? rej(err) : res());
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0);

    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial ${this.path}`;
  }

  //  Procesamiento de datos (VERSIÓN MEJORADA)
  _onChunk(dataChunk) {
    this.buf = Buffer.concat([this.buf, dataChunk]);

    while (this.buf.length >= 8) {

      const headerIndex = this.buf.indexOf(0xAA);

      if (headerIndex < 0) {
        this.buf = Buffer.alloc(0);
        return;
      }

      if (this.buf.length < headerIndex + 8) {
        return;
      }

      const frame = this.buf.slice(headerIndex, headerIndex + 8);

      this.buf = this.buf.slice(headerIndex + 8);

      try {
        const parsed = this._decodeFrame(frame);

        if (parsed && this.onData) {
          this.onData(parsed);
        }

      } catch (err) {
        if (this.verbose) {
          console.log("Bad binary packet:", err.message, frame);
        }
      }
    }
  }

  // Decodificación + validación en un solo lugar
  _decodeFrame(frame) {
    const xVal = frame.readInt16BE(1);
    const yVal = frame.readInt16BE(3);
    const btnA = frame[5];
    const btnB = frame[6];
    const receivedChecksum = frame[7];

    let checksumCalc = 0;
    for (let i = 1; i <= 6; i++) {
      checksumCalc += frame[i];
    }
    checksumCalc %= 256;

    if (checksumCalc !== receivedChecksum) {
      throw new Error("Checksum inválido");
    }

    return {
      x: xVal,
      y: yVal,
      btnA: btnA === 1,
      btnB: btnB === 1
    };
  }

  _handleError(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _handleClose() {
    if (!this.connected) return;

    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);

    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port?.isOpen) return;

    await new Promise((res, rej) => {
      this.port.write(line, err => err ? rej(err) : res());
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd !== "setLed") return;

    const clamp = (v, min, max) =>
      Math.max(min, Math.min(max, Math.trunc(v)));

    const x = clamp(cmd.x, 0, 4);
    const y = clamp(cmd.y, 0, 4);
    const v = clamp(cmd.value, 0, 9);

    await this.writeLine(`LED,${x},${y},${v}\n`);
  }
}

module.exports = MicrobitBinaryAdapter;

```
<DETAILS>
<summary><b>Bridge server:</b></summary>
	
``` js


//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbit-bin") {
     const path = SERIAL_PATH ?? await findMicrobitPort();
     if (!path) {
       log.error("micro:bit not found. Use --serialPort to specify manually.");
       process.exit(1);
     }
     return new MicrobitBinaryAdapter({ path, baud: BAUD });
   }

  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  };

  adapter.onData = (d) => {
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";

    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);

        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        
        try {
          await adapter.connect();
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        
        try {
          await adapter.disconnect();
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
<DETAILS>
<summary><b>Sketch:</b></summary>
	
``` js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            strokeWeight(0.75);
            background(255);
            console.log("Microbit ready to draw");
            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                prevA: false,
                prevB: false,
                ready: false
            };
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;
        this.rxData.x = map(data.x,-2048,2047,0,width);
        this.rxData.y = map(data.y,-2048,2047,0,height);
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        if (this.rxData.btnA && !this.prevA) {
            this.lineSize = random(50, 160);
            this.clickPosX = this.rxData.x;
            this.clickPosY = this.rxData.y;
            console.log("A pressed");
        }

        if (!this.rxData.btnB && this.prevB) {
            this.c = color(random(255), random(255), random(255), random(80, 100));
            console.log("B released");
        }

        this.prevA = this.rxData.btnA;
        this.prevB = this.rxData.btnB;
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(255);
    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        painter.postEvent({
            type: EVENTS.DATA, payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    let mb = painter.rxData;

        if (!mb.ready) return;

            // Dibujar SOLO mientras botón A esté presionado
            if (mb.btnA) {

                push();
                translate(width / 2, height / 2);

                // Resolución del polígono ← eje Y del acelerómetro
                let circleResolution = int(map(mb.y, 0, height, 2, 10));

                // Radio ← eje X del acelerómetro
                let radius = (mb.x - width / 2)*7;

                let angle = TAU / circleResolution;

                // Relleno activado con botón B
                if (mb.btnB) {
                    fill(34, 45, 122, 50);
                } else {
                    noFill();
                }

                beginShape();
                for (let i = 0; i <= circleResolution; i++) {
                    let x = cos(angle * i) * radius;
                    let y = sin(angle * i) * radius;
                    vertex(x, y);
                }
                endShape();

                pop();
            }

}


function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}

```
## Bitácora de reflexión
