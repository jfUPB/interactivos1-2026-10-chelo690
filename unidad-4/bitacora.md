# Unidad 4

## Bitácora de proceso de aprendizaje

### Actividad 01: Investigación
#### Observaciones:

- Al revisar los comportamientos y las acciones ejecutadas en el micro:bit se ven cambios dentro del SerialTerminal, como el cambio de false a true cuando se oprime un botón, a su vez como el cambio en la numeración al interactuar con el acelerómetro.

- Al interpretar los datos enviados utilizando caracteres ASCII se puede identificar la acción realizada por el usuario en el micro:bit, bien sea presionar un botón o mover el micro:bit en sí.

#### Pasos:

Se conecta el micro:bit, el cual ya está configurado con:

	uart.init(115200)

De esta manera se envían datos por comunicación serial, para luego abrir el SerialTerminal y configurar el baud rate: 115200.

Ahora comienzan a aparecer caracteres en el SerialTerminal, dando a entender qué comportamientos está capturando el micro:bit (presionar botones, movimiento, etc.), siendo los dos primeros valores (numéricos) el eje X y Y del acelerometro, y los valores booleanos (true y false) indican si el botón A o B están siendo presionados.

Ya estudiando el micro:bit y entendiendo qué carácter se refiere a qué comportamiento, se pasa a revisar el código nuevamente y ver qué datos del micro:bit se pueden aplicar a las acciones del arte generativo (por ejemplo, cambiar el color del patrón con el botón "B") y asignar dichos comportamientos a las manipulaciones hechas al micro:bit.

## Bitácora de aplicación 

### Actividad 02: Aplicacion

#### Pasos Para realizar la V2 del Microbit Adapter:

1. Revisar el nuevo formato de comunicacion para saber como realizar el nuevo adapter del microbit

		 $T:timestamp|X:valor|Y:valor|A:estado|B:estado|CHK:checksum

2. Crear el nuevo Archivo en la carpeta adapters. (MicrobitV2Adapter.js) y Realizar el codigo

		   import BaseAdapter from "./BaseAdapter.js";
		
		export default class MicrobitV2Adapter extends BaseAdapter {
		
		  parseLine(line) {
		
		    if (!line.startsWith("$")) return;
		
		    try {
		
		      line = line.substring(1);
		
		      let parts = line.split("|");
		
		      let data = {};
		
		      parts.forEach(part => {
		        let [key, value] = part.split(":");
		        data[key] = value;
		      });
		
		      let x = parseInt(data.X);
		      let y = parseInt(data.Y);
		      let a = parseInt(data.A);
		      let b = parseInt(data.B);
		      let chk = parseInt(data.CHK);
		
		      let calcChecksum =
		        Math.abs(x) +
		        Math.abs(y) +
		        a +
		        b;
		
		      if (calcChecksum !== chk) {
		        console.warn("Checksum incorrecto");
		        return;
		      }
		
		      this.onData({
		        x: x,
		        y: y,
		        btnA: a === 1,
		        btnB: b === 1
		      });
		
		    } catch (err) {
		
		      console.error("Error parseando la trama:", err);
		
		    }
		  }
		}

3. Importar el adapter en el archivo Bridge Server:

	   const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");

4. En el archivo BridgeServer cambiar Esta parte del codigo:

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
		
		  // if (DEVICE === "microbit-bin") {
		  //   const path = SERIAL_PATH ?? await findMicrobitPort();
		  //   if (!path) {
		  //     log.error("micro:bit not found. Use --serialPort to specify manually.");
		  //     process.exit(1);
		  //   }
		  //   return new MicrobitBinaryAdapter({ path, baud: BAUD });
		  // }
		
		  return new SimAdapter({ hz: SIM_HZ });
		}

   Por esta parte del codigo:

		async function createAdapter() {
		  if (DEVICE === "microbit") {
		    const path = SERIAL_PATH ?? await findMicrobitPort();
		    if (!path) {
		      log.error("micro:bit not found. Use --serialPort to specify manually.");
		      process.exit(1);
		    }
		    log.info(`micro:bit found at ${path}`);
		   return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
		  }
		
		  // if (DEVICE === "microbit-bin") {
		  //   const path = SERIAL_PATH ?? await findMicrobitPort();
		  //   if (!path) {
		  //     log.error("micro:bit not found. Use --serialPort to specify manually.");
		  //     process.exit(1);
		  //   }
		  //   return new MicrobitBinaryAdapter({ path, baud: BAUD });
		  // }
		
		  return new SimAdapter({ hz: SIM_HZ });
		}

Es exactamente igual solamente se cambia el  MicrobitAsciiAdapter por MicrobitV2Adapter, el cual es el acrhivo nuevo que implementa la nueva forma de comunicacion entre el microbit y el programa

5. Se agrega una funcion para validar el checksum y la integridad de los datos para evitar procesar datos corruptos

		  validarChecksum(x, y, a, b, chk) {
	    const calc = Math.abs(x) + Math.abs(y) + a + b;
	    return calc === chk;

Codigo final del MicrobitV2Adapter:
	
	import BaseAdapter from "./BaseAdapter.js";
	
	export default class MicrobitV2Adapter extends BaseAdapter {
	
	  validarChecksum(x, y, a, b, chk) {
	    const calc = Math.abs(x) + Math.abs(y) + a + b;
	    return calc === chk;
	  }
	
	  parseLine(line) {
	
	    if (!line.startsWith("$")) return;
	
	    try {
	
	      line = line.substring(1);
	
	      let parts = line.split("|");
	
	      let data = {};
	
	      parts.forEach(part => {
	        let [key, value] = part.split(":");
	        data[key] = value;
	      });
	
	      let x = parseInt(data.X);
	      let y = parseInt(data.Y);
	      let a = parseInt(data.A);
	      let b = parseInt(data.B);
	      let chk = parseInt(data.CHK);
	
	      if (!this.validarChecksum(x, y, a, b, chk)) {
	        console.warn("Checksum incorrecto");
	        return;
	      }
	
	      this.onData({
	        x: x,
	        y: y,
	        btnA: a === 1,
	        btnB: b === 1
	      });
	
	    } catch (err) {
	
	      console.error("Error parseando la trama:", err);
	
	    }
	  }
	}


## Bitácora de reflexión
