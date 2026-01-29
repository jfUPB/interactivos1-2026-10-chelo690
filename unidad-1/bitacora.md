# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 1: 
- Los sistemas fisicos interactivos se basan principalmente en la interaccion fisico-digital de un usuario utilizando diferentes tipos de sistemas que realizan diversas actividades, a esos sistemas se les denominan sistemas fisicos interactivos.

- Se podrian utilizar dentro del ambito profesional como una forma mas inmersiva de experimentar el mundo digital, y a su vez una util herramienta visual y sensorial para trabajos y proyectos.

### Actividad 2:
- Dentro de lo que comprendi el Diseño/Arte generativo es una forma de dar diversos puntos de vusta e infirmacion al usuario acerca de productos y marcas utilizando varias herramientas digitales.

- Podria utilizarse dentro del perfil profesional como una forma de diseño de marca, paginas web, animaciones y demas para dar una experiencia diferente y original al usuario.

### Actividad 3:

-Al sacudir el microbitrt el circulo se muestra verde con una C en el centro
-AL presionar send love aparece una carita feliz en el microbit

### Actividad 4:

- Al utilizar was_Pressed() detecta la pulsacion y devuelve en true solo por un corto instante que no refleja la accion en pantalla mientras que con el metodo Is_Pressed() devuelve true todo el tiempo y es una respuesta fluida y constante al microbit que no deja de funcionar solo con una pulsacion sino que sigue enviando informacion mientras se mantenga el boton presionado.


## Bitácora de aplicación 

### Actividad 5:

#### Python editor: from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
        
#### P5.js: 
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);

  
    fill('white');
    Movex=width/2;
    ellipse(Movex,height/2, 100,100)
}

function draw() {

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            fill('red');
          
            Movex = Movex -15;

        }
        else if(dataRx == 'B'){
            fill('yellow');
          
          Movex = Movex +15;
          
        }
        else{
            fill('green');
        }
        background(220);
        ellipse(Movex, height / 2, 100, 100);
        fill('black');
        text(dataRx,Movex, height / 2);
    }


    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}

function sendBtnClick() {
    port.write('h');
}

## Bitácora de reflexión




