---
name: migrar-normativa-dte-2.0-restaurante
description: >-
  Portar la Normativa de Facturación DTE 2.0 (Ministerio de Hacienda, El Salvador)
  a sistemas cliente de formato RESTAURANTE/food-service: raíz administrativa con
  motor DTE + subcarpeta de cobro POS + app de meseros + API móvil. Usar cuando el
  destino tenga ese perfil (cobro de mesas, órdenes, impresión de comandas,
  sesiones compartidas entre niveles, helpers duplicados por subcarpeta) o al
  invocarse con /migrar-normativa-dte-2.0-restaurante. Es una EXTENSIÓN de
  migrar-normativa-dte-2.0: ejecuta esa skill como fundamento y aplica las
  adiciones, sustituciones y recorridos específicos del formato definidos aquí.
  NO sirve para destinos sin subcarpetas de subsistema (usar la skill base).
---

# Migrar Normativa DTE 2.0 a un sistema cliente de formato restaurante

## Qué es esta skill y cómo se relaciona con la base

**Esta skill NO reemplaza a `migrar-normativa-dte-2.0`: la REQUIERE.** La skill
base define el método general (principios, FASE 1 de BD con sync_schema y
catálogos MH, port selectivo por commit, auditorías de coherencia lógica
(FASE 2.5), de correo (FASE 2.6) y de símbolos (FASE 2.7), y el cierre con
registro). Ejecútala COMPLETA como fundamento. **Esta skill define ADEMÁS tres
fases propias del formato** — FASE 0 (mapa/verificación del destino), FASE 2.4
(lectura comprensiva y adaptación de esquema/datos) y FASE 2.8 (smokes de
ejecución y recorridos de usuario) — que la base NO trae: en destinos de
restaurante son tan obligatorias como las de la base, y esta skill manda sobre
ellas. En resumen, lo que CAMBIA en destinos de formato restaurante:

- **SUSTITUCIONES**: donde esta skill redefine un punto para el formato (el
  mapa del destino se VERIFICA contra el perfil esperado en vez de descubrirse;
  los recorridos de smokes son los del restaurante), lo definido AQUÍ manda.
- **ADICIONES**: los frentes y trampas propias del formato (cobro con convención
  de montos, impresión por pipes, triples helpers, endpoints SSP) son trabajo
  obligatorio adicional que la base no conoce.
- Todo lo que esta skill no toque se ejecuta tal como dice la base.

No re-narrar aquí el método general: para BD, catálogos, commits normativos,
correo, símbolos y registro, la base es la única fuente. Si esta skill y la base
contradicen algo NO específico del formato restaurante, gana la base (y anota la
contradicción para corregir esta skill).

## El perfil del destino (mapa esperado)

Los destinos de este formato comparten una arquitectura que la iteración real
confirmó una y otra vez. **Espera este mapa y verifícalo; no lo redescubras
desde cero:**

- **Raíz**: administrativo + MOTOR DTE (`generar_dte.php`, `_helper_dte_class.php`,
  `models/` o `clases/`, `jsondte/`, `pdfdtes/`, PDFs `dte_*_pdf.php`, tickets).
  Los admins de ventas/facturas/compras viven aquí.
- **`pos/`**: consola de cobro del restaurante (órdenes de mesa, modal de pago,
  facturación). **Es el subsistema CRÍTICO**: por aquí pasa toda venta real.
- **`meseros/`**: app del mesero (venta por comanda). Importancia secundaria:
  recibe los cambios de helpers/sesión/llevar-domicilio, no los de cobro-DTE.
- **`api_slim/`** o equivalente: API móvil independiente (JWT, `vendor/` propio).
  No participa del port normativo; solo verificar que nada compartido se rompa.
- **Bootstrap por subsistema**: cada nivel trae SUS copias de `_conexion.php`,
  `_core.php`, `_helper.php` (y `_session.php`), con variantes propias. Un
  helper se edita en su nivel y en sus copias hermanas cuando el flujo lo cruza.
- **Un solo login**: la raíz autentica y REDIRIGE por perfil de empleado
  (cajero → página de órdenes de `pos/`, mesero → `meseros/`, demás → dashboard
  raíz). pos y meseros NO tienen login propio real: comparten la sesión raíz.
- **Admins de DataTables**: la tabla visible la sirve un endpoint SSP
  (`*_dt.php`) invocado por el JS que la página carga — la función PHP de render
  dentro de la página suele estar MUERTA o parcial.

**La consecuencia central del formato:** el sistema es UNO solo con tres
puntos de entrada, no tres sistemas. Todo lo que toque sesión, login,
helpers o flujos de cobro debe resolverse para los TRES niveles a la vez, y
las verificaciones son RECORRIDOS que cruzan niveles (raíz → pos), nunca
artefactos aislados.

## FASE 0 (del formato) — Verificación del mapa esperado

Antes de portear: buscar y LEER completos los archivos de contexto del destino
(`AGENTS.md`, `CLAUDE.md`, `README.md` — raíz y de cada subcarpeta); si no
existen, construir ese contexto por exploración (bootstraps, entry points,
estructura, menús) y **crear `AGENTS.md`** al terminar la fase. El artefacto
obligatorio es `tmp/MAPA_DESTINO.md`: las fases siguientes citan ese mapa en
cada verificación de "¿dónde se ejecuta?" y "¿existe esta columna?", y el
registro del cierre lo incluye como evidencia; si durante el port aparece un
dato que contradice el mapa, actualizarlo y anotar la corrección. Con este
formato, el trabajo es **CONFIRMAR el mapa esperado**, no descubrirlo.
Checklist de verificación mínima antes de portear:

1. **Existencia y variantes**: confirmar que existen los niveles del perfil
   (raíz, `pos/`, `meseros/`, API móvil) y sus bootstraps. Anotar VARIACIONES
   del destino respecto al perfil (p. ej. cobro que no escribe resúmenes de
   factura, helpers con firma distinta entre copias, un nivel ausente): cada
   variación es una fila del mapa y una decisión de port.
2. **Sesión real**: extraer el patrón `session_name` de los tres niveles y
   resolver QUÉ nombre de cookie resulta por nivel (raíz, páginas de pos/
   raíz-level, endpoints en subdirectorios de pos como `procesos/`). Si los
   nombres no coinciden entre niveles que comparten login, el port debe
   unificarlos ANTES de portear DTE (ver Regla 1).
3. **Render paths vivos de los admins tocados**: para cada admin de
   ventas/facturas que el port modifique, grep de la URL ajax en el `.js` que
   la página carga → ese endpoint SSP es la vista viva. Anotar th↔dt de cada
   tabla (cuántas columnas declara el HTML vs cuántas devuelve el endpoint).
4. **Redirecciones por perfil**: leer el `login.php` raíz y anotar a dónde va
   cada perfil. Ese destino es el recorrido canónico de la FASE 2.8.
5. **Puntos de cobro**: listar TODOS los archivos que ejecutan el cobro
   (páginas de órdenes, sus JS, builders del payload de pago). La regla
   "origen manda el QUÉ, destino manda el DÓNDE" de la base aplica con fuerza
   aquí: si hay dos páginas de órdenes activas (orden y orden2, actual y
   respaldo con fecha), AMBAS reciben cada cambio del cobro.
6. **Residuos del formato**: las páginas `venta*.php` de `pos/` suelen ser
   residuos heredados (sus procesos JS no existen; el cobro real vive en las
   páginas de órdenes). Clasificarlas con el método de evidencia exhaustiva
   de la base ANTES de invertir en ellas, y documentar el veredicto.
7. **Esquema real + valores con convención**: columnas reales de `factura`,
   `orden`, `orden_detalle`, `cliente`, `sucursal`, `jsonDTE` y las tablas
   normativas; y CÓMO las usa el cobro del destino (qué escribe y qué no).
   El detalle de DTE sale del detalle de ORDEN en este formato, no de
   `factura_detalle` (vestigial): verificarlo y anotarlo en el mapa.
8. **Carpetas de salida y dueño**: `img/dte/`, `pdfdtes/`, `jsondte/` con
   existencia, propietario y permisos del usuario WEB (no del CLI). Crear las
   que falten con `mkdir` + dueño correcto, o portear el mkdir defensivo.

## Reglas duras del formato (trampas ya resueltas: aplicar de entrada, no descubrir)

Cada regla viene de un defecto REAL ocurrido en el port a un sistema de este
formato. No son advertencias opcionales: son decisiones de diseño que este
formato exige y que el port debe materializar DESDE el primer intento. Todas se
verifican en la FASE 2.8 con el recorrido correspondiente.

### Regla 1 — Sesión UNIFICADA entre niveles

Los tres `login.php` (raíz, pos, meseros) y todos los `_session.php` deben
resolver el MISMO nombre de cookie. El patrón probado: el nombre se calcula
contra la RAÍZ del proyecto, constante para todos los puntos de entrada:

```php
session_name(preg_replace('/[^A-Za-z0-9_-]/', '', basename(dirname(__DIR__))) ?: 'DTE_SESS');
```

- En archivos de NIVEL RAÍZ (login.php, dashboard, admins): `basename(__DIR__)`.
- En archivos de pos/ y meseros/ (y sus subdirectorios: la resolución es sobre
  el directorio del NIVEL, no sobre el del archivo): `basename(dirname(__DIR__))`.
- **Por qué**: con `basename(__DIR__)` por subsistema, pos resolvía "pos",
  la raíz el nombre del proyecto → el login raíz redirigía al Cajero a
  `pos/ordenes.php`, que con OTRA cookie veía sesión vacía → bucle de login.
- **Verificación obligatoria (FASE 2.8, recorrido A)**: crear sesión desde el
  login raíz y leerla desde una página de pos y una de meseros; la identidad
  (`$_SESSION['usuario']`, `id_empleado`, `id_sucursal`) debe ser la misma en
  los tres. Un solo re-login al desplegar (cookies nuevas = sesiones muertas:
  avisarlo al usuario en el informe).
- El aislamiento frente a OTROS proyectos hermanos en el mismo dominio se
  conserva: el nombre sigue derivado del directorio del proyecto.

### Regla 2 — Helpers triplicados y el require cruzado de `_core` desde `clases/`

- Todo cambio en `_helper.php` de la raíz evalúa si requiere copia en
  `pos/` y `meseros/` (funciones que consumen esos niveles: impresión,
  llevar/domicilio, sesiones, constantes compartidas). La copia lleva las
  variantes propias del nivel, no es copy-paste ciego.
- **NUNCA un `clases/` (o `models/`) compartido requiere el `_core.php` de la
  raíz.** Cadena real que reventó: endpoint en `pos/procesos/` → modelo de
  pos → modelo base → `QueryBuilder` → require de `_core.php` RAÍZ → carga
  `_helper.php` + `_conexion.php` de la raíz → colisión con los de pos
  ("Cannot redeclare") → 500. El desenlace dependía del CWD del request
  (desde `pos/` sobrevivía por dedupe de realpath; desde `pos/procesos/`
  cargaba ambas versiones y moría): por eso el fallo se veía en unos
  endpoints sí y otros no, sin tocar el mismo código. Los modelos compartidos
  deben ser autocontenidos (sus cargadores ya bootean el `_core` de su nivel)
  y los cargadores por nivel deben deduplicar por realpath.
- **Verificación**: ejecutar por HTTP local (o CLI con el cwd del nivel) un
  endpoint de CADA subdirectorio que use modelos: 200 con JSON limpio.

### Regla 3 — El admin vivo es el endpoint SSP, no la función de render de la página

- Antes de modificar cualquier menú por fila, columna de estado o acción de un
  admin: identificar el endpoint SSP real (URL ajax del JS) y hacer ahí el
  cambio. Insertar el cambio en `mostrarDatos()`/render PHP de la página
  cuando la tabla la sirve el SSP = código muerto que "está portado" y el
  usuario jamás ve (ocurrió con el evento de retorno: quedó en una función que
  la página ni ejecutaba).
- **Conteo th↔dt SIEMPRE**: cada columna agregada al `<thead>` exige la
  columna correspondiente en el array del endpoint (y re-indexar las
  siguientes). El síntoma del desalineo es el warning "Requested unknown
  parameter 'N'" de DataTables.
- **WHERE calificados**: al portar columnas nuevas (la migración agrega
  `id_sucursal` a `cliente`, etc.), cualquier WHERE de endpoint con JOIN que
  use un nombre ahora ambiguo se califica con el alias de tabla
  (`F.id_sucursal`). Barrer TODOS los `*_dt.php` del destino buscando el
  nombre agregado sin prefijo en queries con JOIN.
- **Hoisting de funciones PHP**: una función definida con guard
  `function_exists` DESPUÉS del código que la llama NO existe al momento de la
  llamada (las declaraciones condicionales no se hoistean). Los helpers que un
  endpoint necesita van INCONDICIONALES al inicio del archivo o antes del
  primer uso top-level. Verificar posición real, no presencia del nombre.
  Igual para las clases: instanciar la que define el `factForm.php` del
  DESTINO (típicamente `FactForm`; `PDFact` suele vivir en otro archivo que
  estos generadores no cargan).

### Regla 4 — Dos generadores PDF conviviendo (normativo vs legacy del admin)

Este formato suele tener DOS caminos de PDF para el mismo documento:
el **normativo** (flujo del cobro/anulación → generador 2.0 con marca de agua,
QR, logo, envío) y el **legacy preexistente** (botones "Pdf"/"Enviar" del
admin → generador propio del destino). Reglas:

- TODOS los elementos visuales normativos (marca de agua ANULADO, sello de
  evento de retorno, distrito, versión) se espejan TAMBIÉN al generador
  legacy si el admin lo usa, con el diseño literal del normativo — salvo que
  el usuario decida lo contrario (la decisión de unificar/re-cablear es SUYA;
  por defecto conviven).
- La salida `output=file`/`output=text` del generador normativo debe responder
  JSON limpio (ob_start + captura de notices) para que el JS del flujo pueda
  consumir el objeto y no texto contaminado.
- Carpetas de QR/PDF con `mkdir` defensivo + dueño `www-data` (si el CLI
  creó la carpeta, php-fpm no puede escribir: Permission denied).

### Regla 5 — Correo: columnas del destino y guards de CC/BCC

- La factura del destino usa `alias_tipodoc`, NO `tipo_documento` (columna
  que no existe). Grep de `tipo_documento` en TODOS los bloques portados de
  correo/PDF/anulación antes de cerrar: este defecto se repitió tres veces en
  el port real y su modo de falla es silencioso (destinatario vacío → el
  proveedor de correo rechaza y el catch lo traga).
- Guards de CC/BCC LOOSE con vacío: `== false || == 'false' ||
  trim((string)$x) === ''` — el `.env` trae esas claves VACÍAS y un string
  vacío enviado como Cc rechaza el envío completo.
- El destinatario real del DTE lo resuelve el receptor del JSON (que refleja
  el email capturado/guardado en el cobro), no una variable suelta del
  generador: trazar esa cadena en la FASE 2.6.

### Regla 6 — Recargas POST-acción, nunca en carrera

Toda acción que dispara axios/fetch (generar DTE, anular, enviar correo,
evento de retorno) debe recargar la vista del admin DESPUÉS de que la promesa
termina (`await`/`.then()` con pausa breve), nunca con un `setTimeout` que
corre en paralelo: la recarga aborta el PDF/correo a medio camino y el
usuario ve éxito a medias sin error. Aplica a los DOS caminos gemelos del
formato: el del cobro (JS auto) y el del admin (JS manual) — auditarlos
JUNTOS, comparten helpers.

### Regla 7 — Nunca entregar una función `async` a jQuery 2.x

Las páginas de `pos/` suelen cargar jQuery 1.x/2.x, cuyo `isFunction` usa
`[object Function]`: una función `async` da `false`. Pasada como callback se
descarta en silencio y el flujo muere ahí, sin error visible:

- `$({to:0}).animate({to:1}, d, async function(){...})` → jQuery la toma como
  CURVA de animación y el temporizador entra en bucle lanzando
  `n.easing[this.easing] is not a function` (miles por segundo). El toast de
  antes de ese tramo sale, y eso engaña: el error no es del toast.
- `{duration: d, complete: async function(){...}}` → tampoco corre.

Patrón: `$({to:0}).animate({to:1}, d, function(){ (async () => {...})(); })`.
Aplica a `animate`, `.on`, y cualquier opción `complete/done/success` de jQuery;
no a promesas de axios (`.then(async ...)`) ni a `setTimeout`.
**Verificación (recorrido B)**: línea de tiempo EN la página (registrador que
escribe cada cambio de toasts/Swal en `sessionStorage`, que sobrevive a la
recarga) — un muestreo desde fuera con `evaluate` en bucle se pierde los pasos
intermedios.

### Regla 8 — Ninguna rama del flujo puede terminar "en el aire"

Auditar el JS del cobro y del admin rama por rama (éxito, rechazo de MH,
`codigoMsg` ausente, anulación, fallo del reintento): **cada rama termina en
recarga o en un aviso que el operador ve**. Fallas reales de este formato:

- escribir el resultado en un elemento del modal que no existe en esa página
  (`#msj`): `null.textContent` corta todo lo que sigue. Verificar con grep que
  cada `getElementById`/selector usado en el flujo existe en la página que
  carga ese JS.
- la rama de rechazo que llama a la corrección de correlativo y no hace nada si
  esa corrección falla: la pantalla queda con la venta abierta. El reintento
  debe devolver si arrancó; si no, aviso con el motivo + recarga.
- los toasts de `pos/` duran ~1.5 s: el aviso del sello de MH debe persistir
  (`toastr.success(..., {timeOut: 6000, extendedTimeOut: 0})`) y la recarga
  esperar a que haya estado en pantalla unos 4 s, o el cajero no lo lee.

**No dar un flujo por arreglado porque el sello está en la BD o "no hubo
errores de JS"**: MH acepta y guarda aunque el navegador se haya detenido
antes. La evidencia es la línea de tiempo hasta la recarga.

### Regla 9 — Modelos compartidos cargados desde `pos/` (ampliación de la Regla 2)

Un `require` nuevo de `_helpers.php` de raíz desde un modelo (`models/Dte.php`)
arrastra `_core.php` + `_conexion.php` de raíz dentro de una página de pos:
redeclara los ~50 helpers de BD, las constantes de impresión y pisaría
`$conexion` a mitad de una transacción. Materializar:

- en el modelo: `if (!function_exists('_query')) { require .../_core.php; }`
  (el bootstrap de pos ya abrió conexión);
- en `_helpers.php` de raíz: `_conexion.php` solo si no hay `_query`, y
  `function_exists`/`defined()` en TODO nombre que también declare
  `pos/_helpers.php` o `pos/_conexion.php` (comparar los tres archivos por
  lista de funciones, no por lectura);
- el require de `_helpers.php` solo con `realpath`, nunca relativo (desde pos
  resolvería al de pos).

**Verificación (recorrido D)**: replay por CLI desde `pos/` que cargue
`_core.php` de pos y el hub, con `$_SESSION['id_usuario']` simulada (los
modelos hacen `isset($_SESSION["id_usuario"]) or exit(...)`), sin un solo
notice. Ojo: `_session.php` redirige sin sesión, hay que sembrarla antes.

### Regla 10 — Respuestas JSON limpias y nombres de parámetro JS↔PHP

- Un notice de PHP antes del `echo json_encode` rompe `JSON.parse` y detiene
  el flujo (visto en el endpoint de anulación). En cada endpoint del dispatcher
  que responde JSON: `ob_start()` al entrar y `while (ob_get_level()>0)
  ob_end_clean();` justo antes del echo.
- El mismo dato viaja con DOS nombres según quién lo manda: `tipodoc` llega
  como alias (`COF`) desde el cobro y como código MH (`01`) desde el modal de
  anular del admin. Cada función que traduce tipo debe aceptar ambos (si es
  numérico es código MH) y devolver vacío si no hay fila, nunca leer
  `$row[0]` de un resultado sin filas. Y cada par JS→PHP del flujo se revisa
  nombre por nombre (`tipodoc` vs `tipo_doc` costó un reintento que nunca
  arrancaba).
- Los errores atrapados en `catch` (correo sobre todo) no existen para el
  operador si solo van a `error_log`: o se muestran, o se declara en el
  registro como fallo silencioso. Para verlos sin logs del servidor: sonda por
  CLI que carga el bootstrap con sesión simulada y `ini_set('error_log',
  archivo)`.

### Regla 11 — Página con DOS modales: ningún init de widget puede ser global

Una página del subsistema suele alojar el modal de AGREGAR (render inline) y el
de EDITAR (contenido AJAX con su script inline). El script inline del modal
AJAX ejecuta `$(".select2")`, `$(".i-checks")`, `$("#porcentaje").niceSelect()`
**globales** y select2 4.x, al re-inicializar, DESTRUYE y recrea la instancia
del otro modal — el select2 recreado queda con el `dropdownParent` del modal
que se está inyectando; al cerrar ese modal su contenido se borra y el dropdown
del otro queda huérfano del DOM (`$dropdown.parents()` vacío): el click no
despliega nada y recargar "lo arregla" (por eso el usuario reporta "funciona
solo después de recargar"). Fix: TODOS los inits de widget llevan el scope de
su propio modal, en ambos scripts — `$("#modalAgregar .select2")`,
`$("#modalEditar .select2")`, etc. Igual para iCheck: cada modal inicializa su
propio `.i-checks` (el checkbox de agregar que depende del init global del
script de editar está muerto si nunca se abrió editar).

**Doble toggle del checkbox invisible**: con iCheck, el click físico puede
caer sobre el `input` invisible y disparar el toggle nativo Y el de iCheck
(netos: el checkbox no cambia nunca). Diagnóstico: `elementFromPoint` en las
coordenadas del click devuelve INPUT. Fix por CSS, no por handlers extra (un
`label.on('click')` propio duplica con el handler que iCheck ya liga al
label): `#modalX .i-checks input{pointer-events:none}` — el cuadro lo alterna
el envoltorio y el texto lo alterna el handler del label de iCheck, una sola
vez.

**El checkbox serializa "on"**: PHP que procesa el alta debe aceptar
`$retiene == 'on' || $retiene == 1` — comparar solo con `1` hace que el flujo
de INSERTAR ignore el checkbox mientras el de EDITAR (que solo mira el
porcentaje) "funciona": el usuario reporta "guarda en editar pero no en
agregar".

**Verificación**: en el navegador real, alternar los dos modales en ambos
órdenes (agregar→cerrar→editar y editar→cerrar→agregar) y click real sobre
cada select y el checkbox; el estado interno se inspecciona con
`jQuery(el).data('select2').$dropdown.parents().length > 0`.

### Regla 12 — Rutas de JS compartido: `rootUrl` solo existe en `pos/`

`js/constantes.js` define `rootUrl` y lo cargan las páginas de `pos/`; las
páginas de la RAÍZ no lo tienen. Consecuencias ya vividas:

- Un JS compartido que arma URLs con `rootUrl` falla silenciosamente en la
  raíz (fallback equivocado → URL fuera del proyecto, 404 o redirección a un
  lugar inexistente). Elegir por contexto: página raíz standalone → rutas
  relativas simples; página de pos → `rootUrl`. Nunca un fallback `'..'` sin
  verificar a dónde resuelve.
- Un archivo de la raíz reconvertido en FRAGMENTO de modal (se inyecta en una
  página de `pos/`) tiene TODAS sus rutas relativas — `<script src>` y URLs
  AJAX — resolviendo contra la página host. Revisar ambas capas: el tag del
  script Y las URLs dentro del JS. Si el fragmento vuelve a página completa,
  revertir las dos capas, no una.
- Tras recargar JS corregido, exigir Ctrl+Shift+R al usuario: los scripts sin
  cache-busting sirven la copia vieja y el fix "no funciona".

## El frente crítico: el cobro del restaurante

Por aquí pasa TODA venta real. El flujo canónico del formato: página de
órdenes de `pos/` → JS de órdenes (modal de pago) → payload al endpoint de
cobro → finalización de orden + fila de factura → generación DTE automática
desde el navegador → PDF/correo → recarga. Todo cambio normativo u opcional
del flujo de venta se aplica a este frente con estas reglas concretas:

### 1. El cobro debe escribir los campos que la normativa 2.0 consume

El cobro legacy del formato normalmente NO escribía los campos que los
generadores 2.0 leen de `factura`. Al portar, el endpoint de cobro debe
escribir: `sumas` (con convención por tipo, ver abajo), `credito`,
fila(s) en la tabla de pagos de factura (`factura_pago` o equivalente),
`datos_extra` (observaciones del modal) y el email del cliente si cambió.

**Convención de `sumas` (se aprendió con un rechazo real de CALCULO
INCORRECTO):** `factura.sumas` alimenta el `totalGravada` del RESUMEN del DTE,
mientras el CUERPO se calcula desde el detalle:

- **COF (01, consumidor final) y tipos con precios CON IVA**: `sumas` = total
  cobrado tal cual (el cuerpo también suma con IVA: cuadran).
- **CCF (03) y tipos con precios NETOS**: `sumas` = `round(total/(1+iva),4)`
  (el cuerpo publica netos; el resumen debe netear igual o MH rechaza por
  descuadre cuerpo/resumen).
- Todo TIPO NUEVO define su convención al escribir `sumas`; la regla semántica
  es: `sumas` = la base que el resumen del tipo espera. Reparar con UPDATE las
  filas históricas del tipo afectado si ya existen.
- Verificación canónica (FASE 2.8): correr los calculadores puros
  (cuerpo+resumen) por tipo con una orden real y comparar totales ítem a ítem.

### 2. El payload del cobro se toca en TODOS los builders

El JS del cobro suele construir el payload en DOS sitios (dos builders: el del
botón de imprimir/pagar y el del flujo alternativo). Cualquier campo nuevo
(observaciones, correo del cliente) se agrega en AMBOS. Igual con los
handlers de cambio de cliente: el prefill del correo en el modal se hace en
TODOS los handlers `cambia_cliente` existentes, alimentado por el endpoint
que devuelve la ficha (verificar que devuelva `email`).
Los selectores de inputs insertados dinámicamente en el modal se escriben por
ID real (`$('#correo_cliente')`), no por contenedor (`$('.modal-body #x')`):
la colocación física del input dentro de la estructura del modal varía y el
selector por contenedor se desincroniza.

### 3. Buscador de clientes del modal de pago (select2 AJAX)

- `type: "post"` SIEMPRE (el dispatcher del backend responde a `$_POST`; un
  GET silencioso produce "Searching" eterno).
- `dropdownParent: <el modal del pago>` para que el `enforceFocus` de
  Bootstrap no le robe el foco al cuadro de búsqueda (síntoma: no se puede
  escribir).
- El endpoint del buscador EXCLUYE clientes sin nombre (`AND nombre != ''`):
  este formato acumula clientes VACÍOS generados por órdenes para llevar/
  domicilio sin cliente, y `ORDER BY nombre` los pone siempre en la primera
  página (dropdown de `<li>` vacíos).
- Guard en llevar/domicilio (pos y meseros): sin cliente/nombre no inserta
  cliente vacío; usa el cliente genérico de mostrador (-1). Reportar al usuario
  la limpieza opcional de los vacíos históricos (con conteo y consulta lista),
  nunca ejecutarla por iniciativa propia.

### 4. El flujo automático de DTE del cobro (JS auto)

- El JS auto (`dte_auto` o equivalente) y el JS manual del admin comparten
  helpers y rutas gemelas: auditarlos JUNTOS (Regla 6 y FASE 2.6 de la base).
- La generación automática llama al generador normativo (no al legacy del
  admin) y espera la promesa completa antes de recargar.
- La transmisión a MH en este formato sale del NAVEGADOR vía el proxy CORS de
  pos (`proxydte.php`): el mapa destino debe tener esa cadena documentada
  (JS auto → proxy → respuesta → guardado) y el port NO la re-cablea
  server-side aunque el origen lo haga así.

### 5. Prefill y modal de pago: verificación del recorrido completo

El recorrido de aceptación de este frente es: abrir modal → buscar cliente →
seleccionar → nombre y correo prellenados → cobrar → orden finalizada +
factura con `sumas` correcto por tipo → DTE generada → PDF con logo/QR →
correo enviado → admin recargado mostrando la fila nueva con sello.

## Impresión local: contratos por pipes, no líneas libres

La reimpresión de comandas/tickets de este formato NO va por HTML: la página
de órdenes finalizadas arma un TEXTO que manda a un script local de impresión,
y esos scripts parsean por PIPES con un número FIJO de campos. Reglas:

1. **Descubrir el contrato real ANTES de tocar la impresión**: leer el script
   de impresión de cada plataforma cliente (típicamente uno para caja Linux y
   otro para Windows/meseros) y contar sus `explode('|')`/campos esperados.
   El contrato usual: 6 campos para caja (`tiquete|fecha|ubicacion|venta|total|totaltxt`)
   y 7 para meseros/Windows (`tiquete|fecha|mesero|venta|total|a_pagar|totaltxt`).
   Son contratos DEL CLIENTE: pueden variar — el fuente de verdad es su script,
   no esta skill.
2. **El texto de la orden va en UN solo string con ese número exacto de
   campos**, no en líneas libres con pipes por ítem: un pipe de más desalinea
   el explode (síntoma real: un solo ítem impreso, texto corrido, totales
   equivocados). El detalle de ítems va DENTRO del campo que el script
   reserve para él, con el separador interno que ese campo use (comas/saltos).
3. **Dos flujos de impresión conviven**: el flujo de cocina/bar marca
   `impreso='0'` al imprimir (y por eso un `print_orden` filtrando ese flag
   devuelve vacío en reimpresiones); la REIMPRESIÓN de órdenes finalizadas
   usa una función aparte sin filtro de `impreso`, con TODOS los ítems. No
   "unificar" ambas: son flujos distintos con semántica distinta.
4. **Orden de ítems**: `ORDER BY id_producto` (o el criterio que use la
   pantalla al mostrar la orden) para que el ticket calce con lo que el
   usuario ve; no el orden de inserción.
5. La función de impresión para finalizadas vive en el helper del nivel
   (Regla 2: copia en pos y meseros), y la página que reimprime elige formato
   según la plataforma del cliente (SO de la caja) cuando aplique.
6. Verificación (FASE 2.8): reimprimir una orden real y CONTAR los campos del
   texto generado contra el `explode` del script destino, por cada formato.

## Distritos select2 en el formato restaurante (los modales mandan)

La base define el port de distritos con búsqueda y cascada inversa. En este
formato añade tres exigencias propias, todas con falla real detrás:

1. **Los modales con contenido AJAX re-inicializan widgets con scripts
   inline.** El modal de edición suele inyectar el HTML del formulario por
   AJAX (archivo `editar_*.php` del subsistema) con SU script inline que hace
   `$(".select").select2()` genérico DESPUÉS del load — pisando el init AJAX
   del distrito puesto por el JS principal ("la última llamada gana"). Fix:
   en el script inline, `.not('#distrito')` en TODOS los inits genéricos y
   inicializar ahí el select2 AJAX del distrito con `dropdownParent` del
   modal correspondiente. Esto aplica a cada vista de edición de cada nivel
   (agregar y editar son archivos/scripts DISTINTOS: el fix va en ambos).
2. **Cascada inversa con triggers namespaced**: `trigger("change.select2")`,
   nunca `trigger("change")` plano — el sitio ya tiene handlers `change`
   propios que recargan municipios y limpian la selección (síntoma: al elegir
   distrito, el municipio vuelve a "Seleccione"). Tras recargar, re-seleccionar
   explícitamente el municipio destino.
3. **El frente cuenta por ARCHIVO**: agregar y editar por nivel (ABM raíz +
   ABM pos si ambos existen) son piezas separadas; el frente solo cierra cuando
   TODAS materializan la búsqueda. Es el análogo de venta/venta2 de la base.

## FASES 2.4-2.8 del formato: recorridos obligatorios

Las FASES 2.5 (coherencia lógica), 2.6 (contrato de correo) y 2.7 (colisiones
de símbolos) de la BASE se ejecutan COMPLETAS, tal como las define. Las fases
2.4 y 2.8 las DEFINE esta skill (la base no las trae) y son igual de
obligatorias:

- **FASE 2.4 — Lectura comprensiva y adaptación de esquema/datos**: cero grep
  como método de verificación de contenido. Por cada función/bloque portado,
  leer el cuerpo COMPLETO (grep/sed permitidos para EXTRAER el bloque íntegro,
  de llave a llave o `<?php` a `?>`; prohibido el muestreo de líneas cercanas
  al match) y producir la **tabla de referencias externas** (cada referencia
  usada y no definida → dónde vive en el destino → ✓ / adaptada / PENDIENTE).
  Adaptar columnas y CONVENCIONES de valores al esquema real del destino
  (nada del origen "porque compila": el fallo es silencioso), barrer el radio
  de explosión del out.sql (columnas agregadas que vuelven ambiguo un WHERE
  con JOIN → calificar), y registrar TODA micro-desviación del origen en el
  ledger (qué cambió, por qué, qué verificación de ejecución la valida).
- **FASE 2.8 — Smokes de ejecución local y recorrido de usuario** (abajo).
  **Precondición**: preguntar al usuario AL INICIO (junto con los parámetros,
  vía diálogo interactivo) si autoriza crear una BD TEMPORAL con su dump +
  migración aplicada para correr ahí las verificaciones sin arriesgar datos
  ("se elimina al terminar. ¿La creo?"). Con autorización, TODA sonda corre
  contra esa copia; sin ella, los recorridos quedan PENDIENTES explícitos
  (nunca contra la BD viva, nunca simulados).
  **Regla de DOS ESCENARIOS para la BD:**
  1. **ANTES de la entrega del port (smokes de esta fase): BD TEMPORAL.**
     Todo el esfuerzo de verificación previa ocurre en la copia — NUNCA
     contra la BD viva. La copia se elimina al cerrar el port: no arrastrar
     verificaciones tardías contra ella ni re-copiar datos a ella (copiar
     registros de la BD real a la copia para "ver el error en contexto" es
     el antipatrón que esta regla elimina: lento y duplica la fuente).
  2. **DESPUÉS de la entrega (iteración de fixes reportados por el usuario):
     BD REAL, directo.** Cuando el usuario prueba el sistema y reporta errores
     que las smokes no destaparon, el diagnóstico consulta DIRECTAMENTE la
     BD real del proyecto (leer, ver, contrastar) — sin crear ni usar copias,
     que en este escenario solo añaden latencia y datos viejos. Lecturas:
     libres. Escrituras (UPDATE/DELETE/DROP/ALTER): permitidas SOLO cuando el
     fix o el diagnóstico realmente las requieren, nunca de exploración — y
     con dos obligaciones: (a) **avisar SIEMPRE al usuario en la respuesta**
     qué se modificó exactamente (tabla, filas, sentencia) antes o al momento
     de hacerlo, jamás en silencio; y (b) documentar en el registro del port
     toda modificación estructural o de datos que deba PERSISTIR (ALTER,
     repairs de datos, drops), porque el vehículo oficial de cambios de
     esquema es el out.sql de la migración — una escritura directa que deba
     quedar en producción se anota como pendiente de SQL o se agrega al
     out.sql, no se deja solo como efecto colateral del diagnóstico.

**Recorridos obligatorios de la FASE 2.8 (cada uno con cookie jar y datos
reales de la BD temporal):**

- **A. Sesión cruzada por perfil**: login raíz → identidad leída desde página
  de pos Y de meseros (Regla 1). Luego login como cajero → cae en órdenes de
  pos con sesión viva; login como mesero → cae en venta de meseros.
- **B. Cobro completo**: abrir modal de pago → buscador de clientes responde
  (con y sin texto) → seleccionar cliente de ficha → correo prellenado →
  cobrar → factura con `sumas` correcto por tipo (COF y CCF) → DTE generada
  (respuesta MH simulada por contrato si no hay autorización de transmisión)
  → sello visible en el admin de facturas de pos tras la recarga. Además, con
  línea de tiempo (Regla 7): aviso del sello en pantalla y recarga con la
  venta cerrada; y una venta RECHAZADA por MH (p. ej. `codigoMsg 004`): aviso
  del rechazo y recarga igual (Regla 8). Anulación desde el admin de pos:
  JSON limpio del endpoint (Regla 10).
- **C. Reimpresión de orden finalizada**: texto generado con el nº de campos
  exacto del script de impresión destino (por cada formato de plataforma).
- **D. Endpoints de cada subdirectorio**: un GET/POST por endpoint tocado en
  `pos/`, `pos/procesos/`, `meseros/` — 200, JSON limpio sin notices (Regla 2:
  el cwd cambia el desenlace de los requires cruzados).
- **E. Generadores PDF por ambos caminos**: normativo (cobro/admin) y legacy
  (botones del admin) — PDF válido con logo, QR y marca de agua según estado;
  `output=file` responde JSON limpio. La marca de agua depende de un método de
  rotación (`Rotate`) de la clase del destino (`FactForm`/`PDFact`): verificar
  con grep que la clase lo define ANTES de dar el PDF por bueno, y probar con
  una factura ANULADA y una con evento de retorno (sin marca de agua la
  ausencia del método no salta).
- **F. Distritos en los 4+ frentes**: búsqueda por texto en cada formulario
  (agregar/editar por nivel), cascada inversa que rellena departamento y
  municipio SIN perderlos.
- **G. Evento de retorno**: visible SOLO en DTE recibidas COF/EXP/FAC no
  anuladas (MH no acepta retorno sobre CCF); volver al admin de origen tras
  generar/cancelar (referrer o fallback).
- **H. Correo por las 4 rutas**: cobro automático, botón del admin, reenvío y
  anulación — adjuntos con contenido (FASE 2.6 de la base) y destinatario
  resuelto desde el receptor del JSON.

## Registro del port (adiciones a la FASE 3 de la base)

El registro obliga lo que la FASE 3 de la base ya pide (qué se portó por
archivo y qué se dejó fuera con su commit, cadenas de llamadas verificadas,
correcciones de coherencia lógica de la FASE 2.5, matriz del correo de la
FASE 2.6, decisiones condicionales, equivalencias origen→destino, verificación
por comportamiento, veredicto de venta2, personalizaciones PDF del cierre) y
ADEMÁS, por ser fases propias de este formato:

- **El `MAPA_DESTINO.md` de la FASE 0** (o referencia a `AGENTS.md` si este lo
  cubre), citado como base de toda verificación de ubicación/esquema.
- **La tabla de referencias externas de la FASE 2.4** (referencia → dónde vive
  en el destino → estado), con los mapeos de columnas origen→destino.
- **El ledger de micro-desviaciones de la FASE 2.4** (desviación → por qué →
  verificación que la valida).
- **La sección "Smokes de ejecución" de la FASE 2.8** (recorrido → método →
  esperado vs obtenido → defecto/fix).

Y añade lo específico del formato:

- **Tabla de copias de helpers**: cada función/helper tocado con sus copias
  por nivel y qué variación recibió cada una.
- **Matriz de frentes del cobro**: página(s) de órdenes en uso × builders ×
  cambios aplicados (la prueba de "¿esté donde esté cobrando?" de la base).
- **Veredicto de residuos**: venta*.php de pos y páginas heredadas, con las
  evidencias del método de clasificación exhaustiva de la base.
- **Convención de sumas por tipo** aplicada al cobro y las reparaciones de
  datos históricas si hubo.
- **Contratos de impresión** descubiertos por plataforma cliente (los campos
  exactos de cada `explode`).
- **Aviso de re-login**: el despliegue invalida cookies (nombre de sesión
  unificado o nuevo) — avisarlo siempre en el informe final.

## Cierre del formato (checklist mínimo antes de dar el port por terminado)

1. Los tres niveles comparten cookie y los tres logins/redirecciones por
   perfil funcionan (recorrido A).
2. Un cobro real por COF y por CCF deja `factura` con los campos 2.0 correctos
   y `factura_pago` poblado (recorrido B).
3. El admin vivo de cada nivel muestra estado/sello/acciones nuevas vía SU
   endpoint SSP con th↔dt alineados (Regla 3).
4. La reimpresión imprime completa y ordenada (recorrido C).
5. Ningún endpoint de subdirectorio muere por requires cruzados (recorrido D).
6. Los DOS generadores PDF muestran logo/QR/marca de agua según estado
   (recorrido E).
7. El correo sale por las 4 rutas con adjunto íntegro (recorrido H).
8. Registro completo entregado en `tmp/` del destino, con los pendientes del
   usuario (aplicar SQL a la BD live si el deploy no lo hizo, permisos de los
   módulos nuevos, pruebas ambiente 00 con sus certificados).
9. Ningún callback `async` entregado a jQuery (Regla 7) y toda rama de los
   flujos JS termina en recarga o aviso (Regla 8).
10. La cadena de includes de las páginas de pos que cargan modelos/hub de raíz
    carga sin redeclaraciones ni notices (Regla 9, replay CLI).
11. Los endpoints JSON responden sin notices delante, y las funciones de tipo
    de documento aceptan alias y código MH (Regla 10).
12. En cada página con modales agregar+editar, alternarlos en ambos órdenes con
    clicks reales: selects despliegan y el checkbox alterna una vez (Regla 11),
    y ningún JS compartido usa `rootUrl` fuera de pos/ ni rutas relativas que
    cambien de sentido según la página host (Regla 12).

**Sobre las afirmaciones del registro**: "ya existe en el destino" escrito en
un registro previo no es evidencia (un `FactForm::Rotate()` declarado como
presente no existía). Antes de apoyarse en ella, grep.

**Entregables de BD**: junto al out.sql completo, el cliente de este formato
suele pedir además una **migración mínima** (`tmp/migracion_minima.sql`): solo
las tablas nuevas con uso real en el código del destino y las columnas nuevas
que ese código referencia, sin `id_server`/`unique_id` ni los datos de ejemplo
del origen, manteniendo corrección de catálogos MH y módulos nuevos. Regla
práctica de poda de columnas: el código previo al port corre contra la BD del
destino SIN esas columnas, así que cualquier mención a una columna "nueva" en
un archivo que el port no tocó es coincidencia de nombre (salvo tablas que
también se crean); lo que manda son las sentencias SQL que tocan esa tabla en
los archivos del port.
