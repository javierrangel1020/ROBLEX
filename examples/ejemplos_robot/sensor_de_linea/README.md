# Seguidor de línea ROBLEX (robot de 4 motores)

Programa para que el robot ROBLEX siga una línea de forma autónoma con el módulo sensor de línea (8 sensores QRE1113 leídos por un ADC ADS7830 por I2C). Corrige la dirección con un control PID cada 5 ms y se controla con el botón de la shield o con la app ROBLEX por Bluetooth (BLE): velocidad en vivo, PID, umbral, color de la línea y calibración de los sensores.

## Contenido

1. [Qué se necesita](#qué-se-necesita)
2. [Instalar el entorno de Arduino](#instalar-el-entorno-de-arduino)
3. [Instalar la librería ROBLEX](#instalar-la-librería-roblex)
4. [Conexiones](#conexiones)
5. [Configurar y cargar el programa](#configurar-y-cargar-el-programa)
6. [Primera prueba](#primera-prueba)
7. [Calibrar los sensores](#calibrar-los-sensores)
8. [Calibrar el PID y la velocidad](#calibrar-el-pid-y-la-velocidad)
9. [Uso sin la app](#uso-sin-la-app)
10. [Protocolo BLE](#protocolo-ble)
11. [Problemas frecuentes](#problemas-frecuentes)

## Qué se necesita

- Robot ROBLEX con shield ESP32 y cuatro motores con sus módulos driver.
- Módulo sensor de línea ROBLEX (ADS7830, dirección I2C `0x48` de fábrica).
- Pantalla OLED SSD1306 de 128x64 por I2C (dirección `0x3C`). Es opcional para seguir la línea, pero el programa la usa para mostrar los sensores.
- Cable USB de datos y un computador con Windows, macOS o Linux.
- Para controlarlo desde el teléfono: la app ROBLEX (repositorio RobotCompetitionController, rama `feat/pid-velocidad-calibracion`).

## Instalar el entorno de Arduino

1. Instala [Arduino IDE 2](https://www.arduino.cc/en/software).
2. En **Archivo > Preferencias > URLs adicionales de gestor de placas** agrega:
   `https://espressif.github.io/arduino-esp32/package_esp32_index.json`
3. En **Herramientas > Placa > Gestor de placas** busca **esp32** e instala **esp32 by Espressif Systems**, versión 3.x. El programa está compilado con la 3.3.12.
4. En **Herramientas > Gestionar bibliotecas** instala, aceptando sus dependencias:
   - Adafruit SSD1306
   - Adafruit GFX Library
   - Adafruit VL53L0X
   - Adafruit PCF8574

`Preferences.h`, que guarda los ajustes en la memoria del robot, viene con el paquete del ESP32: no hay que instalarla.

## Instalar la librería ROBLEX

El programa usa funciones del sensor de línea que solo están en esta versión de la librería. **Si ya tienes otra librería ROBLEX instalada, hay que reemplazarla**: el IDE usa la que esté en la carpeta de librerías, normalmente `Documentos/Arduino/libraries/ROBLEX`.

Opción A, con git (recomendada, permite actualizar con `git pull`):

1. Cierra el Arduino IDE.
2. Borra o mueve fuera la carpeta `Documentos/Arduino/libraries/ROBLEX` si existe.
3. Clona la rama dentro de la carpeta de librerías:

   ```bash
   git clone -b feat/pid-velocidad-calibracion <URL del repositorio ROBLEX> ~/Documents/Arduino/libraries/ROBLEX
   ```

Opción B, sin git:

1. En GitHub, en el repositorio ROBLEX, elige la rama `feat/pid-velocidad-calibracion` y pulsa **Code > Download ZIP**.
2. Borra o mueve fuera la carpeta `Documentos/Arduino/libraries/ROBLEX` si existe.
3. En el Arduino IDE: **Sketch > Incluir biblioteca > Añadir biblioteca .ZIP** y elige el ZIP descargado.

Para comprobar que quedó la versión correcta: **Archivo > Ejemplos > ROBLEX > ejemplos_robot** debe tener el ejemplo **sensor_de_linea**.

## Conexiones

| Elemento | Dónde se conecta |
|---|---|
| Módulo sensor de línea | Bus I2C de la shield: 1 GND, 2 3.3V, 3 SCL, 4 SDA |
| Pantalla OLED | Bus I2C de la shield |
| Motor delantero izquierdo | Puerto 4 |
| Motor delantero derecho | Puerto 5 |
| Motor trasero izquierdo | Puerto 1 |
| Motor trasero derecho | Puerto 3 |

Los dos motores de cada lado reciben la misma señal PWM. El sensor 0 (S0) debe quedar a la **izquierda** del robot mirando hacia adelante; si queda a la derecha, cambia `INVERTIR_SENSORES` a `true`.

## Configurar y cargar el programa

1. Abre **Archivo > Ejemplos > ROBLEX > ejemplos_robot > sensor_de_linea**. Guárdalo como copia si vas a cambiarlo.
2. Revisa los valores al principio del programa:

   | Valor | Qué hace | Por defecto |
   |---|---|---|
   | `RobotName` | Nombre del robot en Bluetooth | `"ROBLEX linea"` |
   | `INVERTIR_SENSORES` | `true` si S0 queda a la derecha | `false` |
   | `TIEMPO_SIN_LINEA` | ms sin ver la línea antes de detenerse (0 = nunca) | `1500` |
   | `VELOCIDAD_AUTOCAL` | Velocidad de giro de la calibración automática (0-255) | `90` |
   | `TIEMPO_GIRO_AUTOCAL` | Duración de cada giro de la calibración automática (ms) | `500` |
   | `CONTRASTE_MINIMO` | Diferencia mínima negro-blanco para aceptar la calibración de un sensor | `30` |
   | `kp`, `ki`, `kd`, `velocidad` | PID y velocidad iniciales; luego se cambian desde la app | `0.0128`, `0`, `0.089`, `128` |

   Los valores de PID, velocidad, umbral, color de línea y calibración que cambies desde la app se guardan en la memoria del robot y tienen prioridad sobre los del programa. Para volver a los del programa hay que borrar la memoria (**Herramientas > Erase All Flash Before Sketch Upload > Enabled** y cargar de nuevo).
3. Elige **Herramientas > Placa > esp32 > ESP32 Dev Module** y el puerto del robot en **Herramientas > Puerto**.
4. Pulsa **Subir**.
5. Abre el **Monitor serie** a 115200 baudios. Deben aparecer `Sensor de linea listo` y la dirección BLE del robot.

También se puede compilar y cargar desde la terminal con `arduino-cli`:

```bash
arduino-cli compile --fqbn esp32:esp32:esp32 ~/Documents/Arduino/libraries/ROBLEX/examples/ejemplos_robot/sensor_de_linea
```

```bash
arduino-cli upload --fqbn esp32:esp32:esp32 -p COM5 ~/Documents/Arduino/libraries/ROBLEX/examples/ejemplos_robot/sensor_de_linea
```

Cambia `COM5` por el puerto del robot (`arduino-cli board list` lo muestra).

## Primera prueba

Con el robot levantado, sin que las ruedas toquen el suelo:

1. Conecta la app y abre **Seguidor de línea**. Las 8 barras deben moverse al pasar la mano o una hoja blanca bajo los sensores.
2. Pasa una línea negra de izquierda a derecha bajo el robot: la posición debe ir de negativa (izquierda) a positiva (derecha). Si va al revés, cambia `INVERTIR_SENSORES`.
3. Con el joystick de la app, mueve el robot hacia adelante: las cuatro ruedas deben girar hacia adelante. Si un lado gira al revés, cambia el signo de ese lado en la función `motores()`.

LED RGB de la shield: verde = ve la línea, azul = no la ve, morado = calibrando, amarillo = a punto de arrancar, rojo = no encuentra el sensor de línea.

## Calibrar los sensores

Cada sensor lee distinto según su altura y la luz. La calibración guarda para cada uno su valor de negro y de blanco, y el robot los conserva al apagarse. Hay que repetirla al cambiar de pista o de iluminación.

- **Automática** (botón **Auto-calibrar** de la app): pon el robot con la línea bajo el centro. Gira solo a la derecha, a la izquierda y vuelve al centro, y termina solo.
- **Manual** (botón **Calibrar sensores** de la app, o mantener el botón de la shield 1 segundo): pasa el robot a mano para que todos los sensores vean la línea y el fondo, y vuelve a tocar el botón para terminar.

Al terminar, la tarjeta **Calibración guardada en el robot** de la app muestra el rango y el contraste de cada sensor. Un sensor en rojo no vio suficiente contraste y conserva su calibración anterior: repite la calibración pasándolo sobre la línea, o revisa que no esté demasiado alto o tapado.

## Calibrar el PID y la velocidad

En la app, sección **Parámetros**, la gráfica **Posición en el tiempo** muestra dónde estuvo la línea los últimos 5 segundos y el **error medio**: cuanto más bajo, más centrado va el robot. Las marcas rojas abajo son momentos en que perdió la línea.

1. Calibra los sensores y revisa que todos tengan contraste.
2. Pon la velocidad baja (80-100), Ki = 0 y Kd = 0, y pulsa **Enviar al robot**.
3. Sube Kp hasta que el robot siga la línea pero zigzaguee (la curva cruza el centro de lado a lado); luego bájalo un poco.
4. Sube Kd hasta que desaparezca el zigzag. Si el robot tiembla o vibra, Kd está demasiado alto.
5. Sube la velocidad de 10 en 10 o de 20 en 20 con el control **Velocidad**, que se aplica al soltarlo incluso con el robot en marcha, y repite los pasos 3 y 4. A más velocidad suele hacer falta más Kd.
6. Ki casi nunca hace falta. Úsalo muy bajo (0.0001) solo si en las rectas el robot va siempre corrido a un lado.

Compara ajustes en el mismo tramo de pista. Kp, Ki y Kd se aplican al pulsar **Enviar al robot**; la velocidad, al soltar su control.

## Uso sin la app

Botón de la shield:

- **Toque corto**: arranca 2 segundos después (para retirar la mano) o, si está en marcha, se detiene.
- **Mantener 1 segundo**: entra en calibración manual; otro toque la termina.

Usa los últimos valores guardados de PID, velocidad, umbral, color de línea y calibración.

## Protocolo BLE

Mensajes de texto, uno por línea, por el Nordic UART Service. Se pueden escribir a mano en la **Terminal** de la app.

App hacia robot:

| Comando | Efecto |
|---|---|
| `SEN,1` / `SEN,0` | Activa o apaga el envío de sensores |
| `PID,kp,ki,kd,vel` | Cambia el PID y la velocidad base (0-255) |
| `VEL,n` | Cambia solo la velocidad base (0-255), también en marcha |
| `UMB,n` | Cambia el umbral de detección (1-254) |
| `LIN,N` / `LIN,B` | Sigue línea negra sobre blanco / blanca sobre negro |
| `GET` | Pide PID, umbral, color, estado y calibración guardada |
| `RUN,1` / `RUN,0` | Arranca / detiene (detiene también la calibración automática) |
| `CAL,1` / `CAL,0` | Entra / termina la calibración manual (`CAL` alterna) |
| `CAL,A` | Calibración automática |
| `outR,outL,R,G,B` | Joystick: mueve el robot mientras no sigue la línea |

Robot hacia app:

| Mensaje | Significado |
|---|---|
| `SEN,s0,...,s7,pos` | Sensores (0-255) y posición de -3500 a 3500; sin `pos` si no ve la línea o calibrando |
| `PID,kp,ki,kd,vel` | PID y velocidad actuales |
| `VEL,n` | Velocidad aplicada, en respuesta a `VEL` |
| `UMB,n` | Umbral actual |
| `LIN,N` / `LIN,B` | Color de línea actual |
| `RUN,1` / `RUN,0` | Arrancó / se detuvo |
| `CAL,1` | Entró en calibración |
| `CAL,0,i,j...` | Terminó la calibración; `i`, `j` = sensores sin contraste |
| `CLB,n0..n7,b0..b7` | Calibración guardada: negro (`n`) y blanco (`b`) sin calibrar de cada sensor |

## Problemas frecuentes

| Síntoma | Causa probable y solución |
|---|---|
| Error al compilar: `SetupLine` o `LineValues` no declarados | El IDE usa una librería ROBLEX antigua. Reemplázala como se indica en [Instalar la librería ROBLEX](#instalar-la-librería-roblex). |
| LED rojo y "Sensor de linea no encontrado" | Revisa el cable I2C del módulo. El Monitor serie lista las direcciones I2C encontradas; si el módulo aparece en otra, cambia `LINEA_DIRECCION`. |
| El robot gira hacia el lado contrario de la línea | S0 está a la derecha (cambia `INVERTIR_SENSORES`) o un lado de motores gira al revés (cambia su signo en `motores()`). |
| Se detiene en curvas cerradas con "Linea perdida" | Sube `TIEMPO_SIN_LINEA`, baja la velocidad o sube Kp. |
| La calibración automática deja sensores en rojo | El giro no alcanzó a pasarlos sobre la línea: sube `VELOCIDAD_AUTOCAL` o `TIEMPO_GIRO_AUTOCAL`, o calibra a mano. |
| Confunde el fondo con la línea | Calibra, revisa el color de línea elegido y baja el umbral. |
