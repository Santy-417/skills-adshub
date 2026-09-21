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
| `analisis-anuncios/` | Lee lo que está cargado en el AdsHub, escribe un análisis y lo guarda. El equipo lo ve en AdsHub > Informes |

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

## Reglas de la casa

- Las skills se escriben en español, como el resto del proyecto.
- Nada de credenciales, llaves ni datos de clientes en el repositorio.
- Una skill dice qué NO puede hacer, no solo qué hace: es lo que evita que
  alguien le pida algo para lo que no tiene permiso.
