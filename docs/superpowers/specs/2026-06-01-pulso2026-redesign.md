# Pulso 2026 — Rediseño y correcciones matemáticas

**Fecha:** 2026-06-01  
**Archivo objetivo:** `index.html` (proyecto de un solo archivo)

---

## Resumen

Rediseño completo de la interfaz del modelo interactivo de segunda vuelta presidencial Colombia 2026, más corrección de cuatro errores matemáticos/de visualización identificados en el código. El objetivo es hacer el modelo más intuitivo para una audiencia colombiana general, sin perder la profundidad para los usuarios más analíticos.

---

## 1. Correcciones matemáticas y de código

### 1.1 Denominador de `pa`/`pc` excluye votos en blanco
**Problema:** `pa = abe / (abe + cep) * 100` — el denominador solo incluye votos de candidatos, no los votos en blanco provenientes de los bloques. Los porcentajes mostrados en los titulares están levemente inflados respecto a los "votos válidos totales" que incluirían blancos.

**Corrección:** Cambiar a `pa = abe / (abe + cep + blancos) * 100`. Esto requiere también recalcular `abeV`, `cepV` y `total` de forma consistente. El ganador sigue siendo quien tiene más votos (signo de `pa - pc` no cambia), pero los porcentajes mostrados coincidirán con lo que marcaría un acta oficial.

### 1.2 Código muerto: `ANCHOR` y `resolve()`
**Problema:** `const ANCHOR = modelFor(PRESETS.base.__resolved||resolve(PRESETS.base)).pa` — `PRESETS.base.__resolved` nunca se define, `ANCHOR` nunca se usa en ningún otro lugar. Además, `resolve()` omite silenciosamente los valores `blank` de los presets.

**Corrección:** Eliminar las líneas 305–306 completas (`ANCHOR` y `resolve`).

### 1.3 Etiqueta "50%" del histograma desplazada
**Problema:** El histograma cubre el rango 45–60% en 20 bins. Con `justify-content: space-between`, la etiqueta central queda a la mitad visual (52.5%), pero el label dice "50%". El 50% real corresponde al bin 6/7 (a ~33% desde la izquierda).

**Corrección:** Usar 5 etiquetas distribuidas correctamente: `45% | 47.5% | 50% | 55% | 60%+`. Posicionar con CSS grid o porcentajes fijos en lugar de `space-between`.

### 1.4 Factor maquinaria (simplificación documentada, no se cambia)
El ajuste de maquinaria aplica puntos porcentuales sobre un total fijo de votos — redistribuye votos entre candidatos sin agregar nuevos. Es una simplificación coherente internamente. Se documenta en el footer como "efecto neto sobre el margen" en lugar de "votos adicionales".

---

## 2. Rediseño visual

### 2.1 Paleta de colores — Editorial Claro con tricolor Colombia
- **Fondo:** `#F5F3EF` (crema claro), cards `#FFFFFF`
- **Abelardo:** `#003893` (azul bandera Colombia)
- **Cepeda:** `#CE1126` (rojo bandera Colombia)
- **Acento Colombia:** `#FCD116` (amarillo bandera) — usado en badges y decoración
- **Texto:** `#1a1a1a` principal, `#555` secundario, `#aaa` muted
- **Franja tricolor:** barra de 6px en el tope y 4px en el footer (`#FCD116 50% | #003893 75% | #CE1126 100%`)
- Tipografías existentes (Fraunces + Libre Franklin + JetBrains Mono) se mantienen

### 2.2 Hero de candidatos con fotos
- Dos tarjetas side-by-side: foto circular del candidato + nombre + porcentaje proyectado + votos estimados
- Fotos desde Wikimedia Commons:
  - Abelardo: `https://upload.wikimedia.org/wikipedia/commons/thumb/6/69/Abelardo_de_la_Espriella_2025.jpg/200px-Abelardo_de_la_Espriella_2025.jpg`
  - Cepeda: `https://upload.wikimedia.org/wikipedia/commons/thumb/5/58/Perfil_Iv%C3%A1n_Cepeda_%28cropped%29.jpg/200px-Perfil_Iv%C3%A1n_Cepeda_%28cropped%29.jpg`
- Fallback `onerror="this.style.display='none'"` si la imagen no carga
- Abelardo a la izquierda (con más votos en primera vuelta), Cepeda a la derecha

### 2.3 Barra de votos y veredicto
- Barra horizontal con marca de 50% (línea punteada vertical)
- Labels: `← más Cepeda | 50% | más Abelardo →`
- Veredicto en prosa bajo la barra (se mantiene lógica actual)

---

## 3. Mejoras de UX

### 3.1 Banner de instrucciones (dismissible)
Caja azul-borde al tope de la página con 4 pasos numerados:
1. Mira el resultado base
2. Mueve las barras de "votos libres" para ver qué pasa con los votos de Paloma, Fajardo, etc.
3. Prueba los escenarios preestablecidos
4. Comparte tu escenario con el botón de compartir

Botón "Entendido, no volver a mostrar ✕" que oculta el banner y guarda en `localStorage`.

### 3.2 Sliders de votos libres simplificados
- **Sección principal:** solo el slider de reparto (← Cepeda / Abelardo →) por cada bloque. Un solo gesto = resultado inmediato.
- **Sección avanzada (colapsada):** abstención por bloque, voto en blanco por bloque, retención de bases, maquinaria. Accesible via "▾ Ajustes detallados".
- Los bloques arrancan colapsados en móvil (comportamiento actual se mantiene).

### 3.3 Terminología colombiana corregida
| Antes | Después | Razón |
|---|---|---|
| Balotaje | Segunda vuelta | Término estándar en medios colombianos (El Tiempo, Semana) |
| Votos huérfanos | Votos libres | Más natural; el subtítulo explica "de los candidatos que no pasaron" |
| Referendo anti-Tigre | Voto de castigo | Más reconocible para audiencia general |
| Colombia · Balotaje 21 jun | Colombia · Segunda vuelta · 21 de junio de 2026 | Fecha completa, término correcto |

Términos que **se mantienen** (estándar colombiano): `preconteo`, `escrutinio`, `abstención`, `votos en blanco`, `maquinaria`, `clientelismo`, `participación`, `censo electoral`, `fortines`.

---

## 4. Fuera de alcance

- No se modifica la lógica del factor maquinaria (simplificación coherente)
- No se agregan nuevas fuentes de datos ni departamentos
- No se cambia el modelo Monte Carlo (correcto)
- No se agrega backend ni base de datos
- La estructura de un solo archivo HTML se mantiene

---

## 5. Archivos modificados

| Archivo | Cambio |
|---|---|
| `index.html` | Todo lo anterior — único archivo del proyecto |
| `.gitignore` | Agregar `.superpowers/` |
