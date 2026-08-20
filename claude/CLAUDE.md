# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es

Calculadora de boletas de honorarios (Chile) en un **único archivo autocontenido**:
`calculadora-boletas.html`. No hay build, dependencias, servidor ni framework: se abre
con `file://` en el navegador. Todo cambio de marcado, estilo o lógica va inline en ese
mismo archivo (`<style>`, `<main>` y `<script>`).

El repositorio es una comparación entre agentes: `claude/` y `kimi/` contienen cada uno su
propia versión del mismo ejercicio. Trabaja solo dentro de `claude/`.

## Comandos

```bash
# Abrir la página
open calculadora-boletas.html

# Chequear la sintaxis del script embebido (no hay suite de tests)
sed -n '/^<script>/,/^<\/script>/p' calculadora-boletas.html | sed '1d;$d' > /tmp/chk.js
node --check /tmp/chk.js

# Verificar un cálculo de forma aislada: reimplementa las pocas líneas puras en node -e
# en vez de montar un runner.
node -e 'const t=0.1525,b=10*39500; console.log(b, b*t, b-b*t)'

# Ver la UF de una fecha, tal como la consulta la página
curl -s https://mindicador.cl/api/uf/15-03-2025
```

`python3` no está disponible directamente (asdf sin versión fijada en este directorio); usa
`/usr/bin/python3` si necesitas un script de edición.

## Arquitectura

`calcular()` es el **único punto de recálculo**: lee los cinco controles del formulario,
arma el arreglo `filas` y pinta `#detalle` más la `#nota` explicativa. Un solo listener
`input` sobre `#formulario` dispara todo; no hay estado intermedio ni framework de vista.

**La fecha de la boleta manda** sobre las dos cosas que dependen del tiempo:
`aplicarTasaDelAnio()` fija la tasa a partir del año, y el botón `#btnUF` trae el valor de
la UF de ese día.

`@media print` oculta lo marcado con `.no-imprimir` (formulario y botones) y deja solo el
detalle. Si agregas controles nuevos, márcalos así o aparecerán en el impreso.

## Invariantes del dominio

No cambiar sin una razón explícita: son decisiones de cálculo, no detalles de estilo.

- **Tasas de la Ley 21.133** (`TASAS`, 2019–2028, gradual hasta 17%): se derivan del año de
  la fecha y quedan editables. Fuera de ese tramo se usa la tasa del extremo más cercano,
  nunca 0%.
- **Quién retiene cambia el líquido, no el bruto.** Si el cliente retiene, entera el
  impuesto al SII y deposita `bruto − retención`. Si no (persona natural, extranjero),
  deposita el bruto completo y el emisor declara el mismo monto como PPM en su F29.
- **Gross-up solo aplica cuando el cliente retiene**: `bruto = líquido / (1 − tasa)`. Si no
  retiene, lo recibido es el bruto y no hay nada que despejar.
- **Fechas como texto `YYYY-MM-DD`**: `new Date(iso)` interpreta UTC y correría el día; usa
  `partes()`, `anioBoleta()` y `fechaLarga()`, que trabajan sobre el string.
- **UF desde mindicador.cl**: `GET /api/uf/DD-MM-YYYY`. `serie` vacía significa fecha futura
  (la UF se publica hasta el día 9 del mes siguiente) o anterior a 2013; en ese caso se
  avisa y el valor queda ingresable a mano. El último valor usado se guarda en
  `localStorage` bajo `valorUF`, así que la página abre utilizable sin conexión.

## Seguridad

La página corre desde `file://`, sin servidor ni backend, así que la superficie es chica pero
tiene tres bordes que hay que cuidar en cada cambio.

- **Nada de `innerHTML` con texto de origen humano.** Hoy `#detalle` se arma por concatenación
  y es seguro solo porque cada valor sale de `pesos`/`num` (números formateados) y las
  etiquetas son literales. En cuanto una fila deba mostrar algo que el usuario escribió —el
  nombre del cliente, un glosa, un RUT— hay que pasar a `createElement` + `textContent`, o se
  abre un XSS de manual. Misma regla para `#nota` y los `.aviso`: ya usan `textContent`,
  mantenlo así.
- **La respuesta de mindicador.cl es dato externo, no fuente de verdad.** Valida forma y rango
  antes de usarla (`Number.isFinite`, valor positivo) en vez de asignar `uf.valor` directo a un
  campo. Solo `https`; nunca interpolar la respuesta en HTML ni pasarla por `eval`.
- **Cero dependencias externas, y así debe quedar.** Sin CDN, sin `<script src>`, sin fuentes
  remotas: no hay tercero que pueda leer lo que se tipea ni cadena de suministro que
  comprometer. Agregar una librería es una decisión de seguridad, no de conveniencia.

Además:

- **La página no debe filtrar datos del usuario.** El único request saliente es la consulta de
  UF, que lleva una fecha y nada más. Los montos, el cliente y el cálculo no salen del equipo;
  cualquier feature que mande algo a un tercero (analítica, tracking, guardar en la nube)
  rompe esa propiedad y requiere decisión explícita.
- **`localStorage` solo para lo no sensible** (`valorUF`, `tema`). Nada de RUT, nombres ni
  historial de boletas: es un archivo local sin cifrar, legible por cualquier página del mismo
  origen `file://` y por quien tenga el equipo.
- **Sin secretos en el archivo.** Todo lo que esté acá es visible: la API de UF es pública y no
  lleva credenciales. Si alguna vez se necesita una API con token, va del lado de un servidor,
  no inline.
- **Nada de `eval`, `new Function` ni `setTimeout` con string.** Si la página llegara a
  servirse por HTTP, agregar un `<meta http-equiv="Content-Security-Policy">` que permita solo
  `'self'` y el origen de mindicador.

## Convenciones

- Toda la interfaz y los comentarios van en español (formato `es-CL`: `$408.580`, `15,25%`,
  `10,00 UF`). Los formateadores `pesos` y `num` son la única vía de formateo.
- La página no es un documento tributario; el cálculo es referencial y la retención efectiva
  es la vigente a la fecha de emisión en el SII.
- No usar `alert`/`confirm`/`prompt`: el estado se comunica en los `<p class="aviso">`.
- Paleta y tipografía viven en variables CSS: `:root` es el tema claro y
  `:root[data-tema="oscuro"]` el nocturno. Cualquier color nuevo se define en ambos bloques
  y se usa por variable, nunca literal, o el otro tema queda roto.
- El tema lo fija un script inline en el `<head>`, antes del primer pintado, para que no
  parpadee: usa la preferencia guardada en `localStorage` bajo `tema` y, si no hay,
  `prefers-color-scheme`. El botón solo alterna ese atributo.
- `localStorage` puede lanzar bajo `file://` según el navegador; accede siempre por los
  helpers `leer()` y `guardar()`, que lo envuelven en try/catch.
