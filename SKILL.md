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
  adienciones, sustituciones y recorridos específicos del formato definidos aquí.
  NO sirve para destinos sin subcarpetas de subsistema (usar la skill base).
---

# Migrar Normativa DTE 2.0 a un sistema cliente de formato restaurante

## Qué es esta skill y cómo se relaciona con la base

**Esta skill NO reemplaza a `migrar-normativa-dte-2.0`: la REQUIERE.** La skill
base define el método completo (principios, FASE 1 de BD con sync_schema y
catálogos MH, port selectivo por commit, auditorías de coherencia/correo/
símbolos, smokes de ejecución, registro). Ejecútala COMPLETA como fundamento;
esta skill define lo que CAMBIA en destinos de formato restaurante:

- **SUSTITUCIONES**: donde esta skill redefine una fase para el formato (la
  FASE 0 pasa de descubrimiento a verificación del mapa esperado; los recorridos
  de la FASE 2.8 son los del restaurante), lo definido AQUÍ manda.
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

La FASE 0 de la base se ejecuta igual (leer `AGENTS.md`/`CLAUDE.md`/READMEs
completos, crear `AGENTS.md` si falta, `tmp/MAPA_DESTINO.md` como artefacto),
pero con este formato el trabajo es **CONFIRMAR el mapa esperado**, no
descubrirlo. Checklist de verificación mínima antes de portear:

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
