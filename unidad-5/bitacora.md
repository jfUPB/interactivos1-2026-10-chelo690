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
<summary><b>Binary Adapter:</b></summary>

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
	
	    # Enviar
	    uart.write(packet)
	
	    sleep(100)
	
	module.exports = MicrobitBinaryAdapter;
```
</DETAILS>


## Bitácora de reflexión
