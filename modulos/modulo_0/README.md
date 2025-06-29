# Setup y herramientas

## Visual Studio Code para Raspberry Pi Pico

El entorno de desarrollo para la Raspberry Pi Pico y  Raspberry Pi Pico 2 va a ser Visual Studio Code. Podemos descargarlo desde este [link](https://code.visualstudio.com/).

Desde 2024, Raspberry Pi desarrolló una extensión oficial para las Raspberry Pi Pico en Visual Studio Code. Podemos buscarla e instalarla desde el menú de extensiones que se encuentra a la izquierda buscando _Raspberry Pi Pico_.

![Raspberry Pi Pico extension](https://www.raspberrypi.com/app/uploads/2024/10/extension-1024x640.png)

> :warning: Esta extensión requiere unas dependencias previas para algunos sistemas operativos. Ver [dependencias](#dependencias).

Una vez que tengamos instalada la extensión, vamos a al menú de esta y vamos a crear un proyecto:

![Create a Raspberry Pi Pico project](https://www.raspberrypi.com/app/uploads/2024/10/create.png)

Del menú de creación de proyecto vamos a tener que elegir:

* Nombre del proyecto
* Tipo de placa que debe corresponder con la placa que vayamos a usar
* Arquitectura (solo para Pico 2 y Pico 2W) que solamente vamos a tildar si usamos arquitectura RISC-V
* Ubicación que es donde va a terminar creándose el proyecto. Normalmente lo vamos a crear dentro de alguno de los directorios del repositorio
* Versión de SDK de la Pico (actualmente v2.1.1)
* Features que nos sirve para habilitar periféricos particulares pero normalmente vamos a habilitarlos desde otro lado
* Soporte para mensajes por consola (stdio) por UART o USB que también habilitaremos desde otro lado
* Opciones adicionales de generación de código que normalmente no vamos a usar
* Debugger que siempre vamos a estar eligiendo CMSIS-DAP

Una vez que elegimos estas cosas, creamos el proyecto y, si es la primera vez que creamos uno, esperamos a que se descargue todo el SDK para el microcontrolador que vamos a usar.

### Estructura de proyecto

Un proyecto creado tiene esta estructura:

```
.
├── .vscode/
│   └── ...
├── build/ 
│   └── ...
├── .gitignore
├── CMakeLists.txt
├── pico_sdk_import.cmake
└── main.c
```

En esta estructura, son particularmente importantes los archivos _CMakeLists.txt_ y el _main.c_, siendo el último donde vamos a escribir nuestra aplicación principal.

### Dependencias

#### Windows

Para Windows no hay dependencias pero si hay que posteriormente instalar el driver de USB para la Raspberry Pi Pico. Una vez que la tengamos conectada, podemos descargar [Zadig](https://zadig.akeo.ie/) y actualizar el driver al apropiado.

#### Linux

Las dependencias se pueden instalar por consola con:

```bash
sudo apt install python3 git tar build-essential
```

#### macOS

Instalamos xcode con:

```bash
xcode-select --install
```

## Biblioteca CMSIS

Common Microcontroller Software Interface Standard (CMSIS) permite un soporte consistente entre microcontroladores de la misma arquitectura. Parte de este conjunto de bibliotecas nos va a permitir incluir bibliotecas de DSP.

![Capas de CMSIS](https://arm-software.github.io/CMSIS_6/latest/General/cmsis_components.png)

### Submódulos de CMSIS_6 y CMSIS-DSP

Podemos traer estas bibliotecas como submódulos del proyecto escribiendo:

```
git submodule init
git submodule update
```

Esto va a traer los repositorios de [CMSIS_6](https://github.com/ARM-software/CMSIS_6) y [CMSIS-DSP](https://github.com/ARM-software/CMSIS-DSP).

## Usar CMSIS-DSP en un proyecto

Ambas bibliotecas ya tienen sus reglas de compilación en su propio CMakeLists.txt. Para incluirlas, hay que definir en el CMakeLists.txt del proyecto que creemos la ruta a `CMSISCORE` desde este proyecto y agregar el directorio de `CMSIS-DSP` como una biblioteca. Para eso, agregamos al final del archivo:

```cmake
# Setea la variable CMSISCORE al directorio de CMSIS desde el proyecto
set(CMSISCORE ${CMAKE_SOURCE_DIR}/../../../CMSIS_6/CMSIS/Core)
# Agrega el directorio de CMSIS-DSP como biblioteca del proyecto
add_subdirectory(${CMAKE_SOURCE_DIR}/../../../CMSIS-DSP ${CMAKE_BINARY_DIR}/CMSIS-DSP)
# Linkea CMSIS-DSP al proyecto
target_link_libraries(${PROJECT_NAME} CMSISDSP)
```

> ⚠️ Estas rutas del ejemplo son relativas a un proyecto que esté creado dentro del directorio del módulo donde se esté trabajando. 

## Ejemplo para compilar

Lo más sencillo que podemos intentar hacer para probar que efectivamente anda el paquete de CMSIS-DSP en nuestro microcontrolador es un programa que haga una suma de dos vectores aprovechando instrucciones de la biblioteca.

### Creación de proyecto

Vamos a crear un proyecto nuevo dentro de este directorio que vamos a llamar `00_hello_cmsis`. Vamos también a habilitar soporte para stdio por USB tildando la casilla _Console over USB_ en Stdio support.

Una vez creado el proyecto vamos a poner este código de ejemplo en el programa principal [00_hello_cmsis.c](./00_hello_cmsis/00_hello_cmsis.c):

```c
#include <stdio.h>
#include "pico/stdlib.h"
#include "arm_math.h"

/**
 * @brief Programa principal
 */
int main(void) {
    // Inicialización de USB
    stdio_init_all();
    sleep_ms(1000);

    // Vectores de entrada
    float32_t vector_a[3] = {1.0f, 2.0f, 3.0f};
    float32_t vector_b[3] = {4.0f, 5.0f, 6.0f};
    // Vector de salida para la suma
    float32_t result[3];

    puts("\nHello CMSIS!");
    puts("Suma de vectores:\n");

    // Suma de vectores de CMSIS
    arm_add_f32(vector_a, vector_b, result, 3);

    // Mostrar resultado
    for (int i = 0; i < 3; i++) {
        printf("result[%d] = %f\n", i, result[i]);
    }

    while(1);
}
```

Luego, compilamos el proyecto, grabamos el firmware en la Raspberry Pi Pico y luego podemos abrir el monitor serial de VS Code para poder ver los mensajes y corroborar el resultado.

## Entorno virtual para Jupyter Notebook

La biblioteca de CMSIS-DSP viene con un [Python Wrapper](../../CMSIS-DSP/PythonWrapper_README.md) que provee un conjunto de funciones de DSP que se asemejan mucho a las que vamos a usar en el microcontrolador para facilitar la migración de una plataforma a otra. Vamos a armar un entorno virtual de Python para poder trabajar con este módulo de DSP y Jupyter Notebook. Para eso, vamos a tener que tener instalado [Python](https://www.python.org/) y el paquete de [virtual environment](https://pypi.org/project/virtualenv/):

```bash
python3 -m pip install virtualenv
```

Luego, podemos crear el entorno virtual dentro de este repositorio como:

```bash
python3 -m venv .env
```

Esto va a crear un directorio oculto `.env` donde vamos a instalar los paquetes de Python que vamos a necesitar para los módulos didácticos. Hacemos esto activando el entorno virtual:

```bash
source .env/bin/activate
```

Luego, instalamos los paquetes que están en el archivo [requirements.txt](../../requirements.txt):

```bash
pip install -r requirements.txt
```

Después, instalamos un kernel para el backend de Jupyter en el entorno virtual y reiniciamos VS Code para poder abrir nuestra [Jupyter Notebook](./desafios.ipynb):

```bash
python3 -m ipykernel install --user --name=.env
```