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

    class Semaforo:
    def __init__(self, _x, _y, _timeInRed, _timeInGreen, _timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.createTimer("Timeout", self.timeInRed)
        self.estado_actual = None
        self.transicion_a(self.estado_waitInRed)

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

    def clear(self):
        display.set_pixel(self.x, self.y, 0)
        display.set_pixel(self.x, self.y+1, 0)
        display.set_pixel(self.x, self.y+2, 0)

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x, self.y, 9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x, self.y, 0)
            self.transicion_a(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x, self.y+2, 9)
            self.myTimer.start(self.timeInGreen)
        if ev == "Timeout":
            display.set_pixel(self.x, self.y+2, 0)
            self.transicion_a(self.estado_waitInYellow)
        if ev == "A":
            self.myTimer.stop()
            display.set_pixel(self.x, self.y+2, 0)
            self.transicion_a(self.estado_waitInYellow)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x, self.y+1, 9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x, self.y+1, 0)
            self.transicion_a(self.estado_waitInRed)

    semaforo1 = Semaforo(0, 0, 2000, 1000, 500)

    while True:
    if button_a.is_pressed():
        semaforo1.post_event("A")
    semaforo1.update()
    utime.sleep_ms(20)

  #### Diagrama UML:

[![](https://img.plantuml.biz/plantuml/dsvg/SoWkIImgAStDuUMArefLqDMrKmZApyhdvUBb0igKf89v2jMyNBK8eR3KefHKD377tCIYp9mSX5AmFf1n4DLM2Y6PkQdvfIMyN101a1HS4o5PHrukE0_cH6HDl5mEgNafO5y00000)](https://editor.plantuml.com/uml/SoWkIImgAStDuUMArefLqDMrKmZApyhdvUBb0igKf89v2jMyNBK8eR3KefHKD377tCIYp9mSX5AmFf1n4DLM2Y6PkQdvfIMyN101a1HS4o5PHrukE0_cH6HDl5mEgNafO5y00000)

### Actividad 3:

Es dificil leer la maquina de estados en algunos casos, muchas veces no entiendo bien el flujo.

## Bitácora de aplicación 

### Actividad 4:

#### Maquina de Estados: 

[![](https://img.plantuml.biz/plantuml/dsvg/ZLBHQi8m57qlz1_kOnsAUnOcZTP9GTknLdoG8SLUj1XJQHAsCVRlXgHQxJ8nR-UUSsvE3l6vo2eX3zHrLayVqEiDOHn7h-7KTLn7SG9h33-k0-gSLKiIfc4qDSCQN1CmW94KecH0u5WXvvY3Lx1DXHd7pWEsKMFByLyRUPzF0cLATjaUmTDGoNwR-4RHIZ-E5r4QnCl8Z2_mbbHxq-A0fHHTE1PVI3aCuTbc8JDrZKN-OfVNbVvzwsrolTJI-repJHa6jsYr_OrctmNR0Yybow4FV5T-QhoNb5hjxM2a5QpcBSqdgdKpilHXUcZeE-z_A8kFBDT_zWG0)](https://editor.plantuml.com/uml/ZLBHQi8m57qlz1_kOnsAUnOcZTP9GTknLdoG8SLUj1XJQHAsCVRlXgHQxJ8nR-UUSsvE3l6vo2eX3zHrLayVqEiDOHn7h-7KTLn7SG9h33-k0-gSLKiIfc4qDSCQN1CmW94KecH0u5WXvvY3Lx1DXHd7pWEsKMFByLyRUPzF0cLATjaUmTDGoNwR-4RHIZ-E5r4QnCl8Z2_mbbHxq-A0fHHTE1PVI3aCuTbc8JDrZKN-OfVNbVvzwsrolTJI-repJHa6jsYr_OrctmNR0Yybow4FV5T-QhoNb5hjxM2a5QpcBSqdgdKpilHXUcZeE-z_A8kFBDT_zWG0)

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
