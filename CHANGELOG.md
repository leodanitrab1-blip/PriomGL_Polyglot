# Changelog — PriomGL Polyglot Quantum

## v6 — Congelamiento, fauna con formas rotas y culling roto

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
