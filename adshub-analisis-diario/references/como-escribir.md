# Cómo se escribe el informe

Dos partes: cómo se redacta para este equipo, y qué Markdown entiende de verdad
la pantalla que lo muestra.

---

# Parte 1 — Cómo se redacta

## Para quién escribes

Marketing y operación de Beauty Glo. Saben de productos, de lives y de ventas.
**No saben de datos ni quieren aprender.** Si una frase necesita que alguien
pregunte "¿y eso qué quiere decir?", está mal escrita.

La prueba: si la persona que lo lee no puede repetirle a otra, en una frase, qué
pasó y qué hay que mirar, el informe falló.

## Las seis reglas

### 1. La cifra primero, la explicación después

| Así no | Así sí |
|---|---|
| Se observa una tendencia negativa en el retorno de la inversión publicitaria | El ROI pasó de 2,4x a 1,1x |
| El rendimiento de la campaña fue subóptimo | La campaña Faja Salomé gastó $430 y devolvió $180 |

### 2. Nada de jerga

| Así no | Así sí |
|---|---|
| Cobertura parcial del dataset | Faltan horas del martes |
| El endpoint reporta valores nulos | No hubo pedidos, así que no hay con qué calcular el ROI |
| CTR de 0.018 | De cada 100 personas que lo vieron, 1,8 le dieron clic |
| Métricas de retención del creativo | Cuánta gente se queda viendo el video |

### 3. Un número sin referencia no dice nada

| Así no | Así sí |
|---|---|
| Se gastaron $1.240 | Se gastaron $1.240, $310 más que la semana pasada |
| El ROI es 1,1x | El ROI es 1,1x: de cada dólar invertido vuelve $1,10, casi sin ganancia |

### 4. Ninguna afirmación sin cifra detrás

Si el dato no alcanza, **dilo**:

> Con 2 de 7 días cargados no se puede concluir si bajó de verdad.

Eso vale más que una conclusión inventada. El equipo puede ir a cargar lo que
falta; una conclusión falsa lo manda a apagar un incendio que no existe.

### 5. Señales, no veredictos

| Así no | Así sí |
|---|---|
| Hay que pausar la campaña Recovery Kit | Recovery Kit gastó $430 en 3 días y no registró pedidos |
| Este creativo es malo | Este video gastó $96 y no ha traído pedidos; el de al lado gastó $88 y trajo 14 |

Tú pones la plata sobre la mesa. La decisión es del equipo.

### 6. Si no pasó nada, el informe es corto

Un informe que inventa hallazgos para parecer útil entrena al equipo a no
leerlo. Tres líneas que dicen "todo estable, nada que mirar hoy" son un buen
informe.

## Los tres campos que se ven

### `titulo`

Que se entienda en la lista, sin abrirlo. Incluye el período.

- ✅ `Anuncios del 13 al 19 de septiembre`
- ✅ `Anuncios del 13 al 19 de septiembre — ROI a la baja`
- ❌ `Informe de análisis publicitario automatizado`

### `resumen`

**En la lista solo se ven dos líneas.** Lo más importante va en la primera.
Dos o tres frases, nunca más.

> El gasto subió a $1.240 (+33%) y el ROI bajó de 2,4x a 1,1x. Dos campañas se
> llevaron $730 sin un solo pedido. Falta cargar el jueves.

### `cuerpo`

Los cinco bloques, en este orden. Los que no aplican, se omiten: no se rellenan.

```
## Antes de creerle a estos números     <- solo si falta algo por cargar
## Qué pasó
## Qué está quemando plata
## Qué está funcionando
## Qué mirar
```

- **Antes de creerle**: qué falta y qué efecto tiene. "Falta el jueves entero y
  al martes le faltan horas: lo de abajo está por debajo de lo real."
- **Qué pasó**: inversión, ingreso, ROI y pedidos, con el cambio contra el
  período anterior. Nombra el día de ayer aparte, que es el que todos buscan.
- **Qué está quemando plata**: de las `alertas`. Cuánto costó, qué devolvió.
- **Qué está funcionando**: lo que conviene repetir, con la cifra que lo aguanta.
- **Qué mirar**: dos o tres cosas. Cada una con su número. Nunca más de tres, o
  deja de ser una lista de prioridades.

## Cómo se dicen los casos raros

| Lo que trae el API | Cómo se escribe |
|---|---|
| `roi: null` | "no hubo pedidos, así que no hay ROI que calcular" |
| `cambio_pct: null` | "antes no había gasto en esta campaña" |
| `es_nueva: true` | "campaña nueva: no estaba la semana pasada" |
| `ctr: 0.018` | "1,8%" |
| `vista_2s: 0.5` | "la mitad llega a los 2 segundos" |
| `roi: 0.4` | "por cada dólar vuelven 40 centavos: está perdiendo" |
| `monedas: ["USD","COP"]` | "hay cifras en dos monedas; no se pueden sumar" |
| `dias_cargados` < `dias_esperados` | "faltan N días por cargar" |

**Nunca** escribas "ROI de 0" por un `null`, ni "subió 100%" ni "subió infinito"
por un `cambio_pct: null`.

---

# Parte 2 — El Markdown que la pantalla entiende

El `cuerpo` **no** se pinta con una librería de Markdown. Lo pinta un renderer
propio y mínimo (`TextoMarkdown.tsx`), a propósito: el texto viene de un cliente
automático y meter HTML ajeno en la página es exactamente como se cuela un
script.

**Lo que no reconoce se muestra tal cual, en crudo.** Un enlace mal puesto le
aparece al equipo como `[texto](url)`.

## Sí funciona

| Qué | Cómo |
|---|---|
| Títulos | `#` hasta `######` — del cuarto en adelante se ven iguales |
| Negrita | `**texto**` |
| Cursiva | `*texto*` |
| Código | `` `texto` `` |
| Bloque de código | tres tildes invertidas |
| Listas | `- item` o `* item` |
| Listas numeradas | `1. item` o `1) item` |
| Cita | `> texto` |
| Línea horizontal | `---`, `***` o `___` |
| Tablas | ver abajo |

## No funciona — se ve en crudo

| Qué | Qué pasa |
|---|---|
| `[texto](url)` | aparece literal, con corchetes y paréntesis |
| `![imagen](...)` | igual |
| HTML | igual |
| `~~tachado~~` | aparece con las virgulillas |
| Listas anidadas | se aplanan a un solo nivel |

Como los enlaces no se pintan, **escribe la ruta en palabras**: "está en
AdsHub > Cargas", no `[Cargas](/hubs/ads/cargas)`.

## Las tablas, con cuidado

```
| Campaña | Gastó | Devolvió |
|---|---|---|
| Faja Salomé | $430 | $180 |
```

Tres reglas:

1. **La línea separadora lleva al menos dos guiones por celda** (`|---|---|`).
   Con uno solo (`|-|-|`) se pinta como una fila de datos más.
2. La primera fila es el encabezado. Mismo número de celdas en todas.
3. Las barras de los extremos son opcionales desde el arreglo del 20 de
   septiembre, pero **póngalas igual**: una tabla pareja se lee mejor en el
   JSON, donde el cuerpo va como una sola línea con `\n`.

## Los hallazgos son texto plano

`titulo` y `detalle` de cada hallazgo se pintan **sin Markdown**. Unos `**` ahí
se ven como asteriscos. Como las `alertas` ya vienen redactadas por el backend,
basta con copiarlas tal cual.

Vienen con punto decimal (`$430.00`, `ROI 0.9x`) porque las arma Python. **No las
reescribas para ponerles coma.** Si las tocas dejan de coincidir con lo que
calculó el backend, y ese es justo el error que esta skill existe para evitar. En
tu texto del `cuerpo` sí escribe como se escribe en español: `$430`, `2,4x`.

## Esqueleto que sí se ve bien

```
## Antes de creerle a estos números

Falta el jueves entero y al martes le faltan horas. Los totales de abajo están
por debajo de lo real.

## Qué pasó

| | Esta semana | Semana pasada |
|---|---|---|
| Inversión | $1.240 | $930 |
| Ingreso | $1.364 | $2.232 |
| ROI | 1,1x | 2,4x |
| Pedidos | 38 | 61 |

Ayer se gastaron $210 y volvieron $198.

## Qué está quemando plata

- **Recovery Kit**: $430 en 3 días, ningún pedido.
- **Faja Salomé**: gastó $310 más que la semana pasada y el ROI pasó de 3,1x a 1,4x.

## Qué está funcionando

- El video de la faja azul gastó $88 y trajo 14 pedidos (ROI 3,8x). La mitad de
  quienes lo abren llega a los 2 segundos.

## Qué mirar

1. Recovery Kit: $430 sin pedidos en 3 días.
2. Cargar el jueves, que falta entero.
3. El video de la faja azul, que es el único por encima de 3x.
```
