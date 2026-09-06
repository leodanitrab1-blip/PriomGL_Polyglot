# Changelog — PriomGL Polyglot Quantum

## v9 — Evaluado en vivo de nuevo: sobreexposición real + red neuronal en el tonemap

Con el motor ya estable (v8), volví a correrlo en el navegador real para
buscar por qué seguía "viéndose feo" y encontré una causa concreta y
medible, no una impresión.

### 🔆 Corregido: el sol estaba sobreexpuesto y aplastaba toda la textura

Medí los píxeles renderizados del suelo (desviación estándar de solo
2.7/255 — prácticamente un color plano) pese a que el terreno **sí** tenía
una textura de pasto procedural con variación real. La causa: la intensidad
del sol llegaba a 6.0 al mediodía, y con la fórmula PBR de este motor
(`albedo * sunColor * intensity * NdotL`) eso empujaba casi cualquier
superficie iluminada muy por encima del rango útil del tonemap antes de
llegar a él — todo el contraste de la textura quedaba comprimido en un
blanco lavado. Bajé el pico de intensidad a 3.2 y subí el contraste/detalle
de las texturas procedurales de pasto y roca (una octava de ruido fino
adicional). Medido después del cambio: desviación estándar ~5-6.5/255, más
del doble, confirmado con la misma cámara y el mismo tipo de escena.

### 🧠 Nuevo: tonemap aprendido con una red neuronal real (opcional)

Entrené un MLP diminuto (1→12→12→1, ~180 pesos) con Python/numpy puro
—sin frameworks, sin internet— para aproximar una curva de tonemap filmico,
y lo incrusté como función GLSL evaluada por píxel en los shaders de
terreno, PBR y agua. Ver `docs/NeuralTonemap.md` para el detalle honesto:
qué tan bien ajusta, cómo reentrenar, y por qué se evalúa una sola vez sobre
luminancia (no 3 veces por canal) para mantener el costo bajo.

**Está gateada por hardware**: al probarla "siempre activa" en un Chromium
real bajo renderizado por software, confirmé que el costo extra podía volver
a estancar el renderizado — exactamente el problema que costó varias rondas
arreglar. Por eso solo se activa en desktop y tiers `high`/`ultra`; en móvil
y tiers `medium`/`low`/`potato` (el caso real reportado) se usa la fórmula
ACES directa de la que la red fue entrenada, sin el costo extra.

### 🧪 Metodología

Todo lo anterior se verificó con el mismo arnés de Chromium headless de la
v8: server local, consola capturada, capturas de pantalla, y esta vez además
lectura directa de píxeles (`PIL`/`numpy`) para medir contraste en vez de
solo mirar las imágenes.


Instalé un Chromium headless y corrí el motor de verdad (WebGL2 real, consola
capturada) para dejar de adivinar. Esto encontró dos bugs concretos que las
rondas anteriores no habían visto:

### 🎯 La causa real del "se congela" — confirmada, no solo teorizada

```
ReferenceError: plant is not defined
    at WorldAI._behaviorGrazing (WorldAI.js:664)
```

En `_behaviorGrazing`, después de encontrar la planta más cercana con un
`for (const plant of this.plants)`, el código para "comer" hacía referencia
a `plant.size` / `plant.health` — pero `plant` es la variable del `for`, que
ya no existe fuera de él. Debía ser `nearestPlant` (la variable donde sí se
guarda el resultado de la búsqueda). Esto lanzaba una excepción **cada vez
que cualquier animal empezaba a pastar** — algo que pasa casi de inmediato
con 30-40 animales activos.

Como `worldAI.update()` corre **antes** de `renderer.render()` en el loop
principal, esta excepción cortaba el frame *antes* de llegar a dibujar nada.
El try/catch de la v7 evitó que el motor muriera del todo, pero sin él
arreglado la pantalla dejaba de actualizarse igual — visualmente
indistinguible de un congelamiento total, solo que ahora con el error
gritando en la consola en vez de morir en silencio. Arreglado: ahora usa
`nearestPlant` correctamente.

### 🌑 Sombra rota: un parche gigante y pixelado sobre el terreno

Con el motor corriendo de verdad pude ver el bug con mis propios ojos: una
sombra de árbol enorme, borrosa y con bordes en bloques cubriendo media
ladera — muchísimo más grande que el árbol que la proyecta.

Causa: mi propio cambio de la v5/v6, que rellenaba el arreglo de splits de
cascada de sombra de Python repitiendo el último valor cuando el tier tenía
menos de 4 (p. ej. `medium` da 3 splits → `[31.75, 76.09, 220]` se rellenaba
a `[31.75, 76.09, 220, 220]`). Dos cascadas con el mismo plano lejano acaban
siendo geométricamente idénticas — trabajo de GPU duplicado y, peor, la capa
de sombreado puede terminar usando la cascada equivocada (más gruesa/borrosa)
para geometría cercana. Arreglado: en vez de repetir el último valor, cada
cascada extra se extrapola creciendo un 40%, así ninguna cascada queda
duplicada.

Confirmé el arreglo con capturas de pantalla reales antes/después: el parche
gigante desapareció y las sombras de los árboles ahora se ven como manchas
suaves y proporcionadas sobre el terreno y el agua.

### 🏔️ Pendiente identificado (no arreglado aún): bordes dentados en riberas/acantilados

Con las pruebas reales también vi que donde el terreno cae con pendiente
pronunciada hacia el agua (riberas del río), el borde entre pasto y roca se
ve dentado/con "dientes de sierra", y en algunos ángulos la repisa de pasto
parece flotar sobre la pared de roca. Esto es una limitación de resolución
de la malla del terreno (192 segmentos sobre 400 unidades ≈ 2m por celda) en
pendientes pronunciadas, no una excepción ni un dato corrupto — es el
siguiente candidato claro para una pasada de pulido visual (más segmentos
cerca de la cámara, o una franja de "playa"/transición que suavice el
encuentro agua-tierra).

### 🧪 Metodología: dejar de adivinar

A partir de ahora, antes de reportar un arreglo como "hecho", lo ejecuto en
un Chromium real (headless, WebGL2 vía software rendering) sirviendo el
proyecto con `python3 -m http.server`, capturo la consola completa y tomo
capturas de pantalla a lo largo de ~45s de simulación. Así encontré ambos
bugs de esta ronda en minutos en vez de seguir leyendo código a ciegas.


Después de tu reporte de que seguía congelándose (esta vez a los ~5s),
investigué más a fondo y encontré el mecanismo exacto — no solo "muchos
draw calls", sino un fallo estructural que convierte *cualquier* error en un
congelamiento permanente.

### 🧯 Corregido: el loop principal no tenía manejo de errores

`PriomEngine.start()` ejecutaba toda la simulación y el render dentro de un
`requestAnimationFrame` **sin `try/catch`**. Si cualquier línea de ese
camino — simulación del mundo, sincronía de fauna, render — lanzaba una
excepción, esta se propagaba fuera del callback y `requestAnimationFrame(loop)`
**nunca se volvía a llamar**. El motor no se "ponía lento": se detenía en
seco, dejando el último frame dibujado congelado en pantalla para siempre,
mientras la pestaña seguía técnicamente viva. Esto es exactamente la
diferencia entre "se ve mal/lento" y "se congela".

Ahora el loop atrapa cualquier error, lo imprime en consola (para poder
diagnosticarlo) y sigue intentando el siguiente frame — un bug puntual deja
de ser fatal. Si algo falla de forma catastrófica y consecutiva durante ~4s
seguidos, el motor se detiene de forma controlada en vez de spamear errores
para siempre.

### 🐛 Corregido: colisión de IDs entre animales (causaba fauna con formas/posiciones mal)

`_spawnAnimal()` y `_spawnAnimalNear()` calculaban el id del nuevo animal
como `this.animals.length + 1`. Como el array se acorta cada vez que un
animal muere (`this.animals = this.animals.filter(...)`), dos animales
**distintos** nacidos en momentos distintos podían terminar con el
**mismo id**. `WildlifeRenderer` indexa sus mallas por id — con un id
duplicado, dos animales de especie/posición distintas terminaban
compartiendo una sola malla, que saltaba entre las posiciones y aparentaba
tener la forma equivocada. Ahora hay un contador monotónico
(`WorldAI._nextAnimalId`) que garantiza unicidad durante toda la sesión.

### 🧟 Corregido: fuga real de mallas "fantasma" al morir un animal

Cuando un animal moría (hambre, depredación, incendio) se quitaba de
`worldAI.animals`, pero su malla en `WildlifeRenderer.instances` **nunca se
eliminaba de la escena**. Con cada muerte, un cadáver invisible al gameplay
pero muy real para el GPU se quedaba dibujándose para siempre. En una sesión
larga con nacimientos y muertes constantes, los draw calls solo podían
crecer, nunca bajar — degradación progresiva hasta el congelamiento, tal
como reportaste. `sync()` ahora borra de la escena y del mapa cualquier
animal que ya no esté vivo, cada frame.

### 📄 Nota honesta sobre "usa lo mejor de cada lenguaje" para gráficos

Ya lo mencioné en la ronda anterior pero lo repito porque es importante:
PySide6 (Python de escritorio) y Kotlin nativo **no pueden ejecutarse dentro
de una pestaña de navegador**. El único pixel real que puede dibujar tu app
en `gl-polyglot.onrender.com` sale de WebGL2 vía JavaScript — por eso ahí
están todos los arreglos de esta ronda. Python/Kotlin siguen aportando datos
y política real (ver v5), y C++ sigue teniendo un camino real a WASM (ver
v5) para quien quiera compilarlo. Una vez confirmes que ya no se congela,
la siguiente ronda puede enfocarse 100% en pulir la parte visual (agua,
atmósfera, iluminación) ahora que el motor no se cae solo.


### 🥶 Corregido: la app se congelaba a los ~15 segundos

**Causa real:** la fauna (venados, lobos, osos, aves) se construía con **20–25
`Mesh` independientes por animal, sin instancing**, y cada uno se volvía a
dibujar hasta 4 veces más en el paso de sombras (una por cascada activa). Con
30-50 animales vivos a la vez, eso son miles de draw calls por frame — exactamente
lo que mostraba tu captura (`Draws: 2159`, `FPS: 11`). En un navegador móvil,
ese volumen de llamadas WebGL por frame ahoga el hilo principal hasta que el
tab queda "congelado" (no es un crash: es el motor pidiéndole al driver más
de lo que puede sostener, frame tras frame, hasta que deja de responder).

**Arreglo — `js/utils/Wildlife.js` reescrito:**
- Cuerpo + cuello + cabeza + hocico + orejas + cornamenta (partes que nunca se
  mueven entre sí, solo la "columna" entera se balancea) ahora se **hornean en
  una sola malla fusionada** (`GeometryMerger.mergeRigid`, nuevo — ver abajo)
  en vez de 5 a 11 draw calls por animal.
- Pierna inferior + pezuña (comparten el mismo pivote de rodilla, nunca se
  mueven por separado) también se fusionan en una sola malla.
- Resultado: un venado con cornamenta pasa de ~25 draw calls a **12**; un lobo
  u oso de ~19 a **10**; un ave de 9 a **4**. La cola y las alas siguen siendo
  mallas separadas porque sí se animan de verdad (menean/aletean cada frame).
- Nuevo `GeometryMerger.mergeRigid(parts)` en `PriomRenderer.js`: fusiona
  partes con posición + rotación (eje-ángulo) + escala completas horneadas en
  los vértices/normales — el `merge()` original solo soportaba un offset en Y.
- Sombra por distancia: cada animal deja de proyectar sombra pasados ~45m de
  la cámara (`Wildlife.js::sync`), otro recorte real de draw calls con costo
  visual nulo a esa distancia.
- Tope de fauna simultánea según el tier de hardware (14 en `potato`, hasta
  80 en `ultra`) en vez de un `maxAnimals: 80` fijo para cualquier GPU.

### 🐛 Corregido: el culling de vegetación nunca funcionó como debía

`PriomEngine._cullVegetation()` llamaba `forest.cull(camera, maxDist)`, pero
`ChunkedForest.cull(camera, dt)` espera un **delta de tiempo** en el segundo
parámetro, no una distancia — la distancia real vive en `forest.maxDist`
(otra propiedad, nunca actualizada). Esto rompía dos cosas a la vez: el
throttle interno de culling (pensado para correr cada 0.5s) se ejecutaba
completo **cada frame** porque `_lastUpdate` siempre superaba el intervalo de
golpe, y el ajuste de distancia de dibujado por calidad nunca se aplicaba de
verdad. Ahora se pasa el `dt` real del frame y se actualiza `forest.maxDist`
por separado.

### 🎨 Corregido: iluminación incorrecta en instancias con escala no uniforme

Los árboles varían su altura de forma independiente al ancho del tronco
(escala no uniforme). El vertex shader de instancias rotaba las normales sin
corregir por esa escala, lo que producía sombreado ligeramente incorrecto en
cada instancia — parte de la sensación de "las formas se ven mal" incluso
después de arreglar los picos. Ahora las normales se transforman con el
inverso de la escala antes de rotar (correcto para matrices de escala
diagonales), igual que se haría con la inversa-transpuesta completa.

### 🌿 Corregido: los "arbustos" eran una esfera con overdraw triple

`buildBushGeometryMerged` calculaba un ángulo (`ang`) para distribuir 3
esferas en círculo… y nunca lo usaba: las tres esferas quedaban exactamente
superpuestas en el mismo punto (una esfera dibujada 3 veces, no un arbusto).
Ahora se reparten realmente en un anillo con tamaños ligeramente distintos.

### 📉 Impacto esperado

Con fauna a tope (tier `ultra`, 80 animales) los draw calls de fauna bajan de
~1600-2000 (con sombras) a ~700-900; en tiers móviles (`low`/`potato`, tope de
14-22 animales + sombra recortada por distancia) la fauna deja de ser el
cuello de botella dominante. Combinado con el culling de vegetación real, el
motor debería sostener frames estables en vez de degradarse hasta congelarse.


### 🐛 Corregido: árboles/vegetación renderizados como picos gigantes

**Causa raíz:** `js/renderer/shaders.js` (`instancedVS` y `shadowInstancedVS`)
declaraba el buffer de instancia como una matriz 3×4 empaquetada en tres
`vec4` consecutivos (`iRow0/iRow1/iRow2`, locations 6-8). Pero
`js/ecs/InstancedEntities.js::setInstance()` nunca escribió ese formato: sube
posición (vec3) + cuaternión de rotación (vec4) + escala (vec3) + color tint
(vec3) + fase de viento (float) como **cinco atributos independientes**
(locations 6-10), y así los configura `vertexAttribPointer` en `build()`.

El shader leía esos bytes crudos como si fueran filas de una matriz de
rotación. El resultado no era una matriz ortonormal — era ruido geométrico:
por eso los árboles se veían como picos/triángulos gigantes desgarrados en
vez de conos de follaje sobre un tronco.

**Arreglo:** ambos vertex shaders ahora declaran los mismos 5 atributos que
la CPU realmente sube (`iPos`, `iQuat`, `iScale`, `iColor`, `iPhase`) y
reconstruyen la matriz de rotación con una función estándar
`quatToMat3(vec4 q)`, igual que se haría en cualquier motor con
instancing basado en cuaterniones.

Efecto colateral corregido de paso: el atributo "LOD" en location 11
(`InstancedEntities.js`) apuntaba a `offset == stride`, es decir leía
siempre los primeros 4 bytes de la **siguiente** instancia. Como ningún
shader lo consumía, era inofensivo pero incorrecto — se eliminó.

### ✨ Mejoras de geometría

- `Primitives.cylinder()` ahora genera tapas (caps) reales en la base y la
  punta cuando el radio > 0. Antes los troncos y las copas de los árboles
  eran cáscaras huecas — visibles como agujeros en ángulos bajos o al mirar
  desde abajo.

### 🐍 Python — de narrativa a datos reales en runtime

- `python/generate_hw_luts.py` genera ahora también: splits prácticos de
  cascada de sombra por tier, curva de ráfagas de viento (suma de senos
  inconmensurables), y distancias de LOD por tier.
- Se generó `data/hw_luts.json` (versión 5) con `python3 python/generate_hw_luts.py -o data/hw_luts.json`.
- `PolyglotBridge.loadPythonData()` hace `fetch('data/hw_luts.json')` al
  arrancar y **reemplaza** las tablas embebidas por las reales. Si falla
  (offline, `file://`, CORS), cae automáticamente al espejo JS — sin romper
  el motor.
- El viento del mundo ahora modula su intensidad con la curva de Python en
  vez de un `Math.sin()` suelto.
- Las distancias de cascada de sombra usadas por el renderer ahora vienen de
  esa tabla cuando está disponible.

### ⚙️ C++ — kernels compilables a WASM de verdad

- Nuevo `native/cpp/priom_kernels.cpp`: wrapper `extern "C"` sobre
  `PriomKernels.hpp` con exports listos para Emscripten
  (`fbm_noise2`, `ridged_noise2`, `frustum_cull_aabbs_ptr`,
  `integrate_particles_ptr`, un bump allocator simple).
- Nuevo `native/cpp/build_wasm.sh`: compila con `emcc` a
  `native/wasm/priom_kernels.wasm` si Emscripten está instalado.
- `PolyglotBridge.loadWasmKernels()` intenta cargar ese `.wasm`; si existe,
  el ruido de terreno se ejecuta en WASM real (`bridge.wasmLive === true`).
  Si no se compiló, el motor sigue funcionando con el port JS idéntico —
  cero cambios de comportamiento, solo de rendimiento.

### 🟣 Kotlin — guardia térmica real, no solo declarativa

- `QualityPolicy.thermalBias(sustainedOverBudgetSeconds, isMobile)`: nueva
  función que reduce la calidad de forma preventiva cuando el dispositivo
  lleva varios segundos por encima del presupuesto de frame, en vez de
  esperar a que el sistema operativo le baje el clock a la GPU por calor.
- Espejada 1:1 en `PolyglotBridge.js` (`KotlinPolicy.thermalBias`) y
  conectada al Council: `councilDecide()` ahora lleva un EMA de segundos
  "sobre presupuesto" y lo aplica como multiplicador extra sobre el voto
  fusionado.

### 📄 Documentación

- `docs/ARCHITECTURE.md` actualizado para reflejar que Python y C++ ya no
  son solo "código que documenta el diseño" — son datos/kernels reales que
  el motor carga en runtime, con fallback automático si no están presentes.
