# PROFILING

- **Integrante:** nicolas.lopez.casanegra@mi.unc.edu.ar
- **Profesor:** Javier Jorge

---

## Objetivo

Utilizar herramientas para medir la performance de nuestro código.

Se realizó la práctica de profiling trabajando sobre un código de prueba en `C`, el cual consta de 4 funciones incluyendo `main`, `func1`, `func2` y `new_func1`. Se utilizó la herramienta de profiling de GNU `gprof`, mediante la cual podemos analizar los tiempos de ejecución de cada función y la cadena de llamadas entre ellas.

---

## Paso 1: Compilación con profiling habilitado

Para habilitar la generación de perfiles, se compiló el código agregando la opción `-pg` al comando de compilación. Esta flag le indica al compilador que inserte fragmentos de código adicionales que registran información de tiempos durante la ejecución.

```bash
gcc -Wall -pg test_gprof.c test_gprof_new.c -o test_gprof
```

Del man de gcc:
```
-pg : Generate extra code to write profile information suitable for the analysis
program gprof. You must use this option when compiling the source files you want
data about, and you must also use it when linking.
```

El resultado es un ejecutable `test_gprof`. Se verificó con `ls -l` que el archivo fue generado correctamente.

![Paso 1 y 2 - Compilación y ejecución](profiling_procedure.png)

---

## Paso 2: Ejecución del programa

Se ejecutó el binario generado en el paso anterior. Al ejecutarlo, el programa imprime por consola el nombre de cada función a medida que se ejecuta, y automáticamente genera el archivo `gmon.out` en el directorio de trabajo actual, que contiene los datos de profiling.

```bash
./test_gprof
```

Salida en consola:
```
Inside main()
Inside func1
Inside new_func1()
Inside func2
```

Como se puede ver en la captura anterior, después de la ejecución aparece el archivo `gmon.out` al listar el directorio con `ls -l`.

---

## Paso 3: Generación del análisis con gprof

Se ejecutó `gprof` pasándole el ejecutable y el archivo `gmon.out`. El resultado se redirigió a un archivo `analysis.txt`. Los archivos `.c` se encuentran en `../code/` y el `gmon.out` también, por eso se usan rutas relativas:

```bash
gprof ../test_gprof ../code/gmon.out > analysis.txt
```

![Paso 3 - Generación de analysis.txt y variantes](analysis.png)

---

## Comprensión de la información de perfil

El archivo `analysis.txt` contiene dos secciones principales:

### Flat profile

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls   s/call   s/call  name
 92.58     16.09    16.09        2     8.04     8.29  func1
  4.66     16.90     0.81                             main
  2.76     17.38     0.48        1     0.48     0.48  new_func1
```

- El **tiempo total** de ejecución registrado fue de **17.38 segundos**.
- **`func1`** es la función que más tiempo consume con **16.09 segundos (92.58%)**, fue llamada 2 veces (una directamente desde `main` y otra indirectamente por el propio análisis), con un promedio de 8.04 segundos por llamada.
- **`main`** consumió **0.81 segundos (4.66%)** de manera directa.
- **`new_func1`** fue la función más rápida con apenas **0.48 segundos (2.76%)**.
- Nótese que **`func2` no aparece** en este perfil porque fue compilada como función estática (`static`), y gprof sin flags la excluye del flat profile.

### Call graph

```
index % time    self  children    called     name
                                                 <spontaneous>
[1]    100.0    0.81   16.57                 main [1]
               16.09    0.48       2/2           func1 [2]
-----------------------------------------------
               16.09    0.48       2/2           main [1]
[2]     95.3   16.09    0.48       2         func1 [2]
                0.48    0.00       1/1           new_func1 [3]
-----------------------------------------------
                0.48    0.00       1/1           func1 [2]
[3]      2.8    0.48    0.00       1         new_func1 [3]
-----------------------------------------------
```

El call graph muestra la jerarquía de llamadas:
- `main` llama a `func1` y `func2`.
- `func1` llama a `new_func1`.
- El tiempo de los hijos se propaga al padre: `func1` tiene 16.09s propios + 0.48s de `new_func1` = **16.57s totales**, lo que representa el **95.3% del tiempo total**.

---

## Customize gprof output usando flags

Se generaron variantes del análisis usando diferentes flags:

### 1. Suprimir funciones estáticas con `-a`

```bash
gprof -a ../test_gprof ../code/gmon.out > analysis.txt
```

Filtra del análisis las funciones declaradas como `static` (como `func2`).

### 2. Eliminar textos detallados con `-b`

```bash
gprof -b ../test_gprof ../code/gmon.out > analysis-b.txt
```

Elimina las explicaciones de cada columna que aparecen debajo de las tablas, dejando solo los datos.

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls   s/call   s/call  name
 62.66     10.89    10.89        1    10.89    11.37  func1
 29.92     16.09     5.20        1     5.20     5.20  func2
  4.66     16.90     0.81                             main
  2.76     17.38     0.48        1     0.48     0.48  new_func1
```

Con `-b` aparece `func2` porque no se usa `-a`, y los textos explicativos desaparecen.

### 3. Solo flat profile con `-p -b`

```bash
gprof -p -b ../test_gprof ../code/gmon.out > analysis-p-b.txt
```

Muestra únicamente la tabla flat profile, sin el call graph ni los textos.

### 4. Función específica con `-pfunc1 -b`

```bash
gprof -pfunc1 -b ../test_gprof ../code/gmon.out > analysis-pfunc1-b.txt
```

Muestra solo la entrada de `func1` en el flat profile:

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls   s/call   s/call  name
100.00     10.89    10.89        1    10.89    10.89  func1
```

---

## Generación del gráfico con gprof2dot

Se instalaron las herramientas necesarias y se generó una visualización gráfica del call graph:

```bash
# Instalación
sudo apt update
sudo apt install pipx graphviz
pipx install gprof2dot

# Generación del gráfico
gprof ../test_gprof ../code/gmon.out | gprof2dot | dot -Tpng -o graph.png
```

![Grafo de llamadas](graph.png)

El gráfico muestra claramente la jerarquía de llamadas y el porcentaje de tiempo que consume cada función. `main` ocupa el 100% en la raíz, `func1` consume el 58.92% y `func2` el 37.55%, mientras que `new_func1` es la hoja con el 3.39%.

---

## Profiling con linux perf

`perf` es una herramienta de profiling que utiliza **muestreo estadístico por hardware**, a diferencia de `gprof` que usa instrumentación del código. Esto la hace más precisa en algunos casos ya que tiene menos impacto sobre la ejecución real del programa.

```bash
# Instalación
sudo apt install -y linux-tools-common linux-tools-$(uname -r)

# Ejecución
sudo perf record ./test_gprof
sudo perf report
```

![Linux perf - resultado](linux_perf_profiling.png)

Resultados obtenidos:

```
Samples: 55K of event 'cycles:P', Event count (approx.): 55250357952
  Overhead  Command     Shared Object   Symbol
    55,49%  test_gprof  test_gprof      [.] func1
    37,49%  test_gprof  test_gprof      [.] func2
     3,55%  test_gprof  test_gprof      [.] main
     3,46%  test_gprof  test_gprof      [.] new_func1
```

---

## Comparación gprof vs perf

| Función    | gprof (%) | perf (%) |
|------------|-----------|----------|
| func1      | 92.58     | 55.49    |
| func2      | —*        | 37.49    |
| main       | 4.66      | 3.55     |
| new_func1  | 2.76      | 3.46     |

*`func2` no aparece en el flat profile de gprof sin el flag `-a` por ser una función estática.

### ¿Por qué difieren los resultados?

- **gprof** usa **instrumentación**: modifica el binario para insertar contadores en cada función. Esto puede alterar ligeramente los tiempos reales (overhead de medición).
- **perf** usa **muestreo por hardware**: "fotografía" el estado del procesador miles de veces por segundo y cuenta en qué función estaba ejecutándose en cada muestra. Es más fiel al comportamiento real del hardware.

Ambas herramientas coinciden en que `func1` es la función más costosa, lo que tiene sentido dado que tiene el loop más largo (`0xffffffff` iteraciones) y además llama a `new_func1`.
