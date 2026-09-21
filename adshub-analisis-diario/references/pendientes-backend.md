# Pendientes — lo que falta antes de que la rutina corra sola

Verificado contra `BackendMatrixBeuty` (`2460dc7`) y `FrontendMatrixBeuty`
(`28ea055` en `origin/feature/ads-tablero`), más una llamada real a la API
contra dev, el 20 de septiembre de 2026.

---

## Ya resuelto

| Qué era | Cómo quedó |
|---|---|
| La API de análisis no existía | completa y verificada: `/contexto`, `/resumen`, `/campanas`, `/creativos`, `/alertas`, `/cobertura`, `/informes` |
| No había forma de autenticar una tarea programada | llave `X-API-Key` (`app/core/llave_api.py`) |
| No había dónde guardar el informe ni pantalla que lo mostrara | `ads_informes` + **AdsHub > Informes** |
| El backend no calculaba las comparaciones | las calcula, junto con las alertas ya redactadas |
| Se podían guardar dos informes del mismo período | índice único `(autor, periodo_desde, periodo_hasta, dia)`; repetir da **409** con `X-Informe-Id`, y `?reemplazar=true` sobrescribe |
| La skill del repo decía `tasa_conversion` | corregido: es `pedidos_por_clic` |
| El renderizador no pintaba `####` ni líneas horizontales, y una fila de tabla sin barra final perdía una celda | los tres arreglados en `28ea055` |
| Los títulos de alerta llegaban a 288 caracteres | recortados por palabra entera: 200 el título, 70 el nombre del video |
| Las migraciones no estaban aplicadas en dev | **las cuatro del AdsHub están corridas en dev**, incluida `20260920140000` |

**Probado contra la base real de dev:** repetir el mismo informe da 409 con el id
en la cabecera; con `?reemplazar=true` da 201 y sigue habiendo uno solo.

---

## 1. Bloqueante para producción: allí no hay nada desplegado

En producción **no hay ninguna** de las migraciones del AdsHub. El hub completo
sigue sin desplegarse.

Mientras siga así, la rutina solo puede apuntar a dev. Orden de despliegue en
cada entorno: migraciones → backend → frontend.

## 2. Bloqueante: la rutina en la nube necesita sus dos variables

| Variable | Dónde está | Qué falta |
|---|---|---|
| `ADS_ANALITICA_API_KEY` | en el `.env` del backend, lado servidor (64 caracteres) | ponerla en el entorno de la rutina |
| `MATRIXBEAUTY_API` | en ningún `.env` de esta máquina | definirla |

Dos decisiones:

- **Contra qué base apunta la rutina.** Hoy solo puede ser dev (ver punto 1). El
  `.env` local alterna entre los dos proyectos de Supabase, así que la URL se
  fija a propósito, nunca se hereda.
- **Cómo se llaman las variables en la nube.** El encargo original las nombró
  `ADSHUB_API_URL` y `ADSHUB_API_KEY`; el código usa `MATRIXBEAUTY_API` y
  `ADS_ANALITICA_API_KEY`. La skill acepta los dos nombres, pero conviene
  quedarse con uno.

## 3. Decidir cada cuánto corre

Si es **diaria, después de la hora límite de carga, con los últimos 7 días**, la
skill ya está escrita para eso y no hay nada más que tocar.

La hora límite por defecto son las **10:00 de Colombia**, configurable desde la
pantalla (`GET /ads/config`). Si alguien la cambia, `/contexto` lo refleja, pero
**la hora a la que se dispara la rutina hay que cambiarla aparte**: son dos
relojes distintos.

## 4. Dónde vive la skill: resuelto

Vive en el repo `skills-adshub`, en `adshub-analisis-diario/`. Al subirla se
retiró `analisis-anuncios/`, la versión anterior: ya no hay dos skills que
alguien pueda confundir, y la vieja sigue recuperable en el historial de git.

Lo de hoy ya está aplicado en estos archivos: el 409 al repetir,
`?reemplazar=true`, los títulos recortados y los tres arreglos del renderizador.

---

## Un hallazgo de la primera llamada real

**`falta_el_dia_vencido: false` no quiere decir "está todo cargado".**

La respuesta real de dev, el 20 de septiembre a la 01:43:

```json
{ "hoy": "2026-09-20", "ultimo_dia": "2026-09-15",
  "dia_vencido": "2026-09-19", "falta_el_dia_vencido": false,
  "hora_limite": "10:00:00" }
```

El último día cargado es el **15** y el día vencido es el **19**: faltan cuatro
días, y la bandera dice `false`. No es un error: `estado_pendiente` la calcula
como `vencido AND no existe el día`, y a la 01:43 todavía no ha pasado la hora
límite, así que no se reclama nada.

La rutina corre después de las 10:00, cuando `vencido` ya es `true`. Aun así, la
bandera solo mira **el día de ayer**, nunca el atraso acumulado. **El atraso se
ve comparando `ultimo_dia` con `dia_vencido`**, y la skill ya lo hace.

## Estado del árbol de trabajo del frontend en esta máquina

No es un pendiente del backend, pero conviene saberlo antes de tocarlo:

- La rama local `feature/ads-tablero` está en `bd39768`; **`28ea055` solo está en
  `origin`**. Los arreglos del renderizador no están en el disco de esta máquina.
- Hay cambios sin confirmar, a mitad de una refactorización: `informes/page.tsx`
  **marcado para borrar** (en el índice), `tablero/page.tsx` y `config/hubs.ts`
  modificados, y `InformesAds.tsx` sin seguimiento.

Un `git pull` encima de eso no es inocuo. No se tocó nada.
