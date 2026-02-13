# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 1:

Estados de Programa: estado waitinInON y estado waitinInOff, significa lo que el programa esta haciendo en ese momento y cambia dependiendo de los eventos y acciones

Eventos: son aquellos que se ejecutan en el orden que lleva el programa y se muestra en el programa como Timeout, en el cual es un evento que inicia una cuenta atras dejando los leds en un estado prendido o apagado cierto tiempo.

Acciones: Las acciones son las que se realizan dentro del evento como prender y apagar los pixeles, o crear formas, realizar un sonido, etc.

### Actividad 2: 

#### Codigo del cemaforo modificado:

  from microbit import *
import utime
import music

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs


FILL = make_fill_images()


class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)


class EscapeTimer:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.time_pixels = 20
        self.max_pixels = 25
        self.min_pixels = 15
        self.current_pixel = 0
        self.myTimer = self.createTimer("Timeout", 1000)
        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def estado_config(self, ev):
        if ev == "ENTRY":
            self.current_pixel = self.time_pixels
            display.show(FILL[self.current_pixel])

        if ev == "A":
            if self.current_pixel < self.max_pixels:
                self.current_pixel += 1
                display.show(FILL[self.current_pixel])

        if ev == "B":
            if self.current_pixel > self.min_pixels:
                self.current_pixel -= 1
                display.show(FILL[self.current_pixel])

        if ev == "S":
            self.time_pixels = self.current_pixel
            self.transicion_a(self.estado_armed)

    def estado_armed(self, ev):
        if ev == "ENTRY":
            self.current_pixel = self.time_pixels
            display.show(FILL[self.current_pixel])
            self.myTimer.start(1000)

        if ev == "Timeout":
            if self.current_pixel > 0:
                self.current_pixel -= 1
                display.show(FILL[self.current_pixel])
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_alarm)

        if ev == "B":
            self.transicion_a(self.estado_config)

    def estado_alarm(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            for i in range(3):
                music.play(music.POWER_DOWN, wait=False)

        if ev == "A":
            self.time_pixels = 20
            self.transicion_a(self.estado_config)


task = EscapeTimer()

while True:
    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)



  #### Diagrama UML:

[![](https://img.plantuml.biz/plantuml/dsvg/SoWkIImgAStDuUMArefLqDMrKmZApyhdvUBb0igKf89v2jMyNBK8eR3KefHKD377tCIYp9mSX5AmFf1n4DLM2Y6PkQdvfIMyN101a1HS4o5PHrukE0_cH6HDl5mEgNafO5y00000)](https://editor.plantuml.com/uml/SoWkIImgAStDuUMArefLqDMrKmZApyhdvUBb0igKf89v2jMyNBK8eR3KefHKD377tCIYp9mSX5AmFf1n4DLM2Y6PkQdvfIMyN101a1HS4o5PHrukE0_cH6HDl5mEgNafO5y00000)

### Actividad 3:

Es dificil leer la maquina de estados en algunos casos, muchas veces no entiendo bien el flujo.

## Bitácora de aplicación 

### Actividad 4:

#### Maquina de Estados: 

[![](https://img.plantuml.biz/plantuml/dsvg/PP71JiCm38RlbVeEFWvG15mdWTPj1I6n5Qr8Eo24J6f1e4sgn9buBnw15wDfDvg9c_L_l_tRoSmnMlPDdIQik2R8dSIM7bL35WGqIbepVLMS9cdoTFeCGbp3ebZVtDq6PQXW2jc7TqnG4R2YfZKHcXl--TQGGTTvzb-V1rr4UlcEdnH4fPLKQAES49vjLldoppOfJm8_Y0jFcX4ilLboQeSZ2GSPpo2nGhXq8uZNaAWbrKFamEF4muXpQDKNrK8ScUwxkZFr2AxW8eRZpVtalNNboR75BhN67LaSIMcqgWmy5Djyyx8ijkknxMXS1fFkJkB-3MQag_uVq-GN)](https://editor.plantuml.com/uml/PP71JiCm38RlbVeEFWvG15mdWTPj1I6n5Qr8Eo24J6f1e4sgn9buBnw15wDfDvg9c_L_l_tRoSmnMlPDdIQik2R8dSIM7bL35WGqIbepVLMS9cdoTFeCGbp3ebZVtDq6PQXW2jc7TqnG4R2YfZKHcXl--TQGGTTvzb-V1rr4UlcEdnH4fPLKQAES49vjLldoppOfJm8_Y0jFcX4ilLboQeSZ2GSPpo2nGhXq8uZNaAWbrKFamEF4muXpQDKNrK8ScUwxkZFr2AxW8eRZpVtalNNboR75BhN67LaSIMcqgWmy5Djyyx8ijkknxMXS1fFkJkB-3MQag_uVq-GN)
#### Codigo:

    from microbit import *
    import utime
    import music

    def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

    FILL = make_fill_images()

    class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

    class EscapeTimer:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.time_pixels = 20
        self.max_pixels = 25
        self.min_pixels = 15
        self.current_pixel = 0
        self.myTimer = self.createTimer("Timeout", 1000)
        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def estado_config(self, ev):
        if ev == "ENTRY":
            self.current_pixel = self.time_pixels
            display.show(FILL[self.current_pixel])
        if ev == "A":
            if self.current_pixel < self.max_pixels:
                self.current_pixel += 1
                display.show(FILL[self.current_pixel])
        if ev == "B":
            if self.current_pixel > self.min_pixels:
                self.current_pixel -= 1
                display.show(FILL[self.current_pixel])
        if ev == "S":
            self.time_pixels = self.current_pixel
            self.transicion_a(self.estado_armed)

    def estado_armed(self, ev):
        if ev == "ENTRY":
            self.current_pixel = self.time_pixels
            display.show(FILL[self.current_pixel])
            self.myTimer.start(1000)
        if ev == "Timeout":
            if self.current_pixel > 0:
                self.current_pixel -= 1
                display.show(FILL[self.current_pixel])
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_alarm)
        if ev == "B":
            self.transicion_a(self.estado_config)

    def estado_alarm(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            for i in range(3):
                music.play(music.POWER_DOWN, wait=False)
        if ev == "A":
            self.time_pixels = 20
            self.transicion_a(self.estado_config)

     task = EscapeTimer()

    while True:
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")
    task.update()
    utime.sleep_ms(20)

## Bitácora de reflexión

#### Actividad 5:

Explicacion: Utilice los conceptos y codigos de la actividad 3 de la unidad 1 para configurar el microbit e importar los HTML correctos, luego copie el codigo del editor y lo realice como p5.js con Ayuda de la IA, revisando los HTML, librerias y eventos. 

Codigo p5.js :

let fillImages = [];
let timePixels = 20;
let maxPixels = 25;
let minPixels = 15;
let currentPixel = 0;
let state = "config";
let timerDuration = 1000;
let lastTime = 0;


let port;
let reader;


function makeFillImages() {
  for (let n = 0; n <= 25; n++) {
    let img = [];
    for (let i = 0; i < 25; i++) {
      img.push(i < n ? 255 : 50); // 255=on, 50=off
    }
    fillImages.push(img);
  }
}

function setup() {
  createCanvas(250, 250);
  makeFillImages();
  currentPixel = timePixels;
  lastTime = millis();

  
  let btn = createButton("Conectar Micro:bit");
  btn.position(10, height + 10);
  btn.mousePressed(connectMicrobit);
}

function draw() {
  background(0);

  
  let cellSize = width / 5;
  let img = fillImages[currentPixel];
  for (let y = 0; y < 5; y++) {
    for (let x = 0; x < 5; x++) {
      fill(img[y * 5 + x]);
      rect(x * cellSize, y * cellSize, cellSize, cellSize);
    }
  }


  if (state === "armed") {
    if (millis() - lastTime >= timerDuration) {
      lastTime = millis();
      if (currentPixel > 0) {
        currentPixel--;
      } else {
        state = "alarm";
        playAlarm();
      }
    }
  }
}


async function connectMicrobit() {
  port = await navigator.serial.requestPort();
  await port.open({ baudRate: 115200 });
  reader = port.readable.getReader();
  readLoop();
}


async function readLoop() {
  while (true) {
    const { value, done } = await reader.read();
    if (done) {
      reader.releaseLock();
      break;
    }
    if (value) {
      let char = String.fromCharCode(...value);
      handleEvent(char);
    }
  }
}


function handleEvent(ev) {
  if (ev === "A") {
    if (state === "config") {
      if (currentPixel < maxPixels) currentPixel++;
    } else if (state === "alarm") {
      timePixels = 20;
      state = "config";
      currentPixel = timePixels;
    }
  }

  if (ev === "B") {
    if (state === "config") {
      if (currentPixel > minPixels) currentPixel--;
    } else if (state === "armed") {
      state = "config";
      currentPixel = timePixels;
    }
  }

  if (ev === "C") {
    if (state === "config") {
      timePixels = currentPixel;
      state = "armed";
      lastTime = millis();
    }
  }

 
  if (ev === "h") {
    console.log("HEART received!");
  }
}


function playAlarm() {
  console.log("ALARM!");
  
}
