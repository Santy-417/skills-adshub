# API de análisis del AdsHub — esquemas verificados

Todo lo de aquí está copiado del código de `BackendMatrixBeuty`, rama
`feature/ads-tablero`, al 20 de septiembre de 2026. Fuentes:
`app/api/v1/ads_analitica.py`, `app/services/ads_analitica.py`,
`app/models/ads_analitica.py`, `app/services/ads_tablero.py`, `docs/adshub.md`.

Si un campo no aparece en este archivo, **no existe**: no lo escribas en el
informe.

---

## Cómo entrar

Base: `{MATRIXBEAUTY_API}/ads/analitica/...`

Cabecera en **todas** las llamadas:

```
X-API-Key: {ADS_ANALITICA_API_KEY}
```

Nunca en la URL: las URLs quedan escritas en los logs del servidor y del proxy.

| Código | Significa |
|---|---|
| 401 | llave inválida o ausente |
| 503 | el servidor no tiene `ADS_ANALITICA_API_KEY` configurada: la API está apagada |
| 400 | rango invertido, o pasado del tope |
| 422 | el informe que mandaste no cuadra |

**Topes de rango:** 31 días donde entran creativos (`resumen`, `creativos`,
`alertas`, `campanas/{id}`); 366 días en `campanas` y `cobertura`.

---

## Cuatro reglas de lectura que no son negociables

**1. Las tasas vienen en FRACCIÓN.** `0.018` es 1,80 %. Vale para `ctr`,
`pedidos_por_clic`, `participacion_costo` y todas las `vista_*`.

**2. El `roi` es un múltiplo, no una tasa.** `2.4` es 2,4x. Por debajo de `1`
está perdiendo plata.

**3. `null` significa "no hay denominador", nunca cero.** Las razones son razón
de sumas y vienen en `null` cuando no hay con qué dividir. Un `roi: null` es "no
se puede decir", no "ROI de 0".

**4. El resumen diario y el detalle de creativos NO se suman.** Son dos reportes
distintos de TikTok, sin conciliar. Las cifras de `cuenta` salen del resumen
diario; las de `campanas` y `creativos`, del detalle. Sumarlos da un número que
no existe en ningún lado.

---

## `GET /contexto`

Sin parámetros. Se pide **primero**, para saber qué rango tiene sentido.

```jsonc
{
  "hoy": "2026-09-20",             // en hora de Colombia
  "primer_dia": "2026-08-01",
  "ultimo_dia": "2026-09-19",      // null si no hay nada cargado
  "dias_cargados": 45,
  "dia_vencido": "2026-09-19",     // siempre ayer, hora de Colombia
  "falta_el_dia_vencido": false,   // true = falta cargar ayer y ya pasó la hora límite
  "hora_limite": "10:00:00"
}
```

Antes de la hora límite `falta_el_dia_vencido` nunca es `true`: no se reclama
nada todavía.

---

## `GET /resumen?desde&hasta[&roi_minimo&costo_minimo]`

**La llamada con la que empieza el análisis.** Todo sale del mismo momento.

`roi_minimo` por defecto `1.0` (0 a 100). `costo_minimo` por defecto `20.0`.

```jsonc
{
  "desde": "2026-09-13", "hasta": "2026-09-19",
  "periodo_anterior": { "desde": "2026-09-06", "hasta": "2026-09-12" },

  "cuenta": {                      // del resumen diario (Campaign overview)
    "desde": "...", "hasta": "...",
    "dias_esperados": 7,
    "dias_cargados": 6,            // si no coinciden, falta un día: DILO
    "totales": { "dias": 6, "costo": 0.0, "pedidos": 0,
                 "ingreso_bruto": 0.0, "costo_por_pedido": null, "roi": null },
    "dias": [ { "fecha": "2026-09-19", "costo": 0.0, "pedidos": 0,
                "ingreso_bruto": 0.0, "roi": null } ]
  },
  "cuenta_anterior": { "dias_cargados": 7, "totales": { /* igual */ } },
  "comparacion_cuenta": {
    "costo":            { "actual": 0.0, "anterior": 0.0, "cambio": 0.0, "cambio_pct": null },
    "pedidos":          { /* igual */ },
    "ingreso_bruto":    { /* igual */ },
    "roi":              { /* igual */ },
    "costo_por_pedido": { /* igual */ }
  },

  "campanas": [ /* ver abajo */ ],
  "total_campanas": 0,             // cuántas hay; `campanas` trae hasta 15
  "totales_campanas": { /* métricas, del detalle de creativos */ },
  "comparacion_campanas": { /* mismos 5 campos que comparacion_cuenta */ },

  "creativos": [ /* los 15 que más pesan por costo; ver abajo */ ],
  "cobertura": { /* ver abajo */ },
  "alertas": [ /* ver abajo */ ],
  "umbrales": { "roi_minimo": 1.0, "costo_minimo": 20.0 },
  "monedas": ["USD"]
}
```

### El bloque de comparación

Cada campo comparado tiene siempre esta forma:

```jsonc
{ "actual": 300.0, "anterior": 40.0, "cambio": 260.0, "cambio_pct": 6.5 }
```

- `cambio` es la diferencia absoluta.
- `cambio_pct` es **fracción**: `6.5` es +650 %, `-0.32` es −32 %.
- `cambio_pct: null` significa que **el anterior estaba en cero**. Dividir daría
  "infinito por ciento". Dilo con palabras: "antes no había gasto".

> **El período anterior es el mismo número de días justo antes**, no "la semana
> pasada" del calendario. Del 15 al 21 se compara contra el 8 al 14. Así vale
> igual para 3 días que para 45.

### Cada campaña

```jsonc
{
  "campana_id": "...", "campana_nombre": "...",
  "costo": 0.0, "pedidos": 0, "ingreso_bruto": 0.0,
  "impresiones": 0, "clics": 0,
  "roi": null, "costo_por_pedido": null,
  "ctr": null,                 // FRACCIÓN
  "pedidos_por_clic": null,    // pedidos ÷ clics. NO es la "Ad conversion rate" de TikTok
  "cpm": null,                 // costo por mil impresiones
  "dias_con_datos": 0,
  "dias_parciales": 0,         // días cuya carga estaba a medias
  "dias_sin_confirmar": 0,
  "ultimo_dia": "...",
  "participacion_costo": null, // FRACCIÓN de la inversión del rango
  "comparacion": { /* costo, ingreso_bruto, pedidos, roi, costo_por_pedido */ },
  "es_nueva": false            // no existía en el período anterior
}
```

`es_nueva: true` con gasto alto es justo lo que hay que mirar: plata que
apareció de la nada.

### Cada creativo

Agrupado **por creativo, no por campaña**: el mismo video corre en varias y para
decidir qué repetir importa el video.

```jsonc
{
  "clave": "...",              // video ID, o tipo + producto si no hay video
  "tipo_creativo": "...", "video_id": "...", "video_titulo": "...",
  "cuenta_tiktok": "...", "producto_id": "...", "productos": 1,
  "campanas": 3,               // en cuántas campañas apareció
  "publicado_en": "...", "dias": 7,
  "costo": 0.0, "pedidos": 0, "ingreso_bruto": 0.0,
  "impresiones": 0, "clics": 0,
  "roi": null, "costo_por_pedido": null,
  "ctr": null, "pedidos_por_clic": null, "cpm": null,
  "retencion": {
    "impresiones_video": 0,
    "vista_2s": null, "vista_6s": null, "vista_25": null,
    "vista_50": null, "vista_75": null, "vista_100": null
  }
}
```

La retención son tasas **ponderadas por impresiones**, solo sobre videos, en
fracción. `vista_2s: 0.5` es "la mitad llegó a los 2 segundos".

### La cobertura

**Se lee antes que cualquier total.**

```jsonc
{
  "desde": "...", "hasta": "...",
  "dias": 7,
  "dias_sin_resumen": 0,
  "dias_sin_creativos": 1,
  "dias_parciales": 1,
  "dias_sin_confirmar": 0,
  "dias_completos": 5,
  "faltantes": [
    { "fecha": "2026-09-17", "resumen_diario": true, "creativos": "parcial" }
  ]
}
```

Qué significa cada condición de `creativos`:

| Marca | Qué significa, en palabras |
|---|---|
| `completo` | el día está entero |
| `parcial` | **faltan horas de ese día**: el total queda por debajo de lo real |
| `no_confirmado` | podría estar completo, pero no hay cómo afirmarlo |
| `falta` | no hay creativos de ese día |

### Las alertas

Ya vienen **redactadas y ordenadas**: primero por gravedad (`alta`, `media`,
`aviso`), luego por la plata en juego. Cópialas tal cual a `hallazgos`.

```jsonc
{
  "gravedad": "alta",
  "tipo": "campana_sin_pedidos",
  "titulo": "Campaña X: $300.00 sin un solo pedido",
  "detalle": "En 3 día(s) con datos gastó $300.00 y no registró pedidos.",
  "campana_id": "..."    // los campos sueltos cambian según el tipo
}
```

Los ocho tipos que puede devolver:

| `tipo` | Gravedad | Campos sueltos |
|---|---|---|
| `campana_sin_pedidos` | alta | `campana_id`, `costo` |
| `campana_roi_bajo` | alta si el ROI va por debajo de la mitad del umbral; si no, media | `campana_id`, `costo`, `roi` |
| `campana_gasto_sube_roi_baja` | media | `campana_id`, `costo` |
| `creativo_sin_pedidos` | alta | `clave`, `costo` |
| `creativos_sin_pedidos_resto` | alta | `creativos`, `costo` |
| `creativo_roi_bajo` | media | `clave`, `costo`, `roi` |
| `creativos_roi_bajo_resto` | media | `creativos`, `costo` |
| `datos_incompletos` | aviso | `dias_sin_resumen`, `dias_sin_creativos`, `dias_parciales` |

Dos cosas que conviene entender:

- **Los `_resto` no son un resumen perezoso.** Se nombran los 5 creativos que más
  pesan y el resto va en una línea con su total, porque veintidós alertas casi
  iguales de $20 tapan la campaña que se gastó $4.500.
- **`datos_incompletos` va siempre que falte algo** y cambia cómo se leen todas
  las demás cifras. Por eso va de último en la lista pero **primero en tu
  informe**.

---

## Los otros endpoints (solo si el resumen deja una pregunta abierta)

| Endpoint | Para qué | Tope |
|---|---|---|
| `GET /cobertura?desde&hasta` | solo la cobertura | 366 días |
| `GET /campanas?desde&hasta&limite` | todas las campañas (`limite` 1–200, 50 por defecto) | 366 días |
| `GET /campanas/{id}?desde&hasta` | una campaña día por día + sus 25 creativos | 31 días |
| `GET /creativos?desde&hasta&orden&limite` | ranking de toda la cuenta | 31 días |
| `GET /alertas?desde&hasta&roi_minimo&costo_minimo` | solo el gasto que no rinde | 31 días |

`orden`: `costo` (por defecto), `ingreso_bruto`, `pedidos` o `impresiones`.
`limite` en `/creativos`: 1–200, 25 por defecto.

`GET /campanas` y `GET /alertas` devuelven además `periodo_anterior`,
`totales_anteriores` y `comparacion`, igual que el resumen.
`GET /alertas` responde `{ "desde", "hasta", "alertas": [...], "total": n }`.

`GET /campanas/{id}` trae `totales`, `retencion`, `dias[]` (cada día con su
`condicion_corte`) más `creativos` y `creativos_total`. Sin datos en el rango
responde listas vacías, **no 404**.

---

## Guardar el informe

### `POST /informes[?reemplazar=true]` → **201**, o **409** si se repite

**Un informe por autor, período y día**, garantizado por un índice único en la
base (`ads_informes_uno_por_dia` sobre `autor, periodo_desde, periodo_hasta,
dia`), no por un `if`: es lo único que aguanta dos corridas simultáneas.

| Situación | Respuesta |
|---|---|
| primera vez | **201** con el informe guardado |
| repetido el mismo día | **409**, y el id del que ya existe en la cabecera `X-Informe-Id` |
| repetido con `?reemplazar=true` | **201**, sobrescribe y sigue habiendo uno solo |
| el mismo período **otro día** | **201**, informe nuevo a propósito |

Un 409 casi siempre significa que la rutina corrió dos veces. Lo normal es
detenerse y decirlo.

```jsonc
{
  "desde": "2026-09-13",          // requerido. Invertido = 422
  "hasta": "2026-09-19",
  "titulo": "...",                // 1 a 200
  "resumen": "...",               // 1 a 2000. En la lista solo se ven 2 líneas
  "cuerpo": "...",                // 1 a 60.000, Markdown
  "hallazgos": [ /* hasta 50 */ ],
  "datos": { /* el JSON de /resumen */ },
  "modelo": "..."                 // hasta 80
}
```

Cada hallazgo exige `gravedad` (`alta` | `media` | `aviso`) y `titulo` (1 a 200).
`tipo` hasta 60, `detalle` hasta 2000. **Los campos sueltos se aceptan y se
guardan**: el modelo permite extras a propósito.

**Los títulos de las alertas ya vienen recortados** por el backend: 200
caracteres el título completo, 70 el nombre del video que va dentro (los de
TikTok llegan a 711). Contra datos reales quedan en unos 90. El corte es por
palabra entera y termina en `…`. Cópialos tal cual; no los recortes otra vez.

El `autor` lo pone el backend (`claude`): no lo mandes.

### `GET /informes?limite` y `GET /informes/{id}`

La lista (1–100, 20 por defecto) viene del más nuevo al más viejo y **no trae el
cuerpo**. El detalle sí, más `datos`.

Sirven para ver si el informe de hoy ya se guardó antes de escribir otro.

---

## Dos avisos operativos

**Los rangos terminan ayer.** El reporte es de día vencido: hoy nunca tiene
datos. Pedir hasta hoy trae un día en cero que hace ver una caída que no existe.

**Una sola tienda** (`beautyglo`), fijada en el backend.
