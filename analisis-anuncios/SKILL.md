---
name: analisis-anuncios
description: Analiza la inversión en anuncios de Beauty Glo (AdsHub de MatrixBeauty) y deja el informe guardado en la plataforma. Úsala cuando se pida revisar los anuncios, el gasto en TikTok Ads, las campañas o los creativos, o cuando corra la tarea programada de análisis de anuncios. Consulta la API de análisis con la llave del entorno; no inventa cifras ni cruza con los lives.
---

# Análisis de anuncios — AdsHub

Lee lo que ya está cargado en el AdsHub, escribe un análisis y lo guarda. El
informe queda visible para el equipo en **AdsHub > Informes**.

## Antes de empezar

Dos datos, del entorno:

- `MATRIXBEAUTY_API` — base de la API. En producción:
  `https://beautyhub.2becommerce.com/api/v1`
- `ADS_ANALITICA_API_KEY` — la llave. Va en la cabecera `X-API-Key` de **cada**
  llamada, nunca en la URL (las URLs quedan escritas en los logs del servidor).

Si falta la llave, las respuestas son 401; si el servidor no la tiene
configurada, 503. En los dos casos: detente y dilo, no sigas con datos a medias.

## Qué responde esta API

Tres preguntas, que son las que le importan al equipo:

1. **qué está quemando plata** — campañas y creativos con gasto y poco o ningún
   retorno;
2. **qué cambió contra el período anterior** — el mismo número de días, justo
   antes;
3. **qué videos funcionan** — retención del 2s al 100%, CTR y conversión.

**No cruces los anuncios con los lives.** Es una decisión del equipo (2026-09-20).
Tampoco sumes el resumen diario con el detalle de creativos: son dos reportes
distintos de TikTok, no conciliados. Compáralos si quieres, nunca los sumes.

## El recorrido

```bash
H="X-API-Key: $ADS_ANALITICA_API_KEY"

# 1. Hasta qué día hay datos (el reporte es de día vencido: nunca incluye hoy)
curl -s -H "$H" "$MATRIXBEAUTY_API/ads/analitica/contexto"

# 2. Todo lo necesario para el análisis, de un solo momento
curl -s -H "$H" "$MATRIXBEAUTY_API/ads/analitica/resumen?desde=2026-09-14&hasta=2026-09-20"
```

`/resumen` trae cifras de la cuenta, campañas con su comparación, los creativos
que más pesan, la cobertura y las alertas. **Empieza siempre por ahí**: pedir las
piezas por separado arriesga mezclar rangos entre llamadas.

Para profundizar, solo si algo lo pide:

| Endpoint | Para qué |
|---|---|
| `GET /ads/analitica/campanas?desde&hasta&limite` | todas las campañas, hasta 366 días |
| `GET /ads/analitica/campanas/{id}?desde&hasta` | una campaña día por día, con sus creativos |
| `GET /ads/analitica/creativos?desde&hasta&orden&limite` | ranking de creativos de toda la cuenta |
| `GET /ads/analitica/alertas?desde&hasta&roi_minimo&costo_minimo` | solo el gasto que no rinde |
| `GET /ads/analitica/cobertura?desde&hasta` | qué días están enteros, a medias o sin cargar |

Rangos: 31 días como máximo donde entran creativos; 366 en `campanas` y
`cobertura`.

## Cómo leer las cifras

- **Mira la cobertura antes que cualquier total.** Un rango con días sin cargar
  o parciales da totales por debajo de lo real, y eso no se nota mirando el
  total. Si falta algo, dilo en la primera línea del informe.
- **ROI es un múltiplo**: 1 significa que recuperó lo invertido. Por debajo de 1
  está perdiendo.
- **`cambio_pct` en `null`** quiere decir que el período anterior estaba en cero:
  no hay porcentaje que calcular. Dilo con palabras ("no había gasto antes"), no
  escribas "infinito".
- **Las tasas son razón de sumas**, no promedios de promedios. No las vuelvas a
  promediar.
- `ctr`, `pedidos_por_clic` y la retención vienen como fracción: 0.018 es 1,8%.
  Ojo con el nombre: es **pedidos ÷ clics**, calculado aquí. No lo llames "tasa
  de conversión": TikTok usa ese nombre para otra cosa y su definición no está
  verificada.
- Los montos están en la moneda que diga `monedas`. Si trae más de una, no las
  sumes: dilo.

## Qué escribir

El cuerpo lo pinta un renderizador propio y mínimo, no una librería de Markdown.
Usa **títulos de `#` a `####`, listas, negrita, cursiva, código, citas, tablas y
líneas horizontales**. Los enlaces se ven como texto plano, así que escribe la
ruta en palabras ("AdsHub > Cargas") en vez de un enlace.

Un informe corto y accionable, en Markdown:

- **Primero lo que falta cargar**, si falta algo.
- **Qué pasó en el período**: inversión, ingreso, ROI y pedidos, con el cambio
  contra el período anterior.
- **Qué está quemando plata**: campaña o creativo, cuánto costó, qué devolvió.
- **Qué está funcionando**: lo que conviene repetir, con el dato que lo sostiene.
- Nada de recomendaciones sin cifra detrás. Si el dato no alcanza para afirmar
  algo, dilo: "con 2 días cargados no se puede concluir".

## Guardarlo

```bash
curl -s -X POST -H "$H" -H "Content-Type: application/json" \
  "$MATRIXBEAUTY_API/ads/analitica/informes" \
  -d '{
    "desde": "2026-09-14",
    "hasta": "2026-09-20",
    "titulo": "Anuncios del 14 al 20 de septiembre",
    "resumen": "Dos o tres frases con lo esencial.",
    "cuerpo": "## Qué pasó\n\n...",
    "hallazgos": [
      {"gravedad": "alta", "tipo": "campana_sin_pedidos",
       "titulo": "Campaña X: $300 sin un solo pedido",
       "detalle": "En 3 días con datos gastó $300 y no registró pedidos.",
       "campana_id": "123"}
    ],
    "datos": { "...": "el JSON de /resumen" }
  }'
```

- **Las alertas de `/resumen` ya vienen redactadas** con `gravedad`, `tipo`,
  `titulo` y `detalle`: cópialas tal cual a `hallazgos`. Los títulos ya vienen
  recortados a 200 caracteres, que es lo que acepta el campo; los de TikTok
  llegan a 711.
- `gravedad`: `alta`, `media` o `aviso`.
- **`datos` no es opcional en la práctica**: guarda ahí el `/resumen` con que
  escribiste. Las cargas posteriores cambian los números del mismo rango, y sin
  esa foto el informe no se puede auditar después.
- Responde 201 con el informe guardado. Un 422 significa que algo del cuerpo no
  cuadra (rango invertido, texto vacío): léelo y corrígelo, no reintentes igual.
- **Un 409 significa que ese período ya se analizó hoy.** No es un error tuyo:
  la corrida se repitió. El id del informe que ya existe viene en la cabecera
  `X-Informe-Id`. Si de verdad quieres reescribirlo, repite con
  `?reemplazar=true`; si no, no insistas. El mismo período **otro día** sí entra
  como informe nuevo, porque las cargas posteriores cambian las cifras.

## Lo que esta llave NO puede

Subir archivos, borrar cargas, tocar lives, personas o nómina. Solo lee el
análisis y guarda informes. Si necesitas algo más, pídeselo a una persona.
