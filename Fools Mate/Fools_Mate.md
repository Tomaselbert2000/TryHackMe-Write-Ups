# TryHackMe Rooms

## "Fools Mate" - Spanish Walkthrough

### Configuración

Como es habitual, la configuración inicial de la sala fue agregar una nueva entrada dentro del archivo **/etc/hosts** con un dominio sencillo y la dirección IP de la máquina objetivo.

```sudo echo "<IP del objetivo> foolsmate.thm" >> /etc/hosts```

### Fase de reconocimiento

Se dió inicio a la fase de reconocimiento del objetivo llevando a cabo un escaneo con Nmap, aplicando:

- Scripts por defecto
- Descubrimiento de versiones de servicios en ejecución
- Descubrimiento de versiones de sistema operativo anfitrión.

Se adjunta a continuación el comando ejecutado:

```nmap -sV -sC -O foolsmate.thm | tee nmap_scan.txt```

- _La salida fue redirigida a un archivo de texto para facilitar análisis posterior_

En la imagen siguiente se muestran los resultados del escaneo inicial.

![alt text](<Imagenes/1 Escaneo con Nmap.png>)

El reporte anterior evidenció como puertos abiertos aquellos correspondientes a **SSH** (22) y **HTTP** (80). Luego, y ya realizando reconocimiento manual con el navegador, se encontró que el servidor expone una página que invita al visitante a jugar una partida de ajedrez.

![alt text](<Imagenes/2 Página en el puerto 80.png>)

Paralelamente a este hallazgo, se llevó a cabo una enumeración de directorios mediante FFUF, apuntando al directorio principal donde se hostea la página mostrada. Los resultados iniciales se detallan en la imagen adjunta.

![alt text](<Imagenes/3 Enumeración con FFUF.png>)

- _El fuzzeo de directorios no arrojó resultados relevantes para la investigación_

### Análisis de funcionamiento del juego

Luego de un fuzzeo sin resultados relevantes, el siguiente paso fue analizar el funcionamiento de la página que sustenta el panel de juego. Al iniciar la página, se encuentran las siguientes piezas para ambos jugadores:

- Rey Negro, 3 peones.
- Rey Blanco, 3 peones y **1 Torre**

El panel invita al jugador a ganar la partida con un "Mate en uno", es decir, ganar con un movimiento tal que inmovilice totalmente al rey oponente. Dadas las posiciones iniciales de las piezas, se determinó que para conseguir ganar, era necesario mover la torre hacia la posición A8. Pero el detalle radica en que el juego bloquea activamente este movimiento, con lo cual no permite al jugador llevar la torre a dicha posición y alertando de tal situación con un mensaje en pantalla.

![alt text](<Imagenes/4 Movimiento bloqueado.png>)

Debido a esto, se investigó en más detalle el código fuente de la página, para luego encontrar que utiliza un módulo escrito en JavaScript de nombre **app.js**.

![alt text](<Imagenes/5 Código fuente de la página principal.png>)

Este módulo se encarga de encapsular toda la lógica referida a las reglas del juego, movimientos de las piezas y validación de jugadas. El origen del mensaje de error y por consiguiente el bloqueo que impide al jugar realizar el "Mate en uno" radica en la función **preMoveCheck()** mostrada a continuación:

```Javascript
function preMoveCheck(from, to, promotion) {
  const probe = new Chess(game.fen());
  let result;
  try {
    result = probe.move({ from, to, promotion: promotion || undefined });
  } catch (e) {
    result = null;
  }
  if (result && probe.isCheckmate()) {
    showSystemNotice("I'll shut down your PC if you play that.");
    return false;
  }
  return true;
}
```

Básicamente, el juego genera una situación en la que se bloquea por software que el jugador pueda realizar el "Mate en uno", ya que cuando se selecciona la torre y se la desplaza a A8, la request enviada por el navegador dispara la ejecución de esta validación. Al ser un movimiento que sí genera el "Mate en uno", el bloque if retorna True y dispara el mensaje en pantalla que ve el jugador, haciendo imposible ganar el juego con un solo movimiento y solo mediante interacción directa con el tablero.

### Bypass con OWASP Zap

Para conseguir saltar esta limitación, se utilizó OWASP Zap para enviar una solicitud al objetivo especificando tanto la posición inicial de la pieza (en este caso **A1**), así como la posición final (la cual será **A8**). Estos datos fueron enviados en el cuerpo de la solicitud. Dado que la solicitud se envía de forma directa, se evade la validación llevada a cabo por **app.js**, y en el cuerpo de la respuesta se visualiza el texto correspondiente a la bandera de la sala.

![alt text](<Imagenes/6 Bandera de la sala obtenida.png>)

- _Para la edición de la request, recordar utilizar letras en **minúsculas**_
