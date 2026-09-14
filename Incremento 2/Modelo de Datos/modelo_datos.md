# Modelo de Datos – Sistema de Gestión Minimarket y Panadería Huáscar

## 0. Decisión metodológica y convenciones

### 0.1 Fuente principal de la información

La identificación de entidades, atributos y restricciones se basa **principalmente en los Requerimientos Funcionales y No Funcionales (RF/RNF)**, complementados con los Casos de Uso Extendidos (CU).

**Justificación:**
1. Los RF contienen las reglas de negocio normativas (tipos de dato, formatos, rangos, unicidad, obligatoriedad). Estas son exactamente las restricciones que necesita un modelo de datos.
2. Los CU describen comportamiento y flujos, no estructura. Aportan validación cruzada y descubrimiento de entidades inferidas (sesiones, contadores de intentos de login, historiales de auditoría de pedidos, etc.) que los RF mencionan tangencialmente.
3. Por ello, cada entidad enlaza su trazabilidad principal al **RF/RNF** y secundaria al **CU** correspondiente.

### 0.2 Alcance

Se incluyen únicamente RF/RNF que **generan datos persistentes**. Se excluyen:
- Módulo 8 (Lector de Código de Barras): comportamiento de hardware, sin persistencia propia.
- RNF de tipo no estructural (RNF01–RNF07, RNF10–RNF21): no generan entidades.
- Sí se incluyen RNF08 (log de errores técnicos) y RNF09 (retención del log de auditoría) porque sí requieren persistencia.

### 0.3 Convenciones del documento

- Cada entidad se presenta con una **tabla markdown** de atributos.
- Cada entidad tiene secciones de **Relaciones** y **Trazabilidad (RF/CU)**.
- **PK:** Primary Key. **FK:** Foreign Key. **UNIQUE:** atributo no PK pero único.
- **Convención de PK:** toda entidad usa una clave primaria surrogada con la forma `<entidad>_id` (p. ej. `producto_id`, `categoria_id`, `trabajador_id`). Las claves naturales del negocio (EAN-13, RUT, número de venta) se mantienen como atributos `UNIQUE`.
- Las entidades marcadas como **[Inferida]** no aparecen literalmente en los RF pero son necesarias para soportar el comportamiento descrito (sesiones, intentos de login, líneas de detalle, etc.). Se indica de qué RF/CU se deriva.
- **Regla de relaciones M:N:** se modelan como **relaciones con atributos**, no como entidades. Su clave es **compuesta** y se forma a partir de las FK de las entidades adyacentes. No llevan surrogate `*_id` propio porque no tienen identidad independiente: existen sólo en tanto vinculan dos instancias. Si una M:N llegara a necesitar identidad propia (p. ej. para registrar múltiples ocurrencias del mismo par), se promovería a entidad con surrogate ID.
- **Regla de relaciones 1:N:** la FK se ubica en el lado N (la entidad dependiente).
- **Atributos vs. extensiones:** los conjuntos cerrados de valores (categorías, cargos) y las reglas fijas que dependen de ellos (qué categorías obligan vencimiento, qué cargos generan usuario) se modelan como **extensiones/restricciones**, no como atributos booleanos. Un atributo se justifica solo cuando el valor puede variar legítimamente entre instancias.
- **Especialización ISA (subtipos disjuntos) con flattening:** cuando una entidad tiene subtipos disjuntos con atributos específicos por subtipo (p. ej. Lote perecible vs. no perecible, Venta en efectivo vs. electrónica), se declaran como **subentidades dentro de la sección "Restricciones / extensiones"** del supertipo. En el nivel relacional se aplica **flattening en una sola tabla** con la convención:
  - `ES_<subtipo>`: discriminador booleano. En especialización disjunta, exactamente un `ES_*` es True por fila.
  - `<subtipo>_<atributo>`: atributo específico del subtipo, prefijado con el nombre del subtipo para indicar a qué subtipo pertenece. Solo está cargado cuando `ES_<subtipo> = True`; NULL en otro caso. Se usa un nombre semánticamente claro (no el nombre literal del subtipo) por legibilidad.
- Se adopta una visión conceptual relajada (no se exige 2FN/3FN). Algunos atributos pueden ser multivaluados o derivados, según el dominio.

### 0.4 Notación de cardinalidad

| Notación | Significado |
|---|---|
| 1..1 | Exactamente uno |
| 0..1 | Cero o uno (opcional) |
| 1..N | Uno o más |
| 0..N | Cero o más |

---

# MÓDULO 1 – GESTIÓN DE INVENTARIO

## 1.1 Entidad: `Producto`

**Trazabilidad:** RF01, RF02, RF03, RF04, RF06, RF07 · CU01, CU02, CU03, CU04, CU06, CU07

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| producto_id | Identificador | Único, autogenerado | Sí | **PK** |
| ean_13 | Numérico (13) | Único, exactamente 13 dígitos (clave de negocio) | Sí | UNIQUE |
| nombre | Texto | Máx. 100 caracteres, no vacío | Sí | |
| categoria_id | Identificador | Referencia a categoría existente | Sí | FK → Categoria.categoria_id |
| precio_costo | Entero | > 0, en pesos chilenos | Sí (al registrar lote / al modificar) | |
| precio_venta | Entero | > 0, > precio_costo, en pesos chilenos | Sí | |
| stock_minimo | Entero | ≥ 0 | Sí | |
| stock_actual | Entero (derivado) | ≥ 0; calculado como Σ cantidad_actual de sus lotes | Sí | |
| estado | Enum {Activo, Inactivo} | Por defecto "Activo" | Sí | |
| fecha_registro | Fecha-hora | Por defecto = ahora() | Sí | |

**Relaciones:**
- Producto **1..1 — 0..N** Lote (un producto tiene cero o más lotes; un lote pertenece a un único producto)
- Producto **1..1 — 0..N** Merma (un producto tiene cero o más mermas registradas)
- Producto **1..1 — 0..N** MovimientoInventario (un producto tiene historial de movimientos)
- Producto **1..1 — 0..N** DetalleVenta (referencia desde las líneas de venta)
- Producto **1..1 — 0..N** DetallePedido
- Producto **N..N — 0..N** PedidoProveedor (vía DetallePedido)

**Restricciones / extensiones:**
- No puede eliminarse (RF03) si existen registros de Venta, Merma o MovimientoInventario asociados. En tal caso debe desactivarse (cambio de estado a "Inactivo").
- En estado Inactivo no puede aparecer en módulos de Venta ni Merma, pero conserva su historial (RF04).
- En reportes para rol Reponedor y Cajero, el atributo `precio_costo` no debe exponerse (RF56).

---

## 1.2 Entidad: `Categoria`

**[Inferida]** – Derivada de la lista predefinida mencionada en RF01, RF06, RF13, RF19, RF44 (abarrotes, panadería, pastelería, carnicería, charcutería, bebidas, lácteos, limpieza). Se modela como entidad para soportar la relación M:N con Proveedor.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| categoria_id | Identificador | Único, autogenerado | Sí | **PK** |
| nombre_categoria | Texto | Único; pertenece al conjunto cerrado predefinido {abarrotes, panadería, pastelería, carnicería, charcutería, bebidas, lácteos, limpieza} (RF01) | Sí | UNIQUE |

**Relaciones:**
- Categoria **1..1 — 0..N** Producto
- Categoria **N..N — 0..N** Proveedor (vía ProveedorCategoria)

**Restricciones / extensiones:**
- Las categorías {panadería, pastelería, carnicería, charcutería, lácteos} obligan a que los `Lote` asociados a productos de esas categorías se clasifiquen como subtipo `Lote_Perecible` (ver especialización ISA en `Lote`), con `lote_perecible_fecha_vencimiento` no nula y posterior a la fecha actual al registrarse (RF05). Es una regla fija del dominio, no un atributo configurable por instancia.
- El conjunto de valores admitidos para `nombre_categoria` es cerrado y proviene de la lista predefinida en RF01. La incorporación de una nueva categoría requiere modificación del requerimiento, no es operación de usuario.

**Trazabilidad:** RF01, RF05, RF13, RF19, RF44 · CU01, CU05, CU13, CU19

---

## 1.3 Entidad: `Lote`

**Trazabilidad:** RF05, RF07, RF09, RF11, RF18, RF38, RF49 · CU05, CU07, CU09, CU11, CU18, CU38, CU49

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| lote_id | Identificador | Único, autogenerado | Sí | **PK** |
| producto_id | Numérico (13) | Producto debe existir | Sí | **FK** → Producto.producto_id |
| proveedor_id | Identificador | Referencia a proveedor existente | No (puede provenir de ajuste manual) | FK → Proveedor.proveedor_id |
| cantidad_inicial | Entero | > 0 | Sí | |
| cantidad_actual | Entero | ≥ 0 (no puede quedar negativa) | Sí | |
| precio_costo_lote | Entero | > 0, en pesos chilenos | Sí | |
| fecha_ingreso | Fecha-hora | Por defecto = ahora() | Sí | |
| pedido_origen_id | Identificador | Si el lote se generó por recepción de pedido | No | FK → PedidoProveedor.pedido_proveedor_id |
| estado | Enum {Activo, Agotado, Vencido} | Derivado | Sí | |
| ES_lote_perecible | Booleano | Discriminador ISA: True si la categoría del producto exige vencimiento | Sí | |
| lote_perecible_fecha_vencimiento | Fecha | Obligatoria y > fecha actual al registrar, **sólo si** `ES_lote_perecible = True`; NULL en caso contrario | Condicional | |
| ES_lote_no_perecible | Booleano | Discriminador ISA: True para categorías que no exigen vencimiento | Sí | |

**Relaciones:**
- Lote **N..1 — 1..1** Producto
- Lote **N..1 — 0..1** Proveedor
- Lote **1..1 — 0..N** MovimientoInventario (los movimientos de tipo ajuste/entrada/venta/merma referencian al lote afectado)
- Lote **N..1 — 0..1** PedidoProveedor (si se originó de la recepción de un pedido)

**Restricciones / extensiones:**

**Especialización ISA disjunta y total de `Lote`** (mapping plano en la misma tabla):
- Subentidad **`Lote_Perecible`**: aplica cuando el `Producto` asociado pertenece a una categoría que exige vencimiento según RF05 (panadería, pastelería, carnicería, charcutería, lácteos). Atributo específico: `lote_perecible_fecha_vencimiento`.
- Subentidad **`Lote_No_Perecible`**: el resto de categorías (abarrotes, bebidas, limpieza). Sin atributos específicos.
- Invariante: por cada fila, exactamente uno de `{ES_lote_perecible, ES_lote_no_perecible}` es True. El valor del discriminador se determina al registrar el lote a partir de la categoría del producto.

**Otras restricciones:**
- El stock se descuenta por **FEFO** (First Expired First Out) al registrar Venta o Merma, ordenando los `Lote_Perecible` por `lote_perecible_fecha_vencimiento` ascendente y los `Lote_No_Perecible` por `fecha_ingreso` ascendente (RF10, RF38).
- Un `Lote_Perecible` con `lote_perecible_fecha_vencimiento` ≤ hoy + 7 días gatilla alerta en dashboard (RF09).
- Un `Lote_Perecible` con `lote_perecible_fecha_vencimiento` < hoy se considera "Vencido" y se muestra en sección separada del dashboard (RF09).

---

## 1.4 Entidad: `Merma`

**Trazabilidad:** RF10, RF48 · CU10, CU48

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| merma_id | Identificador | Único, autogenerado | Sí | **PK** |
| producto_id | Numérico (13) | Producto debe estar Activo | Sí | **FK** → Producto.producto_id |
| cantidad | Entero | > 0, ≤ stock disponible al momento del registro | Sí | |
| motivo | Enum {Vencimiento, Daño, Robo, Error de registro} | Lista predefinida | Sí | |
| observacion | Texto | Máx. 200 caracteres | No | |
| fecha_hora | Fecha-hora | = ahora() al registrar | Sí | |
| usuario_id | Identificador | Usuario responsable (en sesión) | Sí | **FK** → Usuario.usuario_id |
| lotes_descontados | Multivaluado (lista) | Conjunto de pares {lote_id, cantidad_descontada} resultado de FEFO | Sí | |

**Relaciones:**
- Merma **N..1 — 1..1** Producto
- Merma **N..1 — 1..1** Usuario
- Merma **1..1 — 1..N** Lote (a través del descuento FEFO; un registro de merma puede afectar uno o varios lotes)

**Restricciones / extensiones:**
- Cada Merma genera uno o varios `MovimientoInventario` de tipo "Merma" (uno por lote afectado).

---

## 1.5 Entidad: `MovimientoInventario`

**Trazabilidad:** RF11, RF12, RF38, RF52 · CU11, CU12, CU38, CU52

Es el log unificado de cambios de stock. Cada operación (entrada por recepción, venta, merma, ajuste) genera uno o más registros aquí.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| movimiento_inventario_id | Identificador | Único, autogenerado | Sí | **PK** |
| producto_id | Numérico (13) | | Sí | **FK** → Producto.producto_id |
| lote_id | Identificador | Aplica para tipos Ajuste y Entrada; opcional para Venta/Merma (donde se puede referenciar el lote afectado por línea) | Condicional | FK → Lote.lote_id |
| tipo | Enum {Entrada, Venta, Merma, Ajuste} | | Sí | |
| cantidad | Entero | Positiva para Entrada y Ajuste-incremento; negativa para Venta, Merma y Ajuste-decremento | Sí | |
| stock_resultante | Entero | Stock total del producto luego del movimiento | Sí | |
| fecha_hora | Fecha-hora | DD/MM/AAAA HH:MM | Sí | |
| usuario_id | Identificador | Usuario responsable | Sí | **FK** → Usuario.usuario_id |
| referencia_origen | Identificador | venta_id / merma_id / pedido_proveedor_id según corresponda | No | FK polimórfica |
| ES_movimiento_ajuste | Booleano | Discriminador ISA: True si tipo = Ajuste | Sí | |
| movimiento_ajuste_justificacion | Texto | 10..300 caracteres, **sólo si** `ES_movimiento_ajuste = True`; NULL en caso contrario | Condicional | |
| ES_movimiento_no_ajuste | Booleano | Discriminador ISA: True si tipo ∈ {Entrada, Venta, Merma} | Sí | |

**Relaciones:**
- MovimientoInventario **N..1 — 1..1** Producto
- MovimientoInventario **N..1 — 0..1** Lote
- MovimientoInventario **N..1 — 1..1** Usuario

**Restricciones / extensiones:**

**Especialización ISA disjunta y total de `MovimientoInventario`** (mapping plano en la misma tabla):
- Subentidad **`Movimiento_Ajuste`**: aplica cuando `tipo = Ajuste` (RF11, CU11). Atributo específico: `movimiento_ajuste_justificacion` (obligatoria, 10..300 caracteres). El ajuste no puede dejar la cantidad del lote ni el stock total en negativo.
- Subentidad **`Movimiento_No_Ajuste`**: aplica cuando `tipo ∈ {Entrada, Venta, Merma}`. Sin justificación; la trazabilidad se obtiene vía `referencia_origen`.
- Invariante: por cada fila, exactamente uno de `{ES_movimiento_ajuste, ES_movimiento_no_ajuste}` es True, consistente con `tipo`.

**Otras restricciones:**
- Es el origen de los reportes RF12 (historial por producto) y RF52 (auditoría general).

---

# MÓDULO 2 – GESTIÓN DE PROVEEDORES

## 2.1 Entidad: `Proveedor`

**Trazabilidad:** RF13, RF14, RF15, RF17 · CU13, CU14, CU15, CU17

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| proveedor_id | Identificador | Único, autogenerado | Sí | **PK** |
| rut | Texto | Formato XX.XXX.XXX-X, único (clave de negocio, no editable tras registro RF14) | Sí | UNIQUE |
| nombre_razon_social | Texto | Máx. 100 caracteres | Sí | |
| nombre_contacto | Texto | Máx. 80 caracteres | Sí | |
| telefono | Numérico | Exactamente 9 dígitos | Sí | |
| correo_electronico | Texto | Formato email válido | Sí | |

**Relaciones:**
- Proveedor **N..N — 1..N** Categoria (vía `ProveedorCategoria`)
- Proveedor **1..1 — 0..N** PedidoProveedor
- Proveedor **1..1 — 0..N** Lote (los lotes referencian al proveedor del que provienen)

**Restricciones / extensiones:**
- El campo `rut` es no editable tras el registro (RF14).
- Debe suministrar al menos una categoría (selección múltiple, RF13).

---

## 2.2 Relación: `ProveedorCategoria` (N:M entre Proveedor y Categoria)

Relación con clave compuesta, sin atributos propios. No es entidad: existe sólo en tanto vincula un proveedor con una categoría. Materializa la "selección múltiple" mencionada en RF13.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| proveedor_id | Identificador | | Sí | **PK + FK** → Proveedor.proveedor_id |
| categoria_id | Identificador | | Sí | **PK + FK** → Categoria.categoria_id |

**Clave primaria compuesta:** (proveedor_id, categoria_id).

**Trazabilidad:** RF13, RF15 · CU13, CU15

---

## 2.3 Entidad: `PedidoProveedor`

**Trazabilidad:** RF17, RF18 · CU17, CU18, CU18b

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| pedido_proveedor_id | Identificador | Único, autogenerado | Sí | **PK** |
| proveedor_id | Identificador | | Sí | **FK** → Proveedor.proveedor_id |
| fecha_emision | Fecha-hora | = ahora() al crear | Sí | |
| estado | Enum {Pendiente, Recibido, Recibido parcialmente, Cancelado} | Por defecto "Pendiente" | Sí | |
| fecha_recepcion | Fecha-hora | Se completa al confirmar recepción | No | |
| usuario_emisor_id | Identificador | Usuario que registró el pedido | Sí | **FK** → Usuario.usuario_id |
| usuario_receptor_id | Identificador | Usuario que confirmó la recepción/cancelación | No | **FK** → Usuario.usuario_id |
| nota_recepcion | Texto | Texto libre, opcional, registrado al confirmar | No | |

**Relaciones:**
- PedidoProveedor **N..1 — 1..1** Proveedor
- PedidoProveedor **1..1 — 1..N** DetallePedido
- PedidoProveedor **1..1 — 0..N** Lote (al confirmar recepción se generan lotes)
- PedidoProveedor **1..1 — 1..N** HistorialAuditoriaPedido

**Restricciones / extensiones:**
- Un mismo producto no puede aparecer más de una vez en el mismo pedido (RF17).
- Si estado = "Cancelado" no se generan lotes ni movimientos (RF18, CU18b).

---

## 2.4 Relación: `DetallePedido` (N:M entre PedidoProveedor y Producto)

Relación con atributos. Línea de pedido: materializa "uno o más productos, cada uno con cantidad solicitada" (RF17). PK compuesta sin surrogate ID propio.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| pedido_proveedor_id | Identificador | | Sí | **PK + FK** → PedidoProveedor.pedido_proveedor_id |
| producto_id | Numérico (13) | | Sí | **PK + FK** → Producto.producto_id |
| cantidad_solicitada | Entero | > 0 | Sí | |
| cantidad_recibida | Entero | ≥ 0, ≤ cantidad_solicitada en recepción parcial; = cantidad_solicitada en recepción total; 0 si cancelado | Condicional | |

**Clave primaria compuesta:** (pedido_proveedor_id, producto_id).

**Trazabilidad:** RF17, RF18 · CU17, CU18

---

## 2.5 Entidad: `HistorialAuditoriaPedido`

**[Inferida]** – RF18 menciona "queda registrada como parte del historial de auditoría del pedido"; CU18b confirma que la cancelación se registra con usuario, fecha y hora.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| historial_auditoria_pedido_id | Identificador | Único, autogenerado | Sí | **PK** |
| pedido_proveedor_id | Identificador | | Sí | **FK** → PedidoProveedor.pedido_proveedor_id |
| tipo_evento | Enum {Creación, Recepción total, Recepción parcial, Cancelación} | | Sí | |
| fecha_hora | Fecha-hora | = ahora() | Sí | |
| usuario_id | Identificador | | Sí | **FK** → Usuario.usuario_id |
| nota | Texto | Texto libre | No | |

**Relaciones:**
- HistorialAuditoriaPedido **N..1 — 1..1** PedidoProveedor
- HistorialAuditoriaPedido **N..1 — 1..1** Usuario

**Trazabilidad:** RF18 · CU18, CU18b

---

# MÓDULO 3 – VALORIZACIÓN Y EXPORTACIÓN DE INVENTARIO

Este módulo (RF19, RF20) **no introduce entidades persistentes nuevas**: se compone de consultas calculadas (RF19: valorización) y exportaciones (RF20). La información requerida se obtiene de `Producto`, `Lote` y `Categoria`. Trazabilidad informativa:

- RF19 · CU19: cálculo de valor de inventario sobre `Producto.stock_actual × Producto.precio_costo`, agrupable por `Categoria`.
- RF20 · CU20: exportación de listado de productos activos. La exportación se rastrea en `LogAuditoria` (acción "exportación de reportes", RF57).

---

# MÓDULO 4 – GESTIÓN DE TRABAJADORES Y ASISTENCIA

## 4.1 Entidad: `Trabajador`

**Trazabilidad:** RF21, RF22, RF23, RF24 · CU21, CU21b, CU22, CU23, CU24

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| trabajador_id | Identificador | Único, autogenerado | Sí | **PK** |
| rut | Texto | Formato XX.XXX.XXX-X, único (clave de negocio, no editable tras registro RF22) | Sí | UNIQUE |
| nombre_completo | Texto | Máx. 100 caracteres | Sí | |
| cargo_id | Identificador | Referencia a cargo existente | Sí | FK → Cargo.cargo_id |
| ES_trabajador_con_usuario | Booleano | Discriminador ISA: True si cargo ∈ {dueño, cajero, reponedor} | Sí | |
| ES_trabajador_operativo | Booleano | Discriminador ISA: True si cargo ∈ {panadero, carnicero, pastelero, charcutero} | Sí | |
| telefono | Numérico | Exactamente 9 dígitos | Sí | |
| correo_electronico | Texto | Formato email válido si se ingresa | No | |
| fecha_ingreso | Fecha | Por defecto = fecha de registro | Sí | |
| estado | Enum {Activo, Inactivo} | Por defecto "Activo" | Sí | |

**Relaciones:**
- Trabajador **1..1 — 0..1** Usuario (los cargos operativos no generan usuario; RF21, CU21b)
- Trabajador **1..1 — 0..N** Turno
- Trabajador **1..1 — 0..N** Asistencia
- Trabajador **1..1 — 0..N** Ausencia
- Trabajador **1..1 — 0..N** Remuneracion
- Trabajador **N..1 — 1..1** Cargo

**Restricciones / extensiones:**

**Especialización ISA disjunta y total de `Trabajador`** (mapping plano en la misma tabla):
- Subentidad **`Trabajador_Con_Usuario`**: aplica cuando el cargo ∈ {dueño, cajero, reponedor}. Cada instancia tiene exactamente una fila asociada en `Usuario` (relación 1:1; la FK reside en `Usuario.trabajador_id`). No hay atributos escalares específicos en el supertipo: lo que distingue al subtipo es la **existencia** del Usuario vinculado.
- Subentidad **`Trabajador_Operativo`**: aplica cuando el cargo ∈ {panadero, carnicero, pastelero, charcutero}. No tiene Usuario asociado (no participa en autenticación). Sin atributos específicos.
- Invariante: por cada fila, exactamente uno de `{ES_trabajador_con_usuario, ES_trabajador_operativo}` es True, consistente con el cargo. Además: `ES_trabajador_con_usuario = True ⟺ existe Usuario con trabajador_id = este.trabajador_id`.

**Otras restricciones:**
- El campo `rut` es no editable tras el registro (RF22).
- Un trabajador Inactivo no aparece para asignación de turnos ni asistencia, pero conserva su historial (RF23).

---

## 4.2 Entidad: `Cargo`

**[Inferida]** – Catálogo derivado de la lista predefinida en RF21: dueño, cajero, reponedor, panadero, carnicero, pastelero, charcutero.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| cargo_id | Identificador | Único, autogenerado | Sí | **PK** |
| nombre_cargo | Texto | Único; pertenece al conjunto cerrado {dueño, cajero, reponedor, panadero, carnicero, pastelero, charcutero} (RF21) | Sí | UNIQUE |

**Relaciones:**
- Cargo **1..1 — 0..N** Trabajador

**Restricciones / extensiones:**
- Solo los cargos {dueño, cajero, reponedor} generan un registro en `Usuario` al crear el Trabajador; los cargos operativos {panadero, carnicero, pastelero, charcutero} no tienen acceso al sistema (RF21, CU21b). Es una regla fija del dominio.
- El rol del `Usuario` asociado coincide con el `nombre_cargo` para los cargos con acceso: dueño→dueño, cajero→cajero, reponedor→reponedor (RF21, RF56). Es un mapeo fijo, no configurable por instancia.
- El conjunto de valores admitidos para `nombre_cargo` es cerrado y proviene de RF21. Su modificación requiere cambio de requerimiento.

**Trazabilidad:** RF21, RF56 · CU21, CU21b

---

## 4.3 Entidad: `Usuario`

**[Inferida]** – No mencionada como entidad explícita pero presupuesta por RF21, RF55, RF56, RF57, RF58. Se separa de Trabajador porque hay trabajadores sin acceso al sistema.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| usuario_id | Texto | Generado a partir del RUT del trabajador, único, sin espacios, máx. 50 chars | Sí | **PK** |
| trabajador_id | Identificador | El trabajador asociado | Sí | **FK** → Trabajador.trabajador_id (UNIQUE; 1:0..1) |
| password_hash | Texto | Hash con sal (bcrypt o equivalente; RNF04) | Sí | |
| rol | Enum {dueño, cajero, reponedor} | Definido por el cargo del trabajador | Sí | |
| requiere_cambio_password | Booleano | True tras generar contraseña temporal | Sí | |
| password_temporal_actual_id | Identificador | Última contraseña temporal vigente | No | FK → ContrasenaTemporal.contrasena_temporal_id |
| fecha_creacion | Fecha-hora | | Sí | |
| ultimo_login | Fecha-hora | Actualizado en cada login exitoso | No | |

**Relaciones:**
- Usuario **1..1 — 1..1** Trabajador
- Usuario **1..1 — 0..N** SesionUsuario
- Usuario **1..1 — 0..N** IntentoLogin
- Usuario **1..1 — 0..N** ContrasenaTemporal
- Usuario **1..1 — 0..N** Venta (como cajero responsable)
- Usuario **1..1 — 0..N** MovimientoInventario (como responsable)
- Usuario **1..1 — 0..N** LogAuditoria

**Restricciones / extensiones:**
- Contraseña: mínimo 8 caracteres, al menos una mayúscula, una minúscula y un número (RF55).
- Tras 5 intentos fallidos consecutivos la cuenta se bloquea 15 minutos (RF55).
- Cierre automático de sesión tras 30 minutos de inactividad (RF55).

---

## 4.4 Entidad: `ContrasenaTemporal`

**[Inferida]** – RF21, RF22, RF58 mencionan generación de "contraseña temporal de 8 caracteres alfanuméricos … mostrada una única vez … validez 24 horas (RF58)".

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| contrasena_temporal_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | | Sí | **FK** → Usuario.usuario_id |
| hash_temporal | Texto | Hash con sal | Sí | |
| fecha_generacion | Fecha-hora | | Sí | |
| fecha_expiracion | Fecha-hora | fecha_generacion + 24 h (RF58) | Sí | |
| estado | Enum {Vigente, Usada, Expirada, Invalidada} | | Sí | |
| generada_por_usuario_id | Texto | Dueño que la generó | Sí | **FK** → Usuario.usuario_id |

**Trazabilidad:** RF21, RF22, RF58 · CU21, CU22, CU58, CU55b

---

## 4.5 Entidad: `SesionUsuario`

**[Inferida]** – RF55 ("sesión se cerrará automáticamente tras 30 minutos de inactividad"). RNF18–RNF20 (sync engine) presuponen sesiones rastreables.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| sesion_usuario_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | | Sí | **FK** → Usuario.usuario_id |
| fecha_inicio | Fecha-hora | | Sí | |
| fecha_ultimo_acceso | Fecha-hora | Actualizado en cada interacción | Sí | |
| fecha_cierre | Fecha-hora | | No | |
| motivo_cierre | Enum {Logout, Inactividad, Bloqueo} | | No | |
| ip_origen | Texto | | No | |

**Trazabilidad:** RF55 · CU55

---

## 4.6 Entidad: `IntentoLogin`

**[Inferida]** – RF55 menciona contador de "5 intentos fallidos consecutivos" y bloqueo temporal por 15 minutos.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| intento_login_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | Puede ser nulo si el usuario ingresado no existe | No | **FK** → Usuario.usuario_id |
| nombre_usuario_intentado | Texto | Lo que efectivamente se tipeó | Sí | |
| fecha_hora | Fecha-hora | | Sí | |
| exitoso | Booleano | | Sí | |
| ip_origen | Texto | | No | |

**Restricciones / extensiones:**
- El sistema mantiene un contador derivado de intentos fallidos consecutivos por `usuario_id`. Tras 5 fallidos consecutivos se gatilla bloqueo por 15 minutos. El contador se reinicia tras un login exitoso o tras transcurridos los 15 minutos (RF55).

**Trazabilidad:** RF55 · CU55

---

## 4.7 Entidad: `Turno`

**Trazabilidad:** RF25, RF26, RF27, RF28 · CU25, CU26, CU27, CU28

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| turno_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | Trabajador Activo | Sí | **FK** → Trabajador.trabajador_id |
| fecha | Fecha | DD/MM/AAAA | Sí | |
| hora_inicio | Hora | HH:MM (24h) | Sí | |
| hora_termino | Hora | HH:MM, > hora_inicio | Sí | |
| estado | Enum {Pendiente, Ejecutado, Eliminado} | Derivado de fecha vs. hoy y existencia de asistencia | Sí | |

**Relaciones:**
- Turno **N..1 — 1..1** Trabajador
- Turno **1..1 — 0..1** Asistencia (un turno ejecutado tiene típicamente una asistencia asociada en la fecha)

**Restricciones / extensiones:**
- No puede existir otro turno del mismo trabajador en el mismo día con rango horario solapado (RF25).
- Solo puede modificarse o eliminarse si la fecha es posterior a la fecha actual y no tiene asistencia registrada (RF26, RF27).

---

## 4.8 Entidad: `Asistencia`

**Trazabilidad:** RF29, RF30, RF31, RF33, RF51 · CU29, CU29b, CU30, CU31, CU33, CU51

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| asistencia_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | | Sí | **FK** → Trabajador.trabajador_id |
| fecha | Fecha | | Sí | |
| hora_entrada | Hora | HH:MM (registrada automáticamente) | Sí | |
| hora_salida | Hora | HH:MM; null mientras no se registra | No | |
| horas_trabajadas | Hora-derivada | HH:MM = hora_salida – hora_entrada; "Pendiente" si null | Sí | |
| sin_turno_asignado | Booleano | True si se registró entrada sin turno para ese día (CU29b) | Sí | |
| turno_id | Identificador | Si existe turno asignado para esa fecha | No | FK → Turno.turno_id |

**Restricciones / extensiones:**
- Único por (trabajador_id, fecha): no puede haber dos asistencias en el mismo día (RF29, RF30).
- No puede registrarse salida sin entrada previa en el mismo día (RF30).
- Si un trabajador tiene Ausencia en la misma fecha, no puede registrar Asistencia (y viceversa, RF32).

---

## 4.9 Entidad: `Ausencia`

**Trazabilidad:** RF32, RF33, RF51 · CU32, CU33, CU51

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| ausencia_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | | Sí | **FK** → Trabajador.trabajador_id |
| fecha | Fecha | DD/MM/AAAA, no futura | Sí | |
| tipo | Enum {Justificada, Injustificada} | | Sí | |
| observacion | Texto | Máx. 200 caracteres | No | |
| usuario_registrador_id | Texto | Dueño que registró | Sí | **FK** → Usuario.usuario_id |
| fecha_registro | Fecha-hora | | Sí | |

**Restricciones / extensiones:**
- Único por (trabajador_id, fecha).
- No puede coexistir con Asistencia para el mismo (trabajador_id, fecha) (RF32).

---

## 4.10 Entidad: `Remuneracion`

**Trazabilidad:** RF34 · CU34

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| remuneracion_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | | Sí | **FK** → Trabajador.trabajador_id |
| mes | Entero | 1..12 | Sí | |
| anio | Entero | AAAA | Sí | |
| monto_bruto | Entero | > 0, en pesos chilenos | Sí | |
| pct_afp_aplicado | Decimal | Snapshot del % AFP al momento del cálculo | Sí | |
| pct_salud_aplicado | Decimal | Snapshot del % Salud | Sí | |
| pct_cesantia_aplicado | Decimal | Snapshot del % Seguro Cesantía | Sí | |
| monto_liquido | Entero | = bruto – descuentos; ≥ 0 | Sí | |
| observacion | Texto | Máx. 200 caracteres | No | |
| fecha_registro | Fecha-hora | | Sí | |
| usuario_registrador_id | Texto | | Sí | **FK** → Usuario.usuario_id |

**Restricciones / extensiones:**
- Único por (trabajador_id, mes, anio): un trabajador no puede tener dos remuneraciones del mismo período (RF34).
- Los porcentajes por defecto se leen desde `ConfiguracionDescuentos`, pero se guardan como snapshot en cada `Remuneracion` para preservar histórico.

---

## 4.11 Entidad: `ConfiguracionDescuentos`

**[Inferida]** – RF34: "los porcentajes de descuento previsional deben ser configurables por el dueño desde el módulo de configuración del sistema, con valores por defecto AFP 11,5%, Salud 7%, Cesantía 0,6%".

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| configuracion_descuentos_id | Identificador | Único | Sí | **PK** |
| pct_afp | Decimal | Por defecto 11,5 | Sí | |
| pct_salud | Decimal | Por defecto 7,0 | Sí | |
| pct_cesantia | Decimal | Por defecto 0,6 | Sí | |
| vigente_desde | Fecha-hora | | Sí | |
| vigente_hasta | Fecha-hora | Null = configuración actual | No | |
| usuario_modificador_id | Texto | | Sí | **FK** → Usuario.usuario_id |

**Trazabilidad:** RF34 · CU34

---

# MÓDULO 5 – GESTIÓN DE VENTAS

## 5.1 Entidad: `Venta`

**Trazabilidad:** RF35, RF36, RF37, RF40, RF42, RF43, RF44 · CU35, CU35b, CU35c, CU36, CU37, CU40, CU42, CU43, CU44

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| venta_id | Identificador | Único, autogenerado correlativo | Sí | **PK** |
| usuario_cajero_id | Texto | Usuario en sesión al confirmar | Sí | **FK** → Usuario.usuario_id |
| fecha_hora | Fecha-hora | = ahora() al confirmar | Sí | |
| subtotal | Entero | ≥ 0 | Sí | |
| descuento_tipo | Enum {Ninguno, Porcentaje, MontoFijo} | | Sí | |
| descuento_valor | Entero o Decimal | Porcentaje 1..100 o monto > 0 y < subtotal | Condicional | |
| descuento_razon | Texto | Máx. 100 caracteres, obligatoria si hay descuento | Condicional | |
| total | Entero | = subtotal – descuento | Sí | |
| metodo_pago | Enum {Efectivo, Débito, Crédito, Transferencia} | | Sí | |
| ES_venta_efectivo | Booleano | Discriminador ISA: True si metodo_pago = Efectivo | Sí | |
| venta_efectivo_monto_recibido | Entero | ≥ total, **sólo si** `ES_venta_efectivo = True`; NULL en caso contrario | Condicional | |
| venta_efectivo_vuelto | Entero | = venta_efectivo_monto_recibido – total, **sólo si** `ES_venta_efectivo = True`; NULL en caso contrario | Condicional | |
| ES_venta_electronica | Booleano | Discriminador ISA: True si metodo_pago ∈ {Débito, Crédito, Transferencia} | Sí | |
| estado | Enum {Confirmada, Anulada} | Por defecto "Confirmada" | Sí | |
| cierre_caja_id | Identificador | Día/caja a la que pertenece la venta | Sí | **FK** → CierreCaja.cierre_caja_id |

**Relaciones:**
- Venta **N..1 — 1..1** Usuario (cajero responsable)
- Venta **1..1 — 1..N** DetalleVenta
- Venta **1..1 — 0..1** AnulacionVenta
- Venta **N..1 — 1..1** CierreCaja (día operativo)

**Restricciones / extensiones:**

**Especialización ISA disjunta y total de `Venta`** (mapping plano en la misma tabla):
- Subentidad **`Venta_Efectivo`**: aplica cuando `metodo_pago = Efectivo` (RF35, CU35b). Atributos específicos: `venta_efectivo_monto_recibido`, `venta_efectivo_vuelto`. El sistema solicita el monto recibido (≥ total) y calcula el vuelto automáticamente.
- Subentidad **`Venta_Electronica`**: aplica cuando `metodo_pago ∈ {Débito, Crédito, Transferencia}` (CU35c). Sin atributos específicos: no se solicita monto recibido ni se calcula vuelto.
- Invariante: por cada fila, exactamente uno de `{ES_venta_efectivo, ES_venta_electronica}` es True, y debe ser consistente con `metodo_pago`.

**Otras restricciones:**
- El cajero responsable se asocia automáticamente desde la sesión y es inmutable (RF42).
- No se permite registrar ventas en un día cuyo `CierreCaja` ya esté en estado "Cerrado" (RF41).
- Al confirmar, gatilla descuento de stock FEFO en `Lote` y crea registros en `MovimientoInventario` (RF38).

---

## 5.2 Relación: `DetalleVenta` (N:M entre Venta y Producto)

Relación con atributos. Línea de venta: materializa "lista de productos con cantidad y precio unitario, subtotal por producto" (RF35). PK compuesta sin surrogate ID propio. Categoría queda **snapshoteada** aquí (RF44).

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| venta_id | Identificador | | Sí | **PK + FK** → Venta.venta_id |
| producto_id | Numérico (13) | | Sí | **PK + FK** → Producto.producto_id |
| cantidad | Entero | > 0 | Sí | |
| precio_unitario | Entero | Snapshot del precio_venta al momento de la venta | Sí | |
| subtotal_linea | Entero | = cantidad × precio_unitario | Sí | |
| categoria_snapshot | Texto | Snapshot del nombre de la categoría al vender (RF44) | Sí | |
| lotes_descontados | Multivaluado | Conjunto {lote_id, cantidad} resultado de FEFO | Sí | |

**Clave primaria compuesta:** (venta_id, producto_id).

**Restricciones / extensiones:**
- Un mismo producto no puede aparecer más de una vez por venta (debe consolidarse cantidad).
- `lotes_descontados` permite revertir el stock con precisión al anular (RF37).

**Trazabilidad:** RF35, RF38, RF44 · CU35, CU38, CU44

---

## 5.3 Entidad: `AnulacionVenta`

**[Inferida]** – RF37 menciona "registrar la anulación con usuario responsable, fecha y hora".

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| anulacion_venta_id | Identificador | Único | Sí | **PK** |
| venta_id | Identificador | Venta a anular, debe estar Confirmada | Sí | **FK** UNIQUE → Venta.venta_id |
| usuario_id | Texto | Cajero (solo ventas del día) o Dueño (cualquier día) | Sí | **FK** → Usuario.usuario_id |
| fecha_hora | Fecha-hora | = ahora() | Sí | |
| razon | Texto | 10..200 caracteres | Sí | |

**Relaciones:**
- AnulacionVenta **1..1 — 1..1** Venta (0..1 desde la perspectiva de Venta)

**Restricciones / extensiones:**
- Al confirmar la anulación se revierte el stock de cada producto involucrado y se generan movimientos compensatorios en `MovimientoInventario` (RF37).
- Solo el rol Dueño puede anular ventas de días anteriores (RF37, RF56).

**Trazabilidad:** RF37 · CU37

---

## 5.4 Entidad: `CierreCaja`

**Trazabilidad:** RF41 · CU41

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| cierre_caja_id | Identificador | Único | Sí | **PK** |
| fecha | Fecha | Única por día | Sí | |
| estado | Enum {Abierta, Cerrada} | Por defecto "Abierta" al primer movimiento del día | Sí | |
| fecha_hora_cierre | Fecha-hora | Se completa al cerrar | No | |
| usuario_cierre_id | Texto | Usuario (Dueño o Cajero) que cerró | No | **FK** → Usuario.usuario_id |
| total_ventas | Entero | Snapshot del monto total vendido en el día (calculado al cerrar) | Sí | |
| total_efectivo | Entero | Snapshot | Sí | |
| total_debito | Entero | Snapshot | Sí | |
| total_credito | Entero | Snapshot | Sí | |
| total_transferencia | Entero | Snapshot | Sí | |
| cantidad_anulaciones | Entero | Snapshot | Sí | |
| monto_anulaciones | Entero | Snapshot | Sí | |

**Relaciones:**
- CierreCaja **1..1 — 0..N** Venta

**Restricciones / extensiones:**
- Único por `fecha`.
- Mientras `estado = Cerrada`, no se admiten nuevas ventas para esa fecha (RF41).

---

# MÓDULO 6 – REPORTES Y EXPORTACIÓN

Los RF45–RF53 (reportes diarios, mensuales, productos más vendidos, mermas, lotes por vencer, rentabilidad, asistencia, movimientos, exportación) **no requieren nuevas entidades persistentes**: son **consultas/derivaciones** sobre las entidades existentes:

| RF | CU | Fuentes de datos |
|---|---|---|
| RF45 (reporte diario) | CU45 | Venta, DetalleVenta, CierreCaja |
| RF46 (reporte mensual) | CU46 | Venta, DetalleVenta |
| RF47 (productos más vendidos) | CU47 | Venta, DetalleVenta, Producto |
| RF48 (mermas) | CU48 | Merma, Producto |
| RF49 (lotes por vencer) | CU49 | Lote, Producto, Categoria, Proveedor |
| RF50 (rentabilidad por categoría) | CU50 | Venta, DetalleVenta, Lote, Categoria |
| RF51 (asistencia) | CU51 | Trabajador, Asistencia, Ausencia, Cargo |
| RF52 (auditoría de movimientos) | CU52 | MovimientoInventario, Producto, Usuario |
| RF53 (exportación de reportes) | CU53 | (orquestador) — la acción se registra en LogAuditoria |

> Toda exportación de reportes queda registrada como evento en `LogAuditoria` con tipo de acción "Exportación de reportes" (RF57).

---

# MÓDULO 7 – DASHBOARD Y CONTROL DE ACCESO

El RF54 (Dashboard) y RF56 (Control de acceso por rol) son **vistas/políticas** sobre las entidades ya definidas; no requieren entidades nuevas. Las entidades de soporte (Usuario, SesionUsuario, IntentoLogin, ContrasenaTemporal) ya fueron descritas en el Módulo 4 (Trabajadores y Asistencia) por ser dependientes de Trabajador.

A continuación se describen las entidades **propias** del módulo: `LogAuditoria` y `LogErroresTecnicos`.

## 7.1 Entidad: `LogAuditoria`

**Trazabilidad:** RF57, RNF09 · CU57, CU62

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| log_auditoria_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | Usuario que ejecutó la acción | Sí | **FK** → Usuario.usuario_id |
| nombre_usuario_snapshot | Texto | Snapshot al momento de la acción | Sí | |
| rol_snapshot | Enum {dueño, cajero, reponedor} | Snapshot al momento de la acción | Sí | |
| fecha_hora | Fecha-hora | | Sí | |
| tipo_accion | Enum {Inicio sesión, Cierre sesión, Registro, Edición, Eliminación, Anulación, Exportación reportes, Consulta rentabilidad, Consulta remuneraciones} | RF57 | Sí | |
| modulo | Texto | Módulo donde ocurrió | Sí | |
| descripcion | Texto | Descripción de la acción | Sí | |
| ip_origen | Texto | | No | |

**Restricciones / extensiones:**
- Retención mínima 12 meses (RF57, RNF09). Pasado ese período pueden eliminarse automáticamente.
- Los registros son inmutables: ningún usuario, incluyendo el rol Dueño, puede modificarlos ni eliminarlos manualmente (RNF09, CU62).
- Consultable solo por rol Dueño (RF57).

**Relaciones:**
- LogAuditoria **N..1 — 1..1** Usuario

---

## 7.2 Entidad: `LogErroresTecnicos`

**[Inferida desde RNF08]** – "El sistema debe registrar automáticamente cada error técnico en un log interno".

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| log_errores_tecnicos_id | Identificador | Único | Sí | **PK** |
| fecha_hora | Fecha-hora | DD/MM/AAAA HH:MM | Sí | |
| tipo_error | Texto | | Sí | |
| modulo | Texto | Módulo donde ocurrió | Sí | |
| descripcion_tecnica | Texto | Stack trace / detalle técnico | Sí | |
| usuario_id | Texto | Usuario afectado, si aplicaba sesión | No | FK → Usuario.usuario_id |

**Restricciones / extensiones:**
- Retención mínima de 3 meses; luego puede eliminarse automáticamente (RNF08).
- No visible desde la interfaz de usuario; accesible únicamente por el administrador del sistema (RNF08).

**Trazabilidad:** RNF08

---

# MÓDULO 8 – LECTOR DE CÓDIGO DE BARRAS

**Sin entidades persistentes.** Los RF59–RF61 describen comportamiento de hardware/integración. La validación del EAN-13 (RNF12) reutiliza la restricción ya definida sobre `Producto.ean_13` (atributo UNIQUE). Los códigos escaneados no se persisten salvo cuando resultan en una operación (Venta, Merma, etc.), caso en el cual se almacenan en las entidades correspondientes.

---

# RESUMEN GENERAL DE RELACIONES

Matriz consolidada de relaciones principales del modelo:

| Entidad origen | Cardinalidad | Entidad destino | RF/CU |
|---|---|---|---|
| Producto | 1..1 — 0..N | Lote | RF05 · CU05 |
| Producto | 1..1 — 0..N | Merma | RF10 · CU10 |
| Producto | 1..1 — 0..N | MovimientoInventario | RF12 · CU12 |
| Categoria | 1..1 — 0..N | Producto | RF01 · CU01 |
| Categoria | N..N — 0..N | Proveedor (vía ProveedorCategoria) | RF13 · CU13 |
| Proveedor | 1..1 — 0..N | Lote | RF05 · CU05 |
| Proveedor | 1..1 — 0..N | PedidoProveedor | RF17 · CU17 |
| PedidoProveedor | N..N — 1..N | Producto (vía DetallePedido) | RF17 · CU17 |
| PedidoProveedor | 1..1 — 0..N | Lote (al recibir) | RF18 · CU18 |
| PedidoProveedor | 1..1 — 1..N | HistorialAuditoriaPedido | RF18 · CU18, CU18b |
| Trabajador | 1..1 — 0..1 | Usuario | RF21 · CU21, CU21b |
| Trabajador | 1..1 — 0..N | Turno | RF25 · CU25 |
| Trabajador | 1..1 — 0..N | Asistencia | RF29 · CU29 |
| Trabajador | 1..1 — 0..N | Ausencia | RF32 · CU32 |
| Trabajador | 1..1 — 0..N | Remuneracion | RF34 · CU34 |
| Cargo | 1..1 — 0..N | Trabajador | RF21 · CU21 |
| Usuario | 1..1 — 0..N | Venta (como cajero) | RF42 · CU42 |
| Usuario | 1..1 — 0..N | SesionUsuario | RF55 · CU55 |
| Usuario | 1..1 — 0..N | IntentoLogin | RF55 · CU55 |
| Usuario | 1..1 — 0..N | ContrasenaTemporal | RF58 · CU58 |
| Usuario | 1..1 — 0..N | LogAuditoria | RF57 · CU57, CU62 |
| Usuario | 1..1 — 0..N | MovimientoInventario | RF12 · CU12 |
| Turno | 1..1 — 0..1 | Asistencia | RF29 · CU29 |

!!!

| Venta | N..N — 1..N | Producto (vía DetalleVenta) | RF35 · CU35 |
| Venta | 1..1 — 0..1 | AnulacionVenta | RF37 · CU37 |
| Venta | N..1 — 1..1 | CierreCaja | RF41 · CU41 |
| Lote | 1..1 — 0..N | MovimientoInventario | RF12, RF38 · CU12, CU38 |

---

# ANEXO – Entidades inferidas (resumen)

| Entidad | Justificación de inferencia | Derivada de |
|---|---|---|
| Categoria | Lista predefinida usada en múltiples RF; necesaria para M:N con Proveedor. La obligatoriedad de `fecha_vencimiento` por categoría se modela como extensión fija | RF01, RF05, RF13 |
| Cargo | Lista predefinida; la regla "cargos con acceso al sistema" y el mapeo cargo→rol se modelan como extensión fija | RF21 |
| Usuario | Necesaria para autenticación, sesiones, y para separarse de Trabajador (cargos operativos no tienen Usuario) | RF21, RF55, RF57 |
| ContrasenaTemporal | RF58 menciona "validez 24 horas" → requiere persistencia | RF21, RF22, RF58 |
| SesionUsuario | RF55 menciona expiración por inactividad → requiere estado de sesión persistente | RF55 |
| IntentoLogin | RF55 menciona "5 intentos fallidos consecutivos" → requiere contador persistente | RF55 |
| ConfiguracionDescuentos | RF34 menciona "porcentajes configurables por el dueño" → requiere persistencia con versiones | RF34 |
| HistorialAuditoriaPedido | RF18 y CU18b mencionan "historial de auditoría del pedido" | RF18, CU18b |
| AnulacionVenta | RF37 requiere registrar usuario, fecha, hora y razón de la anulación de manera persistente | RF37 |
| CierreCaja | RF41 menciona "registrar fecha, hora y usuario que lo ejecutó" y bloquear ventas posteriores | RF41 |
| LogErroresTecnicos | RNF08 requiere log interno con retención mínima | RNF08 |

> **Nota:** `ProveedorCategoria`, `DetallePedido` y `DetalleVenta` no aparecen en este anexo porque **no son entidades**, sino **relaciones N:M con atributos** (PK compuesta de las FK de las entidades adyacentes, sin identidad propia). Se documentan en sus módulos respectivos.
