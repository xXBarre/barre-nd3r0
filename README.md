## Descripción General

Este firmware convierte un ESP32 en una herramienta multifuncional portátil, apodada "barre-nd3r0". Utiliza una pantalla de tinta electrónica (e-paper) para la interfaz de usuario, acepta entrada de teclado a través de un puerto serie secundario (Serial2), y ofrece diversas funcionalidades orientadas a la experimentación y utilidad en movilidad:

* **Editor de Texto:** Permite crear y editar archivos de texto simple y código directamente en el dispositivo, con almacenamiento en tarjeta SD.
* **Shell de Comandos:** Ofrece una interfaz básica tipo línea de comandos para interactuar con el sistema de archivos de la SD.
* **Explorador de Archivos:** Permite navegar por los directorios de la tarjeta SD y abrir archivos.
* **Comunicación LoRa:** Capacidad para enviar y recibir mensajes a través de LoRa, con un modo de monitor/chat.
* **Teclado Bluetooth (BLE):** Emula un teclado Bluetooth, permitiendo escribir en dispositivos cercanos. Puede ejecutar scripts almacenados en la SD (enviar secuencias de teclas) y realizar fuzzing BLE.
* **Servidor Web WiFi:** Crea un punto de acceso WiFi y aloja una interfaz web para editar archivos (usando CodeMirror) y subir nuevos archivos (incluyendo diccionarios de fuzzing) a la tarjeta SD.
* **Gestión de Energía:** Incluye modo Deep Sleep para ahorrar batería y monitorización del nivel de la misma.
* **Persistencia:** Guarda el estado actual (modo, texto en edición, etc.) para restaurarlo tras un reinicio o despertar del Deep Sleep.
