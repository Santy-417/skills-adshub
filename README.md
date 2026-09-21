# skills-adshub

Las skills que usa Claude para trabajar contra MatrixBeauty (Beautyglo Hub).
Hoy hay una: el análisis diario de anuncios del AdsHub.

Es un repo aparte, hermano de los otros dos, dentro de la misma carpeta:

```
MatrixHub/
  MatrixBeutyBack/     backend (FastAPI + Supabase)
  MatrixBeutyFront/    frontend (Next.js)
  skills-adshub/       este: las skills de Claude
```

**Por qué separado.** Una skill no se despliega con el backend ni con el
frontend: se instala donde corre Claude, que puede ser otra máquina. Mezclarla
con el código obligaría a desplegar la plataforma para corregir una frase del
instructivo, y a la inversa: un cambio de la skill ensuciaría el historial del
backend. Aparte, cada uno se mueve a su ritmo.

**Lo que NO cambia por estar separado:** la skill depende del API del backend.
Si cambia un endpoint o un campo, la skill queda desactualizada sin que nada
avise. Por eso cada skill dice contra qué versión del API se escribió y qué
endpoints usa, y el contrato vive documentado en el backend
(`MatrixBeutyBack/docs/adshub.md`).

## Skills

| Carpeta | Qué hace |
|---|---|
| `adshub-analisis-diario/` | Lee lo que el AdsHub ya calculó, lo explica en palabras simples y guarda un informe. El equipo lo lee en AdsHub > Informes |

### `adshub-analisis-diario/`

```
SKILL.md                        la skill
references/
  api-adshub.md                 esquemas verificados de /ads/analitica
  como-escribir.md              cómo se redacta + qué Markdown pinta la pantalla
  pendientes-backend.md         lo que falta decidir o confirmar
  ejemplos/
    README.md                   de dónde salen los ejemplos
    contexto.json               GET /contexto
    resumen.json                GET /resumen
    informe.json                POST /informes escrito desde ese resumen
```

Corre sola una vez al día, después de las 10:00 de Colombia, en una rutina de
Claude Code en la nube. Lee el paquete que el AdsHub ya calculó, lo interpreta y
guarda el informe.

**El backend calcula; la skill lee, interpreta y redacta.** Ninguna cifra del
informe se recalcula: todas se copian de la respuesta del API. El AdsHub calcula
con `ads_tablero`, el mismo módulo que pinta el tablero; si la skill sumara por
su cuenta, el informe y el tablero dirían cosas distintas del mismo día y nadie
sabría a cuál creerle.

Verificada contra `MatrixBeutyBack` en `2460dc7` y el renderizador de
`MatrixBeutyFront` en `28ea055` (20 de septiembre de 2026): el 409 al repetir un
informe, `?reemplazar=true`, los títulos recortados y los tres arreglos del
renderizador ya están aplicados.

Reemplazó a `analisis-anuncios/`, la versión anterior, retirada el 20 de
septiembre de 2026. Sigue disponible en el historial de git.

## Cómo se instala

Copiar la carpeta de la skill donde Claude las busca (por ejemplo
`~/.claude/skills/`), o apuntar la tarea programada a este repo clonado.

Las dos variables que necesita, en el entorno donde corre:

```
MATRIXBEAUTY_API=https://beautyhub.2becommerce.com/api/v1
ADS_ANALITICA_API_KEY=<la llave de la API de análisis>
```

La llave la genera un admin y vive en el `.env` del servidor
(`ADS_ANALITICA_API_KEY`). **Nunca se escribe en este repo.**

## Antes de ponerla a correr

Lee `adshub-analisis-diario/references/pendientes-backend.md`. Lo que falta hoy:

- **En producción no hay ninguna de las cuatro migraciones del AdsHub**; en dev
  están corridas las cuatro. Mientras siga así, la rutina solo puede apuntar a
  dev. Orden de despliegue en cada entorno: migraciones → backend → frontend.
- Falta darle `MATRIXBEAUTY_API` y `ADS_ANALITICA_API_KEY` en su entorno, y
  fijar a propósito contra qué base apunta: nunca se hereda.
- Falta decidir cada cuánto corre. Si es diaria, con los últimos 7 días y
  después de la hora límite de carga, la skill ya está escrita para eso.

## Reglas de la casa

- Las skills se escriben en español, como el resto del proyecto.
- Nada de credenciales, llaves ni datos de clientes en el repositorio.
- Una skill dice qué NO puede hacer, no solo qué hace: es lo que evita que
  alguien le pida algo para lo que no tiene permiso.
