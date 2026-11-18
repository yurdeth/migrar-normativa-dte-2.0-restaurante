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
