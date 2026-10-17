# Informe final: Mandelbrot distribuido con Master-Worker y Bag of Tasks

**Integrantes:** Juan Camilo Borrero, Anna Sophia Caicedo y Andrés Martínez
> Los datos salen de la consola del cliente (tabla del Exp. 1 y bloques `RESULTADOS DEL BENCHMARK`), con corridas del 2026-10-16. Las imágenes generadas están en `client/output/e2.png` y `client/output/e3.png`, y las gráficas en `docs/img/`.


## 1. Entorno experimental

- **Región y parámetros:** `seahorse`, resolución 4K (3840×2160) y maxIter = 5000.
- **Montaje:** el Master corre en el PC del cliente (por Wi-Fi o hotspot) y cada worker en un PC distinto.
- **Tiempo secuencial:** en el Exp. 1, T_sec = **79 134 ms** (monolito con 1 hilo, en el PC del Master, que tiene 16 hilos lógicos). En los Exp. 2 y 3 el cliente vuelve a medir el monolito en cada corrida y T_sec varió entre **80 402 y 83 845 ms** (media ≈ 81 300 ms, dispersión ≈ 4 %). Por eso cada S se calculó con el T_sec de su propia corrida.
- **Validación:** todas las corridas se compararon contra el monolito y dieron `VERIFICACION OK`.
- **Métricas:** S = T_sec / T_dist y E = S / p · 100 %. En los tres experimentos hay una sola medición por celda, salvo en el Exp. 3, que tiene dos.

## 2. Experimento 1: granularidad (p = 2)

Se usaron Worker-3 y Worker-2 (6 hilos cada uno) sobre `seahorse` en 4K con 5000 iteraciones, y T_sec = 79 134 ms. Todas las corridas quedaron validadas.

**Tabla 1.** Impacto de la granularidad (una medición por celda, 2026-10-16):

| Estrategia | Parámetro | Tareas | T_sec (ms) | T_dist (ms) | Speedup (S) | Eficiencia (E) |
|---|---|---|---|---|---|---|
| FixedCount (franjas) | N = 2 | 2 | 79 134 | 47 429 | 1.67× | 83.4 % |
| FixedCount (franjas) | N = 4 | 4 | 79 134 | 29 560 | 2.68× | 133.9 % |
| FixedCount (franjas) | N = 8 | 8 | 79 134 | 19 858 | 3.98× | 199.2 % |
| FixedCount (franjas) | N = 16 | 16 | 79 134 | 19 589 | 4.04× | 202.0 % |
| FixedTileSize (grano ultrafino) | 32×32 | 8 160 | 79 134 | 45 334 | 1.75× | 87.3 % |
| FixedTileSize | 64×64 | 2 040 | 79 134 | 23 246 | 3.40× | 170.2 % |
| FixedTileSize (grano intermedio) | 128×128 | 510 | 79 134 | 17 495 | 4.52× | 226.2 % |
| FixedTileSize (grano grueso) | 256×256 | 135 | 79 134 | 15 558 | 5.09× | 254.3 % |

![S vs granularidad](docs/img/s_vs_granularidad.svg)

**Lo que se observa:**

- **Baldosas.** El speedup sube a medida que crece la baldosa: la peor configuración es 32×32 (45 334 ms, S = 1.75×) y la mejor es 256×256 (15 558 ms, S = 5.09×). Pasar de 32×32 a 64×64 reduce el tiempo casi a la mitad (−49 %), mientras que de 128×128 a 256×256 la mejora es de solo −11 %. Esto sugiere que, con baldosas grandes, el costo por tarea ya no domina.
- **Franjas.** Con N = 2 y N = 4 el resultado es flojo (1.67× y 2.68×). Desde N = 8 el tiempo se estabiliza alrededor de 19.6–19.9 s, y pasar de 8 a 16 franjas mejora apenas un 1.4 %.
- **Mejor configuración.** Las baldosas de 256×256 superan en un 21 % a la mejor partición en franjas (15 558 vs. 19 589 ms). Aun así, en los Exp. 2 y 3 usamos 128×128, tanto por coherencia con la guía como porque sus 510 tareas dejan más margen de balanceo cuando se agregan workers. Con 135 tareas grandes, el reparto del final se vuelve más irregular a medida que crece p.
- Casi todas las celdas pasan del 100 % porque T_sec se midió con **un solo hilo** y cada worker usa 6. Como E = S/p cuenta nodos y no núcleos, sirve para comparar filas entre sí, no como eficiencia absoluta.
- **Limitaciones.** Hay una sola medición por celda, así que las diferencias pequeñas (como N = 8 vs. N = 16, de 1.4 %) caben dentro de la variación que ya vimos en T_sec (≈ 4 %).

## 3. Experimento 2: hilos por worker

Aquí se usó Worker-3 (6 núcleos físicos y 12 hilos lógicos) como único worker, con baldosas de 128×128 y una corrida por configuración, todas validadas. Como hay un solo worker, la eficiencia que imprime el programa es E = S · 100 %. Además se calculó S_rel = T(1 hilo) / T(h).

| Hilos | T_sec (ms) | T_dist (ms) | S = T_sec/T_dist | S_rel = T(1)/T(h) |
|---|---|---|---|---|
| 1 | 83 845 | 88 075 | 0.95× | 1.00× |
| 2 | 80 671 | 46 849 | 1.72× | 1.88× |
| 6 (núcleos físicos) | 80 402 | 33 740 | 2.38× | 2.61× |
| 12 (hilos lógicos) | 80 731 | 20 249 | 3.99× | 4.35× |
| 64 (sobredimensionado) | 81 146 | 18 102 | 4.48× | 4.87× |

La guía pide probar 1, 2, los núcleos físicos (6) y 64 hilos. Los 12 hilos son una medición extra que agregamos nosotros.

![S vs hilos](docs/img/s_vs_hilos.svg)

**Lo que se observa:**

- **Con 1 hilo, el sistema distribuido es más lento que el monolito** (88 075 vs. 83 845 ms, S = 0.95×). Con exactamente el mismo cómputo, el sistema paga unos 4.2 s extra (≈ 5 %) en RPC, serialización y consolidación en el Master. Es una medida directa del sobrecosto fijo del middleware.
- **De 1 a 2 hilos** el tiempo baja 1.88×, bastante cerca del ideal (2×).
- **De 2 a 6 hilos** la mejora es de solo 1.39× (el ideal sería 3×), pero **de 6 a 12** el salto es de 1.67×, más de lo que normalmente da el hyperthreading (≈ 1.2–1.3×). Con una sola corrida por celda no podemos sacar una conclusión firme. Lo más probable es que la corrida de 6 hilos haya salido penalizada por variaciones del PC o de la red, algo parecido a lo que ya vimos entre sesiones en T_sec.
- **De 12 a 64 hilos** T_dist mejora solo 1.12× (20 249 → 18 102 ms). Con 12 hilos la CPU ya estaba saturada, así que los 52 hilos adicionales solo se turnan las mismas unidades de ejecución (ver P5).
- **Limitación.** Hay una corrida por celda y no sacamos el cómputo por baldosa del log del Master. Por eso el costo de los cambios de contexto lo argumentamos con el tiempo de pared.

## 4. Experimento 3: escalabilidad y Ley de Amdahl

Se usaron baldosas de 128×128, 6 hilos por worker y 2 ejecuciones por configuración (2026-10-16). Cada corrida se verificó contra el monolito.

| p  | T_dist c1 (ms) | T_dist c2 (ms) | Media (ms) | S_rel = T(1)/T(p) | E_rel = S_rel/p |
| :- | :------------- | :------------- | :--------- | :---------------- | :-------------- |
| 1  | 23 656         | 22 532         | 23 094.0   | 1.000             | 1.000           |
| 2  | 15 553         | 15 886         | 15 719.5   | 1.469             | 0.735           |
| 4  | 7 853          | 7 897          | 7 875.0    | 2.933             | 0.733           |

### 4.1 Speedup y eficiencia

El tiempo promedio baja de forma clara al agregar workers: con 2 workers el speedup relativo es de **1.469×** y con 4 llega a **2.933×** frente a un solo worker. Repartir la carga ayuda bastante, aunque la mejora no es lineal.

La eficiencia paralela se calcula como **E = S/p**. Por eso, cuando el benchmark imprime la eficiencia como `S · 100 %`, ese número es en realidad el speedup en porcentaje y no la eficiencia por worker. Con nuestros datos, la eficiencia es de aproximadamente **73.5 % con 2 workers** y **73.3 % con 4 workers**.

Estos tiempos son del mismo orden que los del Exp. 2 con una configuración parecida (128×128 y 6 hilos). Aun así, la corrida de 6 hilos del Exp. 2 (33 740 ms) fue más lenta que la media de p = 1 aquí (23 094 ms), y no tenemos medido qué la causó. Las diferencias pueden venir de la variación normal entre corridas, la carga del sistema, la comunicación entre procesos y el costo de repartir las tareas.

### 4.2 Ley de Amdahl y Karp-Flatt

Para ver qué parte del programa limita la escalabilidad usamos la Ley de Amdahl:

$$
\frac{1}{S}=(1-f)+\frac{f}{p}
$$

junto con la métrica de Karp-Flatt:

$$
e(p)=\frac{\frac{1}{S}-\frac{1}{p}}{1-\frac{1}{p}}
$$

| p  | S_rel | e(p)  |
| :- | :---- | :---- |
| 2  | 1.469 | 0.361 |
| 4  | 2.933 | 0.121 |

Con 2 workers, e(p) ≈ 0.361, y con 4 baja a ≈ 0.121. En Karp-Flatt, una fracción serial real debería mantenerse más o menos constante al subir p, así que esta caída indica que el modelo no se ajusta limpio. Lo más probable es que p = 2 haya salido peor de lo esperado (su eficiencia es de 73.5 %, casi igual a la de p = 4) por la diferencia de velocidad entre los PCs, o por efectos de caché y de red. Las dos corridas de p = 1 difieren cerca de un 5 % y las de p = 2 cerca de un 2 %, y no registramos qué PC corrió cada worker. Por eso no tomamos estos valores como una medida firme de la fracción secuencial.

La estimación más estable viene del Exp. 2: la ejecución con un solo hilo cuesta un poco más que el monolito, y comparando 1 y 2 hilos Karp-Flatt da e ≈ 0.064. Con eso, la fracción realmente no paralelizable queda en el rango de **5 % a 6 %**, algo por debajo del e(4) = 0.121 medido con workers, que seguramente incluye costos de comunicación y de nodos heterogéneos. Según Amdahl, eso limita el speedup máximo teórico a unas **16×–20×**, por mucho que se aumenten los recursos.

Esto es solo una aproximación. Influyen otros factores como las diferencias de rendimiento entre nodos, la comunicación por RPC, la consolidación en el Master, la entrada y salida, y la variabilidad entre corridas. Además, con solo dos corridas por configuración podemos ver la tendencia general, pero no caracterizar bien la variabilidad del sistema.

## 5. Cuestionario

### P1. La trampa del espacio vacío vs. zona de estrés

En 4K y con 5000 iteraciones, el monolito tardó entre 79 y 84 segundos en la región `seahorse`. Con la misma resolución y el mismo límite de iteraciones, la vista `classic` tardó bastante menos.

La diferencia está en cómo se reparte el trabajo. En `classic`, la mayoría de los puntos queda fuera del conjunto y escapa rápido, así que casi nunca llega al máximo de iteraciones. En `seahorse`, en cambio, predominan los puntos cercanos a la frontera, que necesitan muchas más iteraciones y pueden acercarse a las 5000.

Por eso, mirar la imagen completa puede hacer que se subestime el costo de cómputo. Al aumentar la magnificación, el número de operaciones puede crecer mucho, y ahí se vuelve más evidente la necesidad de repartir el procesamiento entre varios workers.

**Nota:** el monolito ajusta el intervalo horizontal para mantener la relación de aspecto 16:9. Por eso la vista `classic` cubre $X \in [-2.0, 2.44]$.

### P2. Colapso por grano fino vs. sobrecosto de red

Con p = 2 y la región `seahorse`, el tiempo mejora cada vez que agrandamos la baldosa, aunque el trabajo de cálculo total es el mismo:

| Tamaño | Tareas | $T_{dist}$ | Tiempo extra vs. 256×256 | Extra por tarea adicional |
|---|---:|---:|---:|---:|
| 32×32 | 8 160 | 45 334 ms | 29 776 ms | ≈ 3.7 ms |
| 64×64 | 2 040 | 23 246 ms | 7 688 ms | ≈ 4.0 ms |
| 128×128 | 510 | 17 495 ms | 1 937 ms | ≈ 5.2 ms |
| 256×256 | 135 | 15 558 ms | – | – |

Las dos últimas columnas las calculamos nosotros: restamos el tiempo de 256×256 y lo dividimos entre las tareas que se agregan respecto a esa configuración. Sale un costo de unos 4 ms de tiempo de pared por cada tarea extra, bastante parecido en las tres filas. Por eso pensamos que la degradación de 32×32 (S = 1.75× frente a 5.09× con 256×256) no viene de que haya más cálculo, sino de un costo fijo que se paga una vez por tarea. Es una estimación gruesa: hay una sola medición por celda y parte del tiempo extra también puede venir de un reparto menos parejo al final.

Ese costo fijo por tarea tiene tres partes que identificamos en el código:

- **Dos llamadas remotas por baldosa.** Cada tarea pasa por `pollTask` y por `reportResult`. Con 8 160 baldosas son más de 16 000 llamadas, cada una con su encabezado de Ice, su viaje por TCP y su espera de respuesta. En una red Wi-Fi o hotspot ese viaje puede tardar más que calcular una baldosa de una zona simple.
- **Armar y desarmar mensajes.** Cada llamada serializa un `TaskTile` o un `TileResult`. En 32×32 el resultado son solo 1 024 enteros (unos 4 KB), así que el trabajo útil es pequeño frente a los campos fijos (identificador, coordenadas, tamaños) que se repiten en cada mensaje.
- **Un solo punto de coordinación.** Todos los hilos de todos los workers piden tareas a la misma `ConcurrentLinkedQueue` y entregan resultados al Master, que los copia a la matriz final. Con miles de reportes pequeños, el Master puede pasar de coordinar a convertirse en cuello de botella.

El otro extremo también tiene un límite: pasar de 128×128 a 256×256 mejora solo un 11 %, mientras que de 32×32 a 64×64 el tiempo baja un 49 %. Eso nos hace pensar que el mejor tamaño está en la zona intermedia-grande y que a partir de ahí la ganancia es pequeña.

**Limitación:** no medimos por separado la serialización, la red y la contención, así que no podemos decir cuánto aporta cada una. La explicación se apoya en los tiempos totales y en cómo está construido el sistema.

### P3. Fracaso de la partición estática

`FixedCount` quedó por debajo de la distribución dinámica de baldosas. Con 2 particiones el tiempo fue de 47 429 ms (S = 1.67×) y con 4 bajó a 29 560 ms (S = 2.68×).

El problema es que dividir la imagen en pocas franjas grandes no asegura que cada worker reciba la misma cantidad de trabajo. Algunas franjas cruzan la frontera del conjunto y exigen muchas más iteraciones, mientras que otras tienen puntos que escapan rápido. Entonces el tiempo total lo marca la franja más lenta, aunque los demás workers ya hayan terminado. Además, con solo 2 particiones tampoco hay garantía de que los dos workers reciban una franja de forma efectiva.

Con 8 y 16 particiones los tiempos bajaron a 19 858 ms y 19 589 ms. El reparto mejora, pero ambos resultados siguen siendo cerca de un 21 % más lentos que los 15 558 ms de las baldosas de 256×256.

*Bag of Tasks* balancea de forma más flexible: cada worker pide una nueva tarea apenas termina la anterior. Así, los workers más rápidos procesan más baldosas y los más lentos no frenan al resto del sistema.

### P4. Puntos de quiebre del hardware y la memoria

#### a) Precisión de `double`

El problema aparece cuando dos píxeles vecinos terminan con el mismo valor de `c`. En `computePoint` la coordenada se calcula como `xMin + x·dx`, y un `double` solo distingue números que difieran en más de su epsilon relativo (≈ $2.2 \times 10^{-16}$, unas 15–16 cifras decimales). Cerca de `xMin ≈ 0.75`, el menor paso representable es de unos $1.1 \times 10^{-16}$.

Con nuestro ancho de 3840 píxeles, `dx` llega a ese valor cuando el ancho de la región baja a unos $4 \times 10^{-13}$. Frente a una vista clásica de ancho cercano a 2.5, eso equivale a una magnificación del orden de $10^{12}$–$10^{13}$.

En `seahorse` (ancho 0.01) el paso entre píxeles es de unos $2.6 \times 10^{-6}$, unas $10^{10}$ veces mayor que ese mínimo, así que este límite no afecta nuestras pruebas. Si se cruzara, los píxeles repetidos se verían como bloques planos de un solo color, patrones en cuadrícula y filamentos que desaparecen.

Cambiar a `BigDecimal` quitaría el problema porque su precisión se configura, pero cada suma o multiplicación deja de ser una instrucción del procesador y pasa a ser una operación sobre objetos con memoria asignada. Estimamos que cada iteración costaría uno o dos órdenes de magnitud más, lo que haría muy caro un 4K con 5000 iteraciones.

**Limitación:** este umbral es un cálculo teórico; no probamos un zoom extremo para verlo.

#### b) Consumo de memoria RAM

La matriz de iteraciones es un `int[][]`, así que ocupa aproximadamente ancho × alto × 4 bytes, sin contar la sobrecarga propia de los arreglos de Java.

| Resolución | Píxeles | Una matriz | Tres copias |
|---|---:|---:|---:|
| 4K (3840×2160) | 8.29 millones | 33 MB | ≈ 100 MB |
| 16K (15360×8640) | 132.7 millones | 531 MB | ≈ 1.6 GB |
| 32K (30720×17280) | 530.8 millones | 2.1 GB | ≈ 6.4 GB |

La columna de tres copias es un mínimo que calculamos nosotros: la matriz consolidada en el Master, la matriz de resultado en el cliente y la matriz del monolito con la que el cliente verifica. A eso se suman las copias temporales y los buffers de Ice al serializar.

Como no fijamos `-Xmx` en `build.gradle`, el heap máximo lo decide la JVM, normalmente un cuarto de la RAM física. Por ejemplo, con 16 GB de RAM el heap rondaría los 4 GB, y un 32K con tres copias (≈ 6.4 GB) ya no cabría. En ese caso la JVM termina con `java.lang.OutOfMemoryError: Java heap space`.

Hay otro tope antes que la memoria: `Ice.MessageSizeMax`, que por defecto es de 1 MB. Nuestra baldosa más grande (256×256) pesa 256 KB y pasa sin problema, pero una de 512×512 ya llegaría a 1 MB y se saldría del límite, igual que cualquier intento de enviar una matriz completa en un solo mensaje.

**Limitación:** no probamos 16K ni 32K; las cifras son estimaciones.

### P5. Límites del ThreadPool y Ley de Amdahl

#### Pool de 64 hilos

Worker-3 tiene seis núcleos físicos y doce hilos lógicos. Al subir el pool de 12 a 64 hilos, T_dist pasó de 20 249 ms a 18 102 ms, una mejora de solo 1.12×. En cambio, de 1 a 2 hilos la mejora había sido de 1.88×.

La razón es que los hilos de software superan por mucho los recursos de ejecución disponibles, y el sistema operativo tiene que alternar entre hilos que compiten por los mismos núcleos.

Esos cambios de contexto obligan a guardar y restaurar el estado de los hilos, y pueden reducir la eficiencia de las cachés L1/L2 y del TLB. Además, con hyperthreading, los hilos lógicos de un mismo núcleo comparten recursos físicos. Administrar los hilos consume tiempo que no aporta al cálculo de píxeles.

Cuando los 12 hilos lógicos ya mantienen la CPU ocupada, agregar más da beneficios limitados. La sobrecarga puede crecer aún más si cada hilo necesita memoria para su pila y sus estructuras de trabajo, sobre todo con baldosas pequeñas.

Por eso, 12 hilos son un buen punto de referencia para este hardware. No es un límite universal, sino lo que muestran nuestras mediciones: subir el pool hasta 64 hilos no dio una mejora proporcional.

#### Fracción secuencial y Ley de Amdahl

La ejecución con un solo hilo fue cerca de un 5 % más lenta que el monolito. Por otro lado, comparando 1 y 2 hilos, Karp-Flatt estima una fracción serial de $e = 0.064$.

Las dos observaciones ubican la parte no paralelizable del sistema entre el 5 % y el 6 %. Ahí entran cosas como la coordinación de las particiones, la consolidación de resultados en el Master y parte de la entrada, salida y comunicación.

Según la Ley de Amdahl, si la fracción estrictamente secuencial es $s$, el speedup máximo teórico está limitado por:

$$
S_{\max} = \frac{1}{s}
$$

Con $s$ entre 0.05 y 0.06, el límite estimado queda entre 16.7× y 20×, suponiendo que el resto del trabajo se paralelice de forma ideal y no aparezcan costos adicionales.

En la práctica el rendimiento puede ser menor por la comunicación y la coordinación entre nodos. Al aumentar los workers, la consolidación centralizada puede convertirse en el cuello de botella principal. Por eso, agregar más máquinas deja de servir cuando la parte secuencial y los costos de coordinación dominan el tiempo total.

## 6. Bitácora de uso ético de IAG (Nivel 3)

**Herramienta:** Claude (modelo Claude Opus 5.5) como asistente de redacción. El equipo ejecutó el Master, los workers y el cliente, tomó las mediciones y validó los resultados.

**Prompts** 

1. "Tengo los siguientes datos en screenshots, y necesito que creemos una guia en la cual pueda tener un camino para poner estos datos en un archivo .md y crear una estructura para el informe."
2. "Tengo el siguiente código y necesito que le hagas las modificaciones pertinentes para que otros dispositivos se puedan conectar, y que los workers puedan trabajar en esta red."
3. "Dame un plan de trabajo organizado donde yo pueda hacer lo siguiente: con el código que tengo, qué tengo que cambiar y cómo lo tendría que hacer, y dame tips de cuáles comandos tengo que ejecutar en cada computador para llevar a cabo las pruebas."

**Justificación crítica del código y del análisis incorporado**

- **Código.** Los cambios se concentraron en `WorkerMain`, para que los workers corrieran en otros dispositivos y se conectaran al Master por la red. El algoritmo de cómputo, las estrategias de partición y el cliente no se tocaron. El equipo revisó los cambios y comprobó que funcionaran con la validación `VERIFICACION OK` de cada corrida.
- **Plan de trabajo y comandos.** Los comandos sugeridos por la IA se ejecutaron a mano en cada computador, y el equipo decidió cuándo lanzar cada corrida.
- **Datos y gráficas.** La IA transcribió los tiempos de las capturas de consola, calculó S_rel, Karp-Flatt y el tamaño de la matriz a 16K/32K, y generó las gráficas de granularidad y de hilos. El equipo contrastó los valores con las capturas.
