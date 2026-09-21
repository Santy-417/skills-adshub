# Ejemplos de respuesta

**De dónde salen.** Están armados a partir del código que construye cada
respuesta (`app/services/ads_analitica.py` y `app/services/ads_tablero.py`) y
contrastados con `tests/test_ads_analitica.py`, en la rama `feature/ads-tablero`
al 20 de septiembre de 2026.

| Archivo | Qué es | De dónde |
|---|---|---|
| `contexto.json` | `GET /ads/analitica/contexto` | **respuesta real** de dev, 20-09-2026 01:43 |
| `resumen.json` | `GET /ads/analitica/resumen?desde=2026-09-13&hasta=2026-09-19` | armado desde el código |
| `informe.json` | el cuerpo de `POST /ads/analitica/informes` escrito a partir de ese resumen | escrito a mano |

**`contexto.json` es una respuesta real** contra dev (solo fechas y conteos: no
hay nada sensible). Vale la pena mirarlo: el último día cargado es el **15** y el
día vencido es el **19** —faltan cuatro días— y `falta_el_dia_vencido` dice
`false`, porque a la 01:43 aún no había pasado la hora límite. Es exactamente la
trampa que la skill enseña a evitar.

**En los otros dos, las cifras son inventadas**, pero coherentes entre sí: los
totales cuadran con los días, los ROI con sus divisiones y las alertas con los
umbrales. Sirven para ver la **forma** de la respuesta y para practicar la
redacción. No son datos de Beauty Glo.

`resumen.json` viene recortado en dos listas, marcado con un comentario: el real
trae hasta 15 campañas y 15 creativos. Todo lo demás está completo.

El ejercicio útil: leer `resumen.json`, escribir el informe, y comparar contra
`informe.json`.
