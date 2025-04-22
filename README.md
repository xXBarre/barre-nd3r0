# barre-nd3r0

Es una navaja suiza digital de bajo consumo y alto potencial para pentesting y comunicación offline, basada en un ESP32 con pantalla e-ink, LoRa, teclado y tarjeta SD.

# Documentación del Firmware ESP32 HackerPad

## Descripción General

Este firmware convierte un ESP32 en una herramienta multifuncional portátil, apodada "HackerPad". Utiliza una pantalla de tinta electrónica (e-paper) para la interfaz de usuario, acepta entrada de teclado a través de un puerto serie secundario (Serial2), y ofrece diversas funcionalidades orientadas a la experimentación y utilidad en movilidad:

* **Editor de Texto:** Permite crear y editar archivos de texto simple y código directamente en el dispositivo, con almacenamiento en tarjeta SD.
* **Shell de Comandos:** Ofrece una interfaz básica tipo línea de comandos para interactuar con el sistema de archivos de la SD.
* **Explorador de Archivos:** Permite navegar por los directorios de la tarjeta SD y abrir archivos.
* **Comunicación LoRa:** Capacidad para enviar y recibir mensajes a través de LoRa, con un modo de monitor/chat.
* **Teclado Bluetooth (BLE):** Emula un teclado Bluetooth, permitiendo escribir en dispositivos cercanos. Puede ejecutar scripts almacenados en la SD (enviar secuencias de teclas) y realizar fuzzing BLE.
* **Servidor Web WiFi:** Crea un punto de acceso WiFi y aloja una interfaz web para editar archivos (usando CodeMirror) y subir nuevos archivos (incluyendo diccionarios de fuzzing) a la tarjeta SD.
* **Gestión de Energía:** Incluye modo Deep Sleep para ahorrar batería y monitorización del nivel de la misma.
* **Persistencia:** Guarda el estado actual (modo, texto en edición, etc.) para restaurarlo tras un reinicio o despertar del Deep Sleep.

## Bibliotecas Utilizadas

* `GxEPD2_BW.h`: Para controlar la pantalla E-Paper monocromática.
* `LoRa.h`: Para la comunicación por radio LoRa.
* `Preferences.h`: Para almacenar datos de configuración y estado de forma persistente en la memoria flash del ESP32.
* `SD.h`: Para interactuar con la tarjeta SD (sistema de archivos).
* `SPI.h`: Necesaria para la comunicación con la tarjeta SD y la pantalla E-Paper.
* `BleKeyboard.h`: Para emular un teclado Bluetooth Low Energy.
* `WiFi.h`: Para funcionalidades WiFi (en este caso, crear un Access Point).
* `ESPAsyncWebServer.h`: Para crear un servidor web asíncrono eficiente.
* `Wire.h`: Aunque incluida, no parece usarse explícitamente en el código proporcionado (podría ser una dependencia de otra librería o para futura expansión I2C).
* `esp_sleep.h`: Para gestionar los modos de bajo consumo (Deep Sleep) del ESP32.

## Pines Definidos

* `RXD2 (16)`: Pin RX para la comunicación serie secundaria (entrada de teclado).
* `TXD2 (17)`: Pin TX para la comunicación serie secundaria.
* `SD_CS (15)`: Pin Chip Select (CS) para la tarjeta SD.
* `PIN_BATTERY (34)`: Pin analógico para leer el voltaje de la batería.
* *Nota:* Los pines para la pantalla E-Paper (CLK=5, DIN=17, CS=16, DC=4, RST=?, BUSY=?) y LoRa (CS=18, RST=14, IRQ/DIO0=26) se configuran en la inicialización de sus respectivos objetos/librerías. Hay una posible superposición/error en la definición de pines (TXD2 y EPD_CS en 17, RXD2 y EPD_DC en 16) que debería revisarse en el hardware real.

## Variables Globales Principales

* `display`: Objeto para controlar la pantalla E-Paper.
* `preferences`: Objeto para guardar/leer datos persistentes.
* `bleKeyboard`: Objeto para la funcionalidad de teclado BLE.
* `server`: Objeto para el servidor web asíncrono.
* `buffer`: String para almacenar temporalmente la entrada de usuario (p.ej., en el Shell).
* `pathActual`: String que guarda la ruta del directorio actual en el explorador de archivos.
* `modoEditor`, `modoShell`, `modoExplorador`, `modoLoRa`, `modoCode`, `modoFuzz`: Booleanos que indican el modo de operación actual del dispositivo.
* `deepSleepEnabled`: Booleano para controlar el estado de Deep Sleep.
* `ultimoInput`: Timestamp del último input recibido, para el temporizador de inactividad.
* `tiempoInactividad`: Duración (en ms) de inactividad antes de entrar en Deep Sleep.
* `archivoEditando`: Objeto `File` para el archivo abierto en el editor.
* `cursorPos`, `cursorLinea`: Posición del cursor dentro del texto en el editor.
* `paginaActual`, `lineasPorPagina`: Variables para la paginación del texto en la pantalla.
* `textoEditor`: String que contiene el texto completo del archivo que se está editando.
* `mensajeLoRa`, `mensajeRecibido`, `historialLoRa`: Variables para gestionar los mensajes LoRa recibidos y el historial del chat.

## Descripción de Funciones

---

### `void drawText(String text, int cursorLine = -1)`

* **Propósito:** Dibuja el `text` proporcionado en la pantalla E-Paper.
* **Funcionamiento:** Limpia la pantalla, calcula qué líneas mostrar según `paginaActual` y `lineasPorPagina`, y las imprime. Si `cursorLine` es un valor válido (>= 0), añade un `>` al inicio de la línea correspondiente para indicar la posición del cursor. Gestiona la actualización por páginas de la pantalla E-Paper (`firstPage`/`nextPage`).

---

### `void avanzarPagina()`

* **Propósito:** Incrementa el número de `paginaActual` y redibuja el `textoEditor` para mostrar la página siguiente.
* **Funcionamiento:** Aumenta `paginaActual` en 1 y llama a `drawText`.

---

### `void retrocederPagina()`

* **Propósito:** Decrementa el número de `paginaActual` (si no es la primera) y redibuja el `textoEditor` para mostrar la página anterior.
* **Funcionamiento:** Disminuye `paginaActual` en 1 (solo si es mayor que 0) y llama a `drawText`.

---

### `void moverCursorArriba()`

* **Propósito:** Mueve el indicador de cursor (`>`) una línea hacia arriba en el `textoEditor` mostrado.
* **Funcionamiento:** Disminuye `cursorLinea` en 1 (solo si es mayor que 0) y llama a `drawText` para actualizar la pantalla.

---

### `void moverCursorAbajo()`

* **Propósito:** Mueve el indicador de cursor (`>`) una línea hacia abajo en el `textoEditor` mostrado.
* **Funcionamiento:** Aumenta `cursorLinea` en 1 y llama a `drawText` para actualizar la pantalla. (Nota: Podría necesitar un límite superior basado en el número de líneas).

---

### `void setup()`

* **Propósito:** Función de inicialización que se ejecuta una vez al arrancar el ESP32.
* **Funcionamiento:**
    * Inicializa las comunicaciones serie (`Serial` y `Serial2`).
    * Inicializa la pantalla E-Paper (`display.init()`) y configura sus parámetros (rotación, tamaño de texto, color).
    * Configura los pines para LoRa e intenta inicializar el módulo LoRa (`LoRa.begin()`). Muestra un mensaje de error en pantalla si falla.
    * Intenta inicializar la tarjeta SD (`SD.begin()`). Muestra un mensaje de error si falla.
    * Inicializa el objeto `Preferences` para acceder al almacenamiento persistente.
    * Inicializa el teclado Bluetooth (`bleKeyboard.begin()`).
    * Configura y activa el punto de acceso WiFi (`WiFi.softAP()`).
    * Configura el servidor web asíncrono (`server`):
        * Define la ruta raíz (`/`) para servir el archivo `index.html` desde la SD (interfaz web con CodeMirror).
        * Define la ruta `/guardar` (POST) para recibir datos de la interfaz web y guardar archivos en `/Notas/`.
        * Define la ruta `/upload` (POST) para gestionar la subida de archivos desde la web, guardándolos en `/Fuzz/diccionarios/`.
    * Inicia el servidor web (`server.begin()`).
    * Llama a `crearEstructuraDirectorio()` para asegurarse de que existen las carpetas necesarias en la SD.
    * Llama a `restaurarEstado()` para cargar la configuración y estado previos.
    * Llama a `mostrarMenuPrincipal()` para dibujar el menú inicial en la pantalla.

---

### `void loop()`

* **Propósito:** Función principal que se ejecuta repetidamente después de `setup()`.
* **Funcionamiento:**
    * Comprueba si ha transcurrido el `tiempoInactividad` desde el `ultimoInput`. Si es así, llama a `activarDeepSleep()`.
    * Llama a `manejarLoRa()` para procesar posibles mensajes LoRa entrantes.
    * Llama a `manejarSerial2()` para procesar la entrada de teclado desde el puerto serie secundario.
    * Llama a `autoGuardarEstado()` para guardar periódicamente el estado actual.
    * Comprueba si hay datos disponibles en el puerto serie principal (`Serial`) y procesa comandos básicos de paginación/cursor (`+`, `-`, `w`, `s`). (Parece ser una entrada de control alternativa o para depuración).

---

### `void iniciarFuzz()`

* **Propósito:** Inicia el modo de Fuzzing BLE.
* **Funcionamiento:**
    * Establece `modoFuzz` a `true`.
    * Muestra un mensaje en pantalla indicando que el modo Fuzzing está activo.
    * Abre el directorio `/Fuzz/diccionarios` en la SD.
    * Itera sobre cada archivo dentro de ese directorio.
    * Para cada archivo, lee su contenido carácter por carácter y lo envía a través del `bleKeyboard` emulado, con una pequeña pausa (`delay(50)`) entre caracteres.
    * Añade una pausa (`delay(100)`) después de procesar cada archivo.
    * Una vez procesados todos los archivos, establece `modoFuzz` a `false`, muestra un mensaje de "Fuzzing terminado" y vuelve al menú principal (`mostrarMenuPrincipal()`).

---

### `void manejarFuzz(char c)`

* **Propósito:** Gestiona la entrada de teclado mientras se está en modo Fuzzing.
* **Funcionamiento:** Si se recibe el carácter `!`, desactiva el modo Fuzzing (`modoFuzz = false`) y vuelve al menú principal (`mostrarMenuPrincipal()`). Permite interrumpir el proceso de fuzzing.

---

### `void manejarComandoGeneral(char c)`

* **Propósito:** Procesa los comandos recibidos cuando no se está en ningún modo específico (Editor, Shell, Explorador, Fuzzing). Actúa como el dispatcher del menú principal.
* **Funcionamiento:** Utiliza un `switch` para ejecutar diferentes acciones según el carácter `c` recibido:
    * `'E'`: Llama a `iniciarEditor()` (Función no mostrada en el código, pero se infiere su existencia).
    * `'X'`: Activa el `modoShell`, limpia el `buffer` y muestra un prompt en pantalla.
    * `'F'`: Llama a `iniciarExplorador()` (Función no mostrada, se infiere).
    * `'S'`: Llama a `activarDeepSleep()`.
    * `'B'`: Llama a `mostrarEstadoBateria()`.
    * `'L'`: Cambia el estado de `modoLoRa` (lo activa o desactiva) y muestra el estado actual en pantalla.
    * `'Z'`: Llama a `iniciarFuzz()`.
    * `default`: Muestra el comando recibido en pantalla (acción por defecto si no coincide con ninguno).

---

### `void manejarSerial2()`

* **Propósito:** Lee y procesa los caracteres recibidos por el puerto serie secundario (`Serial2`), que se asume es la entrada principal del usuario (teclado).
* **Funcionamiento:**
    * Mientras haya caracteres disponibles en `Serial2`:
        * Lee un carácter `c`.
        * Actualiza `ultimoInput` con `millis()` para reiniciar el temporizador de inactividad.
        * Comprueba el modo activo (`modoEditor`, `modoShell`, `modoExplorador`, `modoFuzz`) y llama a la función de manejo correspondiente (`manejarEditor`, `manejarShell`, `manejarExplorador`, `manejarFuzz`). (Nota: `manejarEditor` y `manejarShell` no están completamente definidos en el extracto).
        * Si ningún modo específico está activo, llama a `manejarComandoGeneral(c)` para procesar comandos del menú principal.

---

### `void manejarLoRa()`

* **Propósito:** Comprueba si se han recibido paquetes LoRa y los procesa.
* **Funcionamiento:**
    * Llama a `LoRa.parsePacket()`. Si devuelve un valor mayor que 0, significa que se recibió un paquete.
    * Lee el contenido del paquete carácter por carácter hasta que no haya más disponibles, guardándolo en `mensajeLoRa`.
    * Añade el mensaje recibido al `historialLoRa`.
    * Establece `mensajeRecibido` a `true`.
    * Si `modoLoRa` está activo (modo Chat), muestra el historial completo en pantalla.
    * Si `modoLoRa` no está activo, solo muestra el último mensaje recibido.
    * Comprueba si el mensaje empieza con `"!save"`. Si es así, extrae el contenido del mensaje (después del salto de línea) y lo guarda en un nuevo archivo de texto dentro del directorio `/Recibidos/` en la SD, usando un nombre basado en el timestamp (`millis()`).

---

### `void crearEstructuraDirectorio()`

* **Propósito:** Asegura que la estructura de directorios necesaria exista en la tarjeta SD al inicio.
* **Funcionamiento:** Comprueba la existencia de los directorios `/Notas`, `/Recibidos`, `/Codigo`, `/Enviados`, `/Scripts`, y `/Fuzz/diccionarios`. Si alguno no existe, lo crea usando `SD.mkdir()`. Para directorios anidados (como `/Fuzz/diccionarios`), crea los directorios padre si es necesario.

---

### `void mostrarMenuPrincipal()`

* **Propósito:** Muestra el menú principal de opciones en la pantalla E-Paper.
* **Funcionamiento:** Construye un `String` con las opciones del menú (Editor, Shell, Explorador, Deep Sleep, Batería, LoRa Chat, Fuzzing BLE) y sus letras de comando asociadas. Luego, llama a `drawText()` para mostrar este menú en pantalla.

---

### `void restaurarEstado()`

* **Propósito:** Carga el estado operativo anterior desde el almacenamiento persistente (`Preferences`).
* **Funcionamiento:** Lee los valores guardados para las variables de modo (`modoEditor`, `modoShell`, etc.), el texto del editor (`textoEditor`), la posición del cursor (`cursorPos`, `cursorLinea`), y otros flags de estado usando `preferences.getBool()`, `preferences.getString()`, y `preferences.getInt()`. Utiliza valores por defecto si una clave no existe.

---

### `void autoGuardarEstado()`

* **Propósito:** Guarda periódicamente el estado operativo actual en el almacenamiento persistente.
* **Funcionamiento:** Comprueba si ha pasado un intervalo de tiempo (10 segundos) desde el último guardado. Si es así, escribe los valores actuales de las variables de modo, texto del editor, posición del cursor, etc., en `Preferences` usando `preferences.putBool()`, `preferences.putString()`, y `preferences.putInt()`. Actualiza el timestamp del último guardado.

---

### `bool esArchivoCodigo(String nombre)`

* **Propósito:** Determina si un nombre de archivo corresponde a un tipo de archivo de código común.
* **Funcionamiento:** Comprueba si el `nombre` del archivo termina con extensiones como `.ino`, `.py`, `.sh`, `.cpp`, `.c`, o `.js`. Devuelve `true` si coincide con alguna, `false` en caso contrario.

---

### `void abrirArchivo(String ruta)`

* **Propósito:** Abre un archivo de la tarjeta SD, carga su contenido en `textoEditor` y entra en modo edición.
* **Funcionamiento:**
    * Intenta abrir el archivo en la `ruta` especificada usando `SD.open()`.
    * Si tiene éxito:
        * Limpia `textoEditor`.
        * Lee todo el contenido del archivo y lo almacena en `textoEditor`.
        * Cierra el archivo.
        * Establece `cursorPos` al final del texto.
        * Activa `modoEditor`.
        * Llama a `esArchivoCodigo()` para determinar si es un archivo de código y establece `modoCode` apropiadamente.
        * Resetea `paginaActual` y `cursorLinea` a 0.
        * Llama a `drawText()` para mostrar el contenido del archivo y el título "Editando:".
    * Si falla al abrir, no hace nada (implícitamente).

---

### `void guardarArchivoEditado()`

* **Propósito:** Guarda el contenido de `textoEditor` en un nuevo archivo en la tarjeta SD.
* **Funcionamiento:**
    * Determina el nombre del archivo y el directorio de destino:
        * Si `modoCode` es `true`, guarda en `/Codigo/` con un nombre como `codigo_N.ext`, donde `N` es un contador incremental (`code_count` guardado en `Preferences`) y `ext` es `.sh` si el texto empieza con `#!/` o `.ino` en caso contrario.
        * Si `modoCode` es `false`, guarda en `/Notas/` con un nombre como `nota_M.txt`, donde `M` es otro contador incremental (`nota_count`).
    * Incrementa el contador correspondiente en `Preferences`.
    * Intenta abrir el archivo con el nombre generado en modo escritura (`FILE_WRITE`).
    * Si tiene éxito:
        * Escribe el contenido de `textoEditor` en el archivo.
        * Cierra el archivo.
        * Muestra un mensaje de "Archivo guardado en: [ruta]" en pantalla.
        * Si el archivo guardado es un script (`.sh`), pregunta al usuario (a través de la pantalla y esperando input en `Serial2`) si desea ejecutarlo. Si la respuesta es 'Y' o 'y', llama a `ejecutarScript()`.
    * Si falla al abrir/escribir, muestra un mensaje de "Error al guardar".

---

### `void ejecutarScript(String ruta)`

* **Propósito:** Ejecuta un script almacenado en la SD enviando su contenido como pulsaciones de teclado vía BLE.
* **Funcionamiento:**
    * Intenta abrir el archivo en la `ruta` especificada. Si no lo encuentra, muestra un error y retorna.
    * Muestra "Ejecutando script..." en pantalla.
    * Lee el archivo carácter por carácter.
    * Para cada carácter, lo envía usando `bleKeyboard.print(c)`.
    * Introduce un pequeño retardo (`delay(50)`) entre caracteres para evitar saturar el receptor BLE.
    * Cierra el archivo.
    * Muestra "Script ejecutado" en pantalla.

---

### `void listarDirectorio(String path)`

* **Propósito:** Muestra el contenido (archivos y subdirectorios) del directorio especificado en `path` en la pantalla.
* **Funcionamiento:**
    * Abre el directorio `path` en la SD.
    * Construye un `String` (`listado`) comenzando con "Contenidos de [path]:\n".
    * Itera sobre todas las entradas (archivos/directorios) dentro del directorio usando `dir.openNextFile()`.
    * Para cada entrada, omite la carpeta "System Volume Information".
    * Añade `[DIR]` o `[FILE]` seguido del nombre de la entrada al `listado`.
    * Incrementa un contador.
    * Cierra cada entrada.
    * Si el contador es 0 después de iterar, añade "Vacío\n" al `listado`.
    * Llama a `drawText()` para mostrar el `listado` en pantalla.

---

### `void mostrarArchivo(String nombre)`

* **Propósito:** Muestra el contenido de un archivo específico en la pantalla.
* **Funcionamiento:**
    * Comprueba si el nombre empieza con `/Fuzz/diccionarios/`. Si es así, lo trata como un archivo de texto normal y llama a `abrirArchivo()` para verlo/editarlo en el editor paginado.
    * Intenta abrir el archivo `nombre` en la SD. Si no existe, muestra "No encontrado: [nombre]" y retorna.
    * Comprueba el tamaño del archivo. Si es grande (> 1000 bytes), llama a `abrirArchivo()` para usar el editor paginado.
    * Si es pequeño, lee todo el contenido en un `String` y lo muestra precedido por "Contenido:\n" usando `drawText()`.
    * Cierra el archivo.

---

### `void manejarExplorador(char c)`

* **Propósito:** Gestiona la entrada del usuario mientras se está en el modo Explorador de Archivos.
* **Funcionamiento:** Utiliza una variable estática `inputPath` para acumular la entrada del usuario.
    * Si `c` es `!`: Sale del modo Explorador, limpia `inputPath`, y vuelve al menú principal.
    * Si `c` es `\n` (Enter):
        * Si `inputPath` no está vacío:
            * Si `inputPath` es `".."`: Navega al directorio padre actualizando `pathActual`.
            * Si `inputPath` empieza con `/` (ruta absoluta): Intenta cambiar `pathActual` a esa ruta si existe.
            * Si no, considera `inputPath` como relativo: Construye la ruta completa (`pathActual + inputPath`). Si existe:
                * Si es un directorio (termina en `/` o `SD.exists` lo confirma como tal), actualiza `pathActual`.
                * Si es un archivo, llama a `abrirArchivo()` para verlo/editarlo y sale de la función.
            * Limpia `inputPath`.
            * Llama a `listarDirectorio(pathActual)` para mostrar el contenido del nuevo directorio.
    * Si `c` es `\b` (Backspace): Borra el último carácter de `inputPath` (si no está vacío) y actualiza la pantalla.
    * Si `c` es un carácter imprimible (ASCII 32-126): Lo añade a `inputPath` y actualiza la pantalla mostrando el path actual y el input del usuario.

---

### `void ejecutarComando(String cmd)`

* **Propósito:** Ejecuta comandos simples introducidos en el modo Shell.
* **Funcionamiento:** Procesa el `String cmd`:
    * `"list"`: Llama a `listarDirectorio(pathActual)`.
    * `"cat [archivo]"`: Extrae el nombre del archivo y llama a `mostrarArchivo()`.
    * `"cd [directorio]"`: Extrae el nombre del directorio. Si existe, actualiza `pathActual` y llama a `listarDirectorio()`. Si no, muestra un error.
    * `"run [script]"`: Extrae el nombre del script y llama a `ejecutarScript()`.
    * `"clear"`: Llama a `drawText("")` para limpiar la pantalla.
    * `default`: Muestra "Comando no válido: [cmd]".

---

### `void activarDeepSleep()`

* **Propósito:** Pone el ESP32 en modo Deep Sleep para ahorrar energía.
* **Funcionamiento:**
    * Guarda `deepSleep = true` en `Preferences` (posiblemente para saber al despertar que viene de un deep sleep, aunque no se usa explícitamente después).
    * Muestra "Entrando en Deep Sleep..." en pantalla.
    * Espera 2 segundos (`delay(2000)`) para que el mensaje sea visible.
    * Llama a `esp_deep_sleep_start()` para poner el microcontrolador a dormir. El ESP32 solo despertará por una causa externa configurada (no especificada en este código, usualmente un pin RTC o temporizador).

---

### `void mostrarEstadoBateria()`

* **Propósito:** Lee el voltaje de la batería y lo muestra en pantalla junto con un porcentaje estimado.
* **Funcionamiento:**
    * Lee el valor analógico del `PIN_BATTERY`.
    * Convierte la lectura a voltaje, considerando un posible divisor de voltaje (factor 2 en el código) y el voltaje de referencia del ADC (3.3V).
    * Calcula un porcentaje aproximado asumiendo un rango de LiPo (3.3V vacío, 4.2V lleno), asegurándose de que esté entre 0 y 100 (`constrain`).
    * Construye un `String` con el voltaje (2 decimales) y el porcentaje (0 decimales).
    * Añade mensajes adicionales: "¡Batería baja!" si el voltaje es < 3.5V, o "Cargado" si es > 4.1V.
    * Llama a `drawText()` para mostrar el estado de la batería.

---

### `void handleFileUpload(AsyncWebServerRequest *request, String filename, ...)`

* **Propósito:** Función callback para el servidor web que maneja la subida de archivos fragmentada. (Nota: Existe una ruta `/upload` específica para diccionarios de fuzzing, esta función parece ser un handler más genérico o de una versión anterior, aunque está referenciada como "compatibilidad v2").
* **Funcionamiento:**
    * Se llama múltiples veces durante la subida de un archivo.
    * En la primera llamada (`index == 0`):
        * Determina la ruta de destino en la SD basándose en la extensión del archivo (`filename`): `.txt`/`.md` van a `/Notas/`, `.ino`/`.cpp`/`.py` a `/Codigo/`, `.dic` a `/Fuzz/diccionarios/`, y otros a `/Scripts/`.
        * Abre el archivo en la ruta determinada en modo escritura (`FILE_WRITE`). Guarda el objeto `File` en la variable estática `fsUploadFile`. Maneja errores si no se puede crear el archivo.
    * En llamadas subsiguientes (`index > 0`): Escribe el fragmento de datos (`data`, de longitud `len`) en el archivo abierto (`fsUploadFile`).
    * En la última llamada (`final == true`): Cierra el archivo (`fsUploadFile.close()`) y envía una respuesta HTTP 200 al cliente indicando que el archivo se subió correctamente.
