# Funcionalidad de Fuzzing BLE

## Descripción General

Este dispositivo incluye una función para realizar **Fuzzing sobre Bluetooth Low Energy (BLE)**. El objetivo es enviar secuencias de datos (leídas desde archivos de diccionario almacenados en la tarjeta SD) como si fueran entradas de teclado a un dispositivo BLE conectado. Esto permite probar la robustez del dispositivo receptor y potencialmente encontrar vulnerabilidades en cómo maneja entradas inesperadas o malformadas.

## Cómo Funciona

1.  **Activación:** El modo Fuzzing se inicia seleccionando la opción `'Z'` desde el menú principal del barre-nd3r0.
2.  **Lectura de Diccionarios:** Al activarse, la función `iniciarFuzz()` busca y abre los archivos contenidos en el directorio `/Fuzz/diccionarios/` dentro de la tarjeta SD.
3.  **Transmisión de Datos:** El contenido de cada archivo encontrado en ese directorio se lee carácter por carácter.
4.  **Emulación de Teclado BLE:** Cada carácter leído se envía inmediatamente a través de la conexión Bluetooth LE, utilizando la librería `BleKeyboard`. El barre-nd3r0 actúa como un teclado BLE ("ESP32 barre-nd3r0" por defecto) para el dispositivo objetivo.
5.  **Pausas:** Para evitar la saturación del receptor BLE y mejorar la compatibilidad, se introducen pequeñas pausas controladas por `delay()`:
    * Una pausa corta (ej. 50 ms) entre cada carácter enviado.
    * Una pausa un poco más larga (ej. 100 ms) después de enviar el contenido completo de un archivo y antes de empezar con el siguiente.
6.  **Finalización:** El proceso continúa hasta que se hayan enviado todos los caracteres de todos los archivos en el directorio de diccionarios. Al finalizar, se muestra un mensaje en pantalla y se regresa al menú principal.
7.  **Interrupción:** Es posible detener el proceso de fuzzing en cualquier momento enviando el carácter `!` a través de la entrada Serial2 (teclado conectado al barre-nd3r0). Esto se gestiona en la función `manejarFuzz()`.

## Gestión de Diccionarios de Fuzzing

* **Ubicación:** Los archivos que contienen los datos para el fuzzing (diccionarios) deben estar en formato de texto plano y almacenados exclusivamente en el directorio `/Fuzz/diccionarios/` de la tarjeta SD.
* **Creación del Directorio:** Este directorio se crea automáticamente por la función `crearEstructuraDirectorio()` al arrancar el dispositivo si no existe previamente.
* **Subida de Archivos (Método Recomendado):**
    1.  Conéctate con un PC o móvil a la red WiFi creada por el barre-nd3r0 (SSID: "barre-nd3r0", Contraseña: "12345678", por defecto).
    2.  Abre un navegador web y ve a la dirección IP del punto de acceso del barre-nd3r0 (usualmente `192.168.4.1`).
    3.  Utiliza la interfaz web proporcionada para subir tus archivos de diccionario. El servidor web está configurado (mediante el endpoint `/upload` y la función `handleFileUpload`) para detectar y guardar los archivos apropiados (como `.txt` o `.dic`) directamente en la carpeta `/Fuzz/diccionarios/`.

## Cómo Usar la Función de Fuzzing

1.  **Prepara los Diccionarios:** Crea uno o más archivos de texto (`.txt`, `.dic`, etc.) que contengan las cadenas de caracteres, comandos, o datos binarios (representados como texto) que deseas enviar al dispositivo objetivo. Cada línea, o el archivo completo, puede representar un caso de prueba.
2.  **Sube los Diccionarios:** Usa la interfaz web como se describió anteriormente para cargar estos archivos en el barre-nd3r0.
3.  **Empareja el barre-nd3r0 (como Teclado BLE):** Asegúrate de que el barre-nd3r0 esté emparejado vía Bluetooth como un teclado ("ESP32 barre-nd3r0") con el dispositivo que quieres fuzzear. Asegúrate de que la conexión esté activa.
4.  **Activa el Modo Fuzzing:** En el barre-nd3r0, usando el teclado conectado a Serial2, presiona la tecla `'Z'` desde el menú principal. La pantalla mostrará un mensaje indicando que el Fuzzing BLE está activo.
5.  **Monitoriza el Objetivo:** Observa el comportamiento del dispositivo que está recibiendo los datos. Busca cuelgues, reinicios, mensajes de error, comportamientos inesperados, o cualquier indicio de una vulnerabilidad.
6.  **Detén el Proceso (Si es necesario):** Si quieres parar el fuzzing antes de que termine de procesar todos los diccionarios, envía el carácter `!` usando el teclado conectado a Serial2. El proceso se detendrá y volverás al menú principal.

## Componentes de Código Relevantes

* **Funciones Principales:** `iniciarFuzz()`, `manejarFuzz()`, `crearEstructuraDirectorio()`, `handleFileUpload()`.
* **Activación:** Opción `'Z'` en `manejarComandoGeneral()`.
* **Delegación de Input:** Llamada a `manejarFuzz()` desde `manejarSerial2()` cuando `modoFuzz` es `true`.
* **Librería Clave:** `BleKeyboard.h`.
* **Estado:** Variable global `modoFuzz`.
* **Configuración:** Endpoint `/upload` en `setup()`.
