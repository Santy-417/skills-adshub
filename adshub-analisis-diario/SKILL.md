---
name: adshub-analisis-diario
description: Escribe el análisis diario de la inversión en anuncios de Beauty Glo (AdsHub de MatrixBeauty) y lo deja guardado en AdsHub > Informes, en lenguaje que entiende cualquiera del equipo. Úsala cuando corra la rutina diaria de análisis de anuncios después de las 10:00 de Colombia, o cuando se pida el informe de anuncios, revisar el gasto en TikTok Ads, qué campañas o creativos están quemando plata, o qué videos están funcionando. Lee cifras ya calculadas por el backend y no recalcula ninguna.
---

# Análisis diario de anuncios — AdsHub

Corres sola, una vez al día, después de las 10:00 de Colombia. Lees lo que el
AdsHub ya tiene cargado y calculado, lo explicas en palabras simples y guardas
un informe. El equipo lo lee en **AdsHub > Informes**.

**Quién te lee: gente de marketing y de operación, no técnica.** No saben qué es
un endpoint ni les importa. Quieren tres cosas: si pueden creerle a los números
de hoy, dónde se está yendo la plata, y qué mirar. El informe se lee en dos
minutos o no se lee.

## La regla que manda sobre todas

**El backend calcula. Tú lees, interpretas y redactas.**

No sumes, no promedies, no saques porcentajes, no conviertas ratios. Cada cifra
que escribas se copia **tal cual** de la respuesta del API. Si el número que
quieres decir no está en la respuesta, no lo digas.

Por qué: el AdsHub calcula con `ads_tablero`, el mismo módulo que pinta el
tablero. Si tú sumaras por tu cuenta, el informe y el tablero dirían cosas
distintas del mismo día y nadie sabría a cuál creerle.

Lo único que agregas es el idioma: convertir `cambio_pct: -0.32` en "el gasto
bajó 32%" es traducir, no calcular. Inventarte un promedio semanal, no.

## Antes de empezar

Dos datos del entorno:

| Variable | Qué es |
|---|---|
| `MATRIXBEAUTY_API` | base de la API. Producción: `https://beautyhub.2becommerce.com/api/v1` |
| `ADS_ANALITICA_API_KEY` | la llave, en la cabecera `X-API-Key` de **cada** llamada |

Si tu entorno las trae como `ADSHUB_API_URL` y `ADSHUB_API_KEY`, úsalas: son las
mismas dos cosas con otro nombre. Si falta cualquiera de las dos, **detente y
dilo**. No escribas un informe a medias.

La llave va en la cabecera, **nunca en la URL**: las URLs quedan escritas en los
logs del servidor y del proxy.

## El recorrido

```bash
H="X-API-Key: $ADS_ANALITICA_API_KEY"

# 1. Hasta que dia hay datos
curl -s -H "$H" "$MATRIXBEAUTY_API/ads/analitica/contexto"

# 2. Todo lo demas, en UNA llamada
curl -s -H "$H" "$MATRIXBEAUTY_API/ads/analitica/resumen?desde=AAAA-MM-DD&hasta=AAAA-MM-DD"
```

### Paso 1 — `contexto`, para saber qué rango pedir

Te devuelve `ultimo_dia`, `dia_vencido`, `falta_el_dia_vencido` y `hora_limite`.

**El reporte es de día vencido: hoy nunca tiene datos.** Pedir hasta hoy trae un
día en cero y hace ver una caída que no existe.

- `hasta` = `ultimo_dia`. Nunca `hoy`.
- `desde` = `ultimo_dia` menos 6 días → una ventana de **7 días**.
- Si `falta_el_dia_vencido` es `true`, el informe **abre diciéndolo**: falta
  cargar el día de ayer y las cifras no lo incluyen.

> **Cuidado con `falta_el_dia_vencido: false`.** No quiere decir "está todo
> cargado": quiere decir "todavía no es hora de reclamarlo". La bandera es
> `vencido AND no existe el día`, así que antes de la `hora_limite` siempre es
> `false`, por atrasada que esté la carga.
>
> **Compara siempre `ultimo_dia` contra `dia_vencido`.** Si hay días de
> diferencia, el atraso va en la primera línea del informe con su número: "el
> último día cargado es el 15 y ayer fue el 19: faltan cuatro días". Esto pasa de
> verdad — así estaba dev el 20 de septiembre.

Por qué 7 días y no solo ayer: un día suelto se mueve por cualquier cosa y el
equipo terminaría leyendo sustos que se desinflan solos. Con 7 días, el API
compara automáticamente contra los 7 anteriores, y el día de ayer viene igual
dentro de `cuenta.dias` para nombrarlo aparte. Si alguien pide explícitamente un
día o un mes, cambia el rango; el tope donde entran creativos son **31 días**.

### Paso 2 — `resumen`, que trae todo de un solo momento

Una sola llamada porque todas las piezas salen del mismo instante. Pedirlas por
separado arriesga mezclar rangos entre llamadas y que el informe se contradiga
a sí mismo.

Trae:

| Llave | Qué es |
|---|---|
| `cuenta` | totales de la cuenta y `dias[]` día por día |
| `comparacion_cuenta` | cada cifra contra el período anterior de igual largo |
| `campanas` | cada campaña con su `comparacion` y `es_nueva` |
| `creativos` | los 15 que más pesan, con retención de video |
| `cobertura` | qué días están enteros, a medias o sin cargar |
| `alertas` | el gasto que no rinde, **ya redactado y ordenado** |
| `umbrales` | con qué `roi_minimo` y `costo_minimo` se marcaron las alertas |

Solo si algo del resumen te deja una pregunta abierta, profundiza con
`/campanas/{id}`, `/creativos` o `/alertas`. La mayoría de los días no hace falta.

El detalle de cada respuesta está en `references/api-adshub.md`.

## Cómo leer las cifras sin equivocarte

Aquí es donde se arruina un informe. Ante cualquier duda, `references/api-adshub.md`.
Lo esencial:

- **La cobertura va antes que cualquier total.** Un rango con días sin cargar da
  totales por debajo de lo real, y eso **no se nota mirando el total**. Si falta
  algo, va en la primera línea.
- **ROI es un múltiplo, no un porcentaje.** `2.4` es 2,4x: por cada dólar
  invertido volvieron 2,40. Por debajo de **1** está perdiendo plata.
- **`null` es "no se puede decir", nunca cero.** Un `roi: null` significa que no
  hubo con qué dividir. Escribir "ROI de 0" es una mentira.
- **`cambio_pct: null`** = el período anterior estaba en cero. Dilo con palabras
  ("antes no había gasto"), nunca "subió infinito" ni "subió 100%".
- **Las tasas vienen en fracción**: `ctr: 0.018` es **1,8%**. Vale para `ctr`,
  `pedidos_por_clic`, `participacion_costo` y toda la retención (`vista_2s` a
  `vista_100`).
- **`pedidos_por_clic` no es la "conversión" de TikTok.** Es pedidos ÷ clics. Si
  lo nombras, dilo así; no lo llames tasa de conversión.
- **El resumen diario y el detalle de creativos no se suman.** Son dos reportes
  distintos de TikTok, sin conciliar. Compáralos si aporta; sumarlos da un número
  que no existe en ningún lado.
- **Revisa `monedas`.** Si trae más de una, no mezcles montos: dilo.
- **No cruces los anuncios con los lives.** Decisión del equipo, 2026-09-20.

## Qué escribir

Cómo se redacta para este equipo, con ejemplos de antes y después y las reglas
del Markdown que la pantalla sí entiende, está en `references/como-escribir.md`.
**Léelo antes de redactar.** Lo esencial:

Cinco bloques, en este orden, y ninguno se alarga por rellenar:

1. **Si falta algo por cargar** — solo si falta. Una línea, arriba de todo.
2. **Qué pasó** — inversión, ingreso, ROI y pedidos del período, con el cambio
   contra el período anterior. Nombra el día de ayer aparte.
3. **Qué está quemando plata** — de `alertas`. Campaña o creativo, cuánto costó,
   qué devolvió.
4. **Qué está funcionando** — lo que conviene repetir, con la cifra que lo
   sostiene.
5. **Qué mirar** — dos o tres cosas concretas, cada una con su número. No más.

Reglas de fondo:

- **Ninguna afirmación sin cifra detrás.** Si el dato no alcanza, dilo: "con 2
  días cargados no se puede concluir".
- **Las alertas son señales, no veredictos.** Di lo que costó y lo que devolvió;
  la decisión de pausar o no es del equipo, no tuya.
- **Nada de jerga.** Ni "endpoint", ni "payload", ni "cobertura parcial del
  dataset". Se dice "faltan horas del martes".
- Si el período no tiene nada raro, el informe es corto y lo dice. Un informe que
  inventa hallazgos para parecer útil entrena al equipo a no leerlo.

## Guardarlo

```bash
curl -s -X POST -H "$H" -H "Content-Type: application/json" \
  "$MATRIXBEAUTY_API/ads/analitica/informes" -d @informe.json
```

**No hace falta mirar antes si ya existe: lo impide la base.** Hay un índice
único por `(autor, período, día)`. Si la corrida se repite, responde **409** con
el id del que ya está guardado en la cabecera `X-Informe-Id`.

Un 409 casi siempre significa que la rutina corrió dos veces. **Lo normal es
detenerse y decirlo**, no insistir. Solo si de verdad quieres reescribirlo:

```bash
curl -s -X POST -H "$H" -H "Content-Type: application/json" \
  "$MATRIXBEAUTY_API/ads/analitica/informes?reemplazar=true" -d @informe.json
```

El mismo período **otro día** sí entra como informe nuevo, a propósito: las
cargas posteriores cambian las cifras del mismo rango, así que esa segunda
lectura no es un duplicado.

El cuerpo, con sus topes (ejemplo completo en `references/ejemplos/informe.json`):

| Campo | Regla |
|---|---|
| `desde` / `hasta` | el mismo rango que pediste. Invertido = 422 |
| `titulo` | 1 a 200 caracteres. Que diga el período |
| `resumen` | 1 a 2000. **En la lista solo se ven 2 líneas**: lo esencial primero |
| `cuerpo` | 1 a 60.000, en Markdown |
| `hallazgos` | hasta 50. `gravedad` es `alta`, `media` o `aviso` |
| `datos` | el JSON de `/resumen` con que escribiste |
| `modelo` | tu modelo. Se muestra junto a la fecha |

**`hallazgos`: copia las `alertas` del resumen tal cual.** Ya vienen con
`gravedad`, `tipo`, `titulo` y `detalle` redactados y ordenadas por la plata en
juego, y traen campos sueltos (`campana_id`, `clave`, `costo`, `roi`) que el
modelo acepta y conviene conservar. No las reescribas ni las reordenes: si una te
parece mal redactada, arréglalo en el `cuerpo`, no en el hallazgo.

**`datos` no es opcional en la práctica.** Guarda ahí el `/resumen` completo. Las
cargas posteriores cambian los números del mismo rango, y sin esa foto un informe
de hace un mes no se puede auditar.

Responde **201** con el informe guardado.

## Cuando algo sale mal

| Respuesta | Qué pasó | Qué haces |
|---|---|---|
| **401** | la llave está mal o falta | detente y dilo. No reintentes |
| **503** | el servidor no tiene la llave configurada | detente y dilo. Es del backend |
| **400** | rango invertido, o pasado del tope (31 días con creativos, 366 sin) | corrige el rango |
| **409** | ya hay un informe de ese período escrito hoy | la rutina corrió dos veces. Detente y dilo; el id está en `X-Informe-Id` |
| **422** | algo del informe no cuadra (texto vacío, rango invertido) | léelo, corrígelo. **No reintentes igual** |
| **404** | ese informe no existe | revisa el id |

Si `/contexto` dice que no hay datos (`ultimo_dia` en `null`), no hay nada que
analizar: dilo y termina. **No guardes un informe vacío.**

## Lo que esta llave NO puede

Subir archivos, borrar cargas, tocar lives, personas o nómina. Solo lee el
análisis y guarda informes. Si necesitas algo más, pídeselo a una persona.
