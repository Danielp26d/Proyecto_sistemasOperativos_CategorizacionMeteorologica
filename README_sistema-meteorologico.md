# Sistema de Categorización Meteorológica

**Pontificia Universidad Javeriana — Facultad de Ingeniería**  
**Departamento de Ingeniería de Sistemas**  
**Asignatura:** Sistemas Operativos  
**Período:** Febrero – Mayo 2026  

**Integrantes:**
- Daniel Prieto
- Esteban Cantillo
- Alejandro Macías
- Jorge Simental

---

## ¿De qué trata esto?

Este es el proyecto final del curso de Sistemas Operativos. Implementamos un sistema distribuido y concurrente en C que recibe mediciones meteorológicas de tres estaciones de Bogotá (Kennedy, Usaquén y Teusaquillo), las valida, las organiza y al final determina la categoría del clima.

Lo interesante del proyecto es que tuvimos que usar varias primitivas de concurrencia al mismo tiempo:

- **Named pipes (FIFO)** para que los procesos se puedan comunicar entre sí
- **Semáforos POSIX** para sincronizar el acceso a los buffers
- **Hilos POSIX (pthreads)** para procesar varias estaciones en paralelo
- **Buffers circulares** por estación, implementando el patrón productor/consumidor

El sistema lo desarrollamos para el Instituto Distrital de Gestión de Riesgos y Cambio Climático (según el enunciado del proyecto).

---

## Arquitectura general

```
 +------------------+   +------------------+   +------------------+
 |  agenteM (EK)    |   |  agenteM (ET)    |   |  agenteM (EU)    |
 |  Productor CSV   |   |  Productor CSV   |   |  Productor CSV   |
 +--------+---------+   +--------+---------+   +--------+---------+
          |                      |                       |
          +----------+-----------+-----------+-----------+
                                 |
                           write(FIFO)
                                 |
          +----------------------v------------------------------+
          |            Named Pipe (FIFO: pipeNom)              |
          +----------------------+------------------------------+
                                 |
                       +---------v---------+
                       |  Hilo Recolector  |
                       |   (read + parse)  |
                       +---------+---------+
           +-------------+--------+---------+-------------+
           |                      |                       |
  +--------v--------+   +---------v------+   +----------v-------+
  | Buffer Circular |   | Buffer Circular|   | Buffer Circular  |
  |  Estación EK    |   |  Estación ET   |   |  Estación EU     |
  | sem lleno/vacío |   | sem lleno/vacío|   | sem lleno/vacío  |
  +--------+--------+   +-------+--------+   +----------+-------+
           |                    |                        |
     Consumidor EK        Consumidor ET           Consumidor EU
           |                    |                        |
           +--------------------+------------------------+
                                |
                       consolidado.csv
                                |
                    Categorización Meteorológica
```

---

## Estructura del repositorio

```
sistema-meteorologico/
├── agenteM.c                       # Proceso agente: lee el CSV, valida y manda datos por el pipe
├── monitor.c                       # Monitor: crea el FIFO, maneja buffers, hilos y el reporte final
├── Makefile                        # Compila con GCC -Wall -Wextra -pthread
├── datos/
│   ├── kennedy.csv                 # Mediciones estación Kennedy   (código EK)
│   ├── usaquen.csv                 # Mediciones estación Usaquén   (código ET)
│   └── teusaquillo.csv             # Mediciones estación Teusaquillo (código EU)
├── docs/
│   └── Documentación_Proyecto.pdf  # Documentación técnica completa
└── README.md
```

> `consolidado.csv` y `pipeNom` se crean durante la ejecución y se borran con `make clean`.

---

## Componentes del sistema

### `agenteM.c` — Agente de Medición

Lee un archivo CSV de sensores, valida cada fila y envía las mediciones válidas al monitor a través del named pipe. Cuando encuentra la línea `.` sabe que terminó el archivo y cierra su extremo del pipe.

**Formato del CSV:**
```
ESTACION,HUMEDAD,ROCIO,PRESION,HH:MM:SS
EK,92,10,748,08:00:00
.
```

**Cómo usarlo:**
```bash
./agenteM -f <archivo.csv> -t <segundos> -p <pipeNom> [-k rm]
```

| Flag    | Descripción                                          |
|---------|------------------------------------------------------|
| `-f`    | Ruta del archivo CSV (obligatorio)                   |
| `-t`    | Segundos de espera entre lecturas (obligatorio)      |
| `-p`    | Nombre del pipe (obligatorio)                        |
| `-k rm` | Borra el archivo CSV al terminar (opcional)          |

**Rangos válidos por sensor:**

| Sensor              | Rango válido     | Unidad |
|---------------------|:----------------:|:------:|
| Humedad relativa    | 0 – 100          | %      |
| Punto de rocío      | −20 – 40         | °C     |
| Presión barométrica | 600 – 1100       | hPa    |

Si una medición está fuera de rango se descarta e imprime un aviso, pero el agente sigue ejecutándose.

---

### `monitor.c` — Monitor y Control de Categorización

Es el proceso principal. Crea el FIFO, lanza el hilo recolector que lee del pipe, distribuye las mediciones a los buffers por estación, y cuando todos los agentes terminan calcula los promedios y decide la categoría del clima.

**Cómo usarlo:**
```bash
./monitor -b <tamBuffer> -p <pipeNom>
```

| Flag  | Descripción                                    |
|-------|------------------------------------------------|
| `-b`  | Tamaño del buffer circular (obligatorio)       |
| `-p`  | Nombre del pipe (obligatorio)                  |

**Categorías meteorológicas (según promedios globales):**

| Categoría             | Condición                                          |
|-----------------------|----------------------------------------------------|
| Tormenta              | Humedad > 90% **y** Presión < 720 hPa              |
| Lluvia                | Humedad > 80% **y** Presión < 750 hPa              |
| Nublado               | Humedad > 70% **o** Presión < 760 hPa              |
| Parcialmente nublado  | Humedad 50–70% **y** Presión 760–780 hPa           |
| Despejado             | Humedad < 50% **y** Presión > 780 hPa              |

---

## Compilación

```bash
make          # Compila agenteM y monitor
make datos    # Crea archivos CSV de prueba en datos/
make clean    # Borra binarios, consolidado.csv y pipeNom
make help     # Muestra un resumen rápido de cómo usar el sistema
```

---

## Cómo ejecutar (necesita 4 terminales)

**Primero arrancar el monitor**, porque es el que crea el pipe. Si se lanza un agente antes, falla.

**Terminal 1:**
```bash
./monitor -b 4 -p pipeNom
```

**Terminal 2:**
```bash
./agenteM -f datos/kennedy.csv -t 1 -p pipeNom
```

**Terminal 3:**
```bash
./agenteM -f datos/usaquen.csv -t 1 -p pipeNom
```

**Terminal 4:**
```bash
./agenteM -f datos/teusaquillo.csv -t 1 -p pipeNom
```

Cuando el último agente termina, el monitor calcula los promedios, imprime el parte meteorológico y cierra.

---

## Verificar que funcionó

```bash
cat consolidado.csv       # Todas las mediciones válidas que procesó el sistema
wc -l consolidado.csv     # Con los datos de prueba debería dar ~24 líneas
```

**Salida esperada del monitor:**
```
... Control de Categorización Meteorológica!!!
[ALARMA] Estación EK: datos faltantes entre 13:00:00 y 14:00:00...
Parte Meteorológico Bogotá: "Nublado"
Fin del Monitor...!!!
```

---

## Mecanismos de sincronización

| Mecanismo          | Para qué lo usamos                                              |
|--------------------|-----------------------------------------------------------------|
| `sem_t lleno[i]`   | El consumidor espera aquí hasta que haya datos en el buffer `i` |
| `sem_t vacio[i]`   | El recolector espera aquí si el buffer `i` está lleno           |
| `pthread_mutex_t`  | Protege `ini`, `fin` y `count` del buffer para evitar race conditions |
| `pthread_join()`   | El monitor espera a que todos los consumidores terminen antes de imprimir el reporte |

---

## Requisitos

- Linux con soporte POSIX (probado en Ubuntu 22.04)
- GCC 9.x o superior
- Soporte para pthreads y semáforos POSIX (flag `-pthread`)

---

## Referencias

1. Stevens, W. R., & Rago, S. A. (2013). *Advanced Programming in the UNIX Environment* (3rd ed.). Addison-Wesley.
2. Butenhof, D. R. (1997). *Programming with POSIX Threads*. Addison-Wesley.
3. IEEE Std 1003.1-2017: *POSIX Specification*. The Open Group. https://pubs.opengroup.org/onlinepubs/9699919799/
4. Linux man pages. https://man7.org/linux/man-pages/
5. Notas de clase: Sistemas Operativos 2026-30. Pontificia Universidad Javeriana.
