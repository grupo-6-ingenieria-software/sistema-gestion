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
- **Atributos vs. extensiones:** los conjuntos cerrados de valores (p. ej. categorías) y las reglas fijas que dependen de ellos (qué categorías obligan vencimiento) se modelan como **extensiones/restricciones**, no como atributos booleanos. Un atributo se justifica solo cuando el valor puede variar legítimamente entre instancias.
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
- Producto **1..1 — 0..N** AjusteInventario (historial de ajustes manuales sobre este producto)
- Producto **1..1 — 0..N** DetalleVenta (referencia desde las líneas de venta)
- Producto **1..1 — 0..N** DetallePedido
- Producto **N..N — 0..N** PedidoProveedor (vía DetallePedido)

**Restricciones / extensiones:**
- No puede eliminarse (RF03) si existen registros de Venta, Merma o AjusteInventario asociados. En tal caso debe desactivarse (cambio de estado a "Inactivo").
- En estado Inactivo no puede aparecer en módulos de Venta ni Merma, pero conserva su historial (RF04).
- En reportes para el rol Trabajador, el atributo `precio_costo` no debe exponerse; solo el rol Dueño puede verlo (RF56).

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
- Lote **N..1 — 0..1** PedidoProveedor (si se originó de la recepción de un pedido — vía `Lote.pedido_origen_id`)
- Lote **N..N — 0..N** Venta (vía `VentaLote`; los lotes consumidos por cada venta)
- Lote **N..N — 0..N** Merma (vía `MermaLote`; los lotes afectados por cada merma)
- Lote **1..1 — 0..N** AjusteInventario (un lote puede recibir varios ajustes manuales a lo largo del tiempo)

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

**Relaciones:**
- Merma **N..1 — 1..1** Producto
- Merma **N..1 — 1..1** Usuario
- Merma **N..N — 1..N** Lote (vía `MermaLote`; una merma puede afectar varios lotes por FEFO/FIFO)

**Restricciones / extensiones:**
- Cada Merma genera una o varias filas en `MermaLote` (una por lote afectado vía FEFO/FIFO). La trazabilidad por lote vive en esa entidad asociativa.

---

## 1.4b Relación: `MermaLote` (N:M entre Merma y Lote)

Relación con atributos. Materializa el descuento FEFO/FIFO por lote: una merma de un producto se distribuye en uno o varios lotes según el orden de salida correspondiente.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| merma_id | Identificador | | Sí | **PK + FK** → Merma.merma_id |
| lote_id | Identificador | | Sí | **PK + FK** → Lote.lote_id |
| cantidad_descontada | Entero | > 0 | Sí | |

**Clave primaria compuesta:** (merma_id, lote_id).

**Restricciones / extensiones:**
- La suma de `cantidad_descontada` para una misma `merma_id` debe igualar `Merma.cantidad`.
- Cada `lote_id` referenciado debe pertenecer a `Lote` con `producto_id` consistente con `Merma.producto_id`.

**Trazabilidad:** RF10, RF38 · CU10, CU38

---

## 1.5 Entidad: `AjusteInventario`

**Trazabilidad:** RF11, RF12, RF52 · CU11, CU12, CU52

Registra los **ajustes manuales** de inventario (correcciones de stock sin causa transaccional). Los movimientos de stock por venta, merma o recepción se modelan vía las relaciones N:M explícitas (`VentaLote`, `MermaLote`) y la FK directa `Lote.pedido_origen_id`, respectivamente.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| ajuste_inventario_id | Identificador | Único, autogenerado | Sí | **PK** |
| producto_id | Numérico (13) | | Sí | **FK** → Producto.producto_id |
| lote_id | Identificador | Lote afectado por el ajuste | Sí | **FK** → Lote.lote_id |
| cantidad | Entero | Positiva para incremento, negativa para decremento; ≠ 0 | Sí | |
| justificacion | Texto | 10..300 caracteres, obligatoria (RF11) | Sí | |
| stock_resultante | Entero | Stock total del producto luego del ajuste (snapshot) | Sí | |
| fecha_hora | Fecha-hora | DD/MM/AAAA HH:MM | Sí | |
| usuario_id | Identificador | Usuario responsable | Sí | **FK** → Usuario.usuario_id |

**Relaciones:**
- AjusteInventario **N..1 — 1..1** Producto
- AjusteInventario **N..1 — 1..1** Lote
- AjusteInventario **N..1 — 1..1** Usuario

**Restricciones / extensiones:**
- El ajuste no puede dejar la `cantidad_actual` del lote ni el `stock_actual` del producto en negativo.
- La `justificacion` es siempre obligatoria (a diferencia de venta/merma/entrada, que no requieren justificación específica porque su trazabilidad viene del documento padre).
- Reportes RF12 (historial por producto) y RF52 (auditoría general) se obtienen como **UNION** de: `VentaLote` (con join a Venta), `MermaLote` (con join a Merma), `Lote` filtrado por `pedido_origen_id` (entradas), y `AjusteInventario`.

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
| nombre | Texto | Máx. 50 caracteres | Sí | |
| apellido | Texto | Máx. 50 caracteres | Sí | |
| telefono | Numérico | Exactamente 9 dígitos | Sí | |
| correo_electronico | Texto | Formato email válido si se ingresa | No | |
| fecha_ingreso | Fecha | Por defecto = fecha de registro | Sí | |
| estado | Enum {Activo, Inactivo} | Por defecto "Activo" | Sí | |

**Relaciones:**
- Trabajador **1..1 — 1..1** Usuario (todo trabajador tiene exactamente una cuenta de acceso; la FK reside en `Usuario.trabajador_id`)
- Trabajador **1..1 — 0..N** Turno
- Trabajador **1..1 — 0..N** Asistencia
- Trabajador **1..1 — 0..N** Ausencia
- Trabajador **1..1 — 0..N** Remuneracion

**Restricciones / extensiones:**
- Todo `Trabajador` tiene asociado exactamente un `Usuario` (relación 1:1 obligatoria). El nivel de acceso lo determina `Usuario.rol ∈ {dueño, trabajador}`; ya no existen cargos diferenciados ni subtipos de trabajador. Todos los trabajadores con rol `trabajador` tienen el mismo nivel de acceso operativo (caja, reposición, etc.).
- El campo `rut` es no editable tras el registro (RF22).
- Un trabajador Inactivo no aparece para asignación de turnos ni asistencia, pero conserva su historial (RF23).

---

## 4.2 Entidad: `Usuario`

**[Inferida]** – No mencionada como entidad explícita pero presupuesta por RF21, RF55, RF56, RF57, RF58. Se separa de Trabajador para concentrar los datos de autenticación (credenciales, sesiones, rol de acceso).

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| usuario_id | Texto | Generado a partir del RUT del trabajador, único, sin espacios, máx. 50 chars | Sí | **PK** |
| trabajador_id | Identificador | El trabajador asociado | Sí | **FK** → Trabajador.trabajador_id (UNIQUE; 1:1) |
| rol | Enum {dueño, trabajador} | Nivel de acceso asignado al crear el usuario | Sí | |
| fecha_creacion | Fecha-hora | | Sí | |
| ultimo_login | Fecha-hora | Actualizado en cada login exitoso | No | |

**Relaciones:**
- Usuario **1..1 — 1..1** Trabajador
- Usuario **1..1 — 1..N** Contrasena (historial de contraseñas; rol "para")
- Usuario **1..1 — 0..N** Contrasena_Temporal (rol "generada_por"; sólo si es Dueño)
- Usuario **1..1 — 0..N** SesionUsuario
- Usuario **1..1 — 0..N** IntentoLogin
- Usuario **1..1 — 0..N** Venta (como cajero responsable)
- Usuario **1..1 — 0..N** AjusteInventario (como responsable del ajuste)
- Usuario **1..1 — 0..N** LogAuditoria

**Restricciones / extensiones:**
- La contraseña en sí se gestiona en la entidad `Contrasena` (definitiva o temporal). El usuario no almacena hash directamente.
- Política de contraseña: mínimo 8 caracteres, al menos una mayúscula, una minúscula y un número (RF55).
- Tras 5 intentos fallidos consecutivos la cuenta se bloquea 15 minutos (RF55).
- Cierre automático de sesión tras 30 minutos de inactividad (RF55).
- "Requiere cambio de contraseña" se deriva: True ⟺ existe `Contrasena_Temporal` con `estado = Vigente` para este usuario.

---

## 4.3 Entidad: `Contrasena`

**[Inferida]** – RF21, RF22, RF55, RF58. Unifica las contraseñas definitivas y temporales bajo un mismo supertipo con ISA disjunta y total, preservando el historial completo de contraseñas por usuario.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| contrasena_id | Identificador | Único, autogenerado | Sí | **PK** |
| usuario_id | Texto | Titular de la contraseña | Sí | **FK** → Usuario.usuario_id |
| hash | Texto | Hash con sal (bcrypt o equivalente; RNF04) | Sí | |
| fecha_hora_creacion | Fecha-hora | | Sí | |
| estado | Enum {Vigente, Reemplazada, Usada, Expirada, Invalidada} | Estados aplicables según subtipo | Sí | |
| ES_contrasena_definitiva | Booleano | Discriminador ISA | Sí | |
| ES_contrasena_temporal | Booleano | Discriminador ISA | Sí | |

**Relaciones:**
- Contrasena **N..1 — 1..1** Usuario (rol: "para"; titular)

**Restricciones / extensiones:**

**Especialización ISA disjunta y total de `Contrasena`** (mapping plano en la misma tabla):
- Subentidad **`Contrasena_Definitiva`**: sin atributos específicos. Estados válidos: {Vigente, Reemplazada}.
- Subentidad **`Contrasena_Temporal`**: atributos específicos `contrasena_temporal_fecha_hora_expiracion` (= `fecha_hora_creacion` + 24h, RF58) y la relación **"generada_por"** con Usuario (rol Dueño). Estados válidos: {Vigente, Usada, Expirada, Invalidada}.
- Invariante: por cada fila, exactamente uno de `{ES_contrasena_definitiva, ES_contrasena_temporal}` es True.

**Otras restricciones:**
- Para cada `usuario_id`, debe existir como máximo **una** Contrasena con `estado = Vigente` y `ES_contrasena_definitiva = True` (la contraseña activa actual).
- Para cada `usuario_id`, debe existir como máximo **una** Contrasena con `estado = Vigente` y `ES_contrasena_temporal = True` (la temporal activa actual).
- Al generar una nueva Contrasena_Temporal, la anterior vigente (si existe) se marca como `Invalidada`. Al canjear una temporal por una definitiva, la temporal se marca como `Usada` y la definitiva previa (si existía) se marca como `Reemplazada`.
- La contraseña en texto plano se muestra una única vez al generarla; sólo se persiste el `hash`.
- La contraseña en texto plano se muestra una única vez al dueño que la generó; sólo se persiste el `hash_temporal`.
- `generada_por_usuario_id` debe corresponder a un Usuario con rol Dueño.

**Trazabilidad:** RF21, RF22, RF58 · CU21, CU22, CU58, CU55b

---

## 4.4 Entidad: `SesionUsuario`

**[Inferida]** – RF55 ("sesión se cerrará automáticamente tras 30 minutos de inactividad"). RNF18–RNF20 (sync engine) presuponen sesiones rastreables.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| sesion_usuario_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | | Sí | **FK** → Usuario.usuario_id |
| fecha_inicio | Fecha-hora | | Sí | |
| fecha_ultimo_acceso | Fecha-hora | Actualizado en cada interacción | Sí | |
| fecha_cierre | Fecha-hora | | No | |
| motivo_cierre | Enum {manual, inactividad, sistema} | manual = cierre por logout del usuario; inactividad = expiración por inactividad; sistema = cierre forzado por el sistema/admin | No | |

**Trazabilidad:** RF55 · CU55

---

## 4.5 Entidad: `IntentoLogin`

**[Inferida]** – RF55 menciona contador de "5 intentos fallidos consecutivos" y bloqueo temporal por 15 minutos.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| intento_login_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | Puede ser nulo si el usuario ingresado no existe | No | **FK** → Usuario.usuario_id |
| nombre_usuario_intentado | Texto | Lo que efectivamente se tipeó | Sí | |
| fecha_hora | Fecha-hora | | Sí | |
| exitoso | Booleano | | Sí | |

**Restricciones / extensiones:**
- El sistema mantiene un contador derivado de intentos fallidos consecutivos por `usuario_id`. Tras 5 fallidos consecutivos se gatilla bloqueo por 15 minutos. El contador se reinicia tras un login exitoso o tras transcurridos los 15 minutos (RF55).

**Trazabilidad:** RF55 · CU55

---

## 4.6 Entidad: `Turno`

**Trazabilidad:** RF25, RF26, RF27, RF28 · CU25, CU26, CU27, CU28

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| turno_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | Trabajador Activo | Sí | **FK** → Trabajador.trabajador_id |
| fecha_hora_inicio | Fecha-hora | DD/MM/AAAA HH:MM | Sí | |
| fecha_hora_fin | Fecha-hora | > fecha_hora_inicio | Sí | |
| estado | Enum {Pendiente, Ejecutado, Eliminado} | Derivado de fecha_hora_inicio vs. ahora() y existencia de asistencia(s) | Sí | |

**Relaciones:**
- Turno **N..1 — 1..1** Trabajador
- Turno **0..N — 0..1** Asistencia (vía "registra"; un turno puede ser registrado por varias asistencias si el trabajador entra y sale múltiples veces el mismo día)

**Restricciones / extensiones:**
- No puede existir otro turno del mismo trabajador con rango horario solapado al ya existente (RF25).
- Solo puede modificarse o eliminarse si `fecha_hora_inicio` es posterior a ahora() y no tiene asistencia registrada (RF26, RF27).

---

## 4.7 Entidad: `Asistencia`

**Trazabilidad:** RF29, RF30, RF31, RF33, RF51 · CU29, CU29b, CU30, CU31, CU33, CU51

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| asistencia_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | | Sí | **FK** → Trabajador.trabajador_id |
| fecha_hora_entrada | Fecha-hora | DD/MM/AAAA HH:MM (registrada automáticamente al marcar) | Sí | |
| fecha_hora_salida | Fecha-hora | NULL mientras la jornada esté abierta | No | |
| horas_trabajadas | Derivado | = fecha_hora_salida − fecha_hora_entrada; "Pendiente" si NULL | Sí | |
| turno_id | Identificador | Turno que la asistencia registra; NULL si la entrada fue sin turno asignado (CU29b) | No | FK → Turno.turno_id |

**Relaciones:**
- Asistencia **N..1 — 1..1** Trabajador
- Asistencia **N..1 — 0..1** Turno (vía "registra")

**Restricciones / extensiones:**
- Un trabajador puede tener **múltiples asistencias el mismo día** (jornadas partidas por almuerzo, etc.).
- Dos asistencias del mismo trabajador no pueden tener rangos `[fecha_hora_entrada, fecha_hora_salida]` solapados.
- Sólo puede existir **una** asistencia con `fecha_hora_salida = NULL` por trabajador a la vez (no se admite abrir una nueva sin cerrar la anterior).
- No puede coexistir Asistencia con Ausencia en el mismo (trabajador, día calendario), usando `DATE(asistencia.fecha_hora_entrada)` para la comparación (RF32).
- Para asistencias "sin turno asignado" (CU29b): `turno_id IS NULL`. No se persiste un campo separado para esta condición.

---

## 4.8 Entidad: `Ausencia`

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
- No puede coexistir con Asistencia para el mismo (`trabajador_id`, `fecha`), comparando contra `DATE(asistencia.fecha_hora_entrada)` (RF32).

---

## 4.9 Entidad: `Remuneracion`

**Trazabilidad:** RF34 · CU34

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| remuneracion_id | Identificador | Único | Sí | **PK** |
| trabajador_id | Identificador | | Sí | **FK** → Trabajador.trabajador_id |
| mes | Entero | 1..12 | Sí | |
| anio | Entero | AAAA | Sí | |
| monto_bruto | Entero | > 0, en pesos chilenos | Sí | |
| tasa_afp_aplicada | Decimal | Snapshot de la tasa AFP al momento del cálculo | Sí | |
| tasa_salud_aplicada | Decimal | Snapshot de la tasa Salud | Sí | |
| tasa_cesantia_aplicada | Decimal | Snapshot de la tasa Seguro Cesantía | Sí | |
| monto_liquido | Entero | = bruto – descuentos; ≥ 0 | Sí | |
| observacion | Texto | Máx. 200 caracteres | No | |
| fecha_registro | Fecha-hora | | Sí | |
| usuario_registrador_id | Texto | | Sí | **FK** → Usuario.usuario_id |

**Restricciones / extensiones:**
- Único por (trabajador_id, mes, anio): un trabajador no puede tener dos remuneraciones del mismo período (RF34).
- Los valores por defecto al crear una nueva Remuneración se leen de la **Remuneración más reciente del sistema** (`ORDER BY anio DESC, mes DESC LIMIT 1`). Si no existe ninguna previa, se usan los valores RF34: AFP 11.5%, Salud 7%, Cesantía 0.6%. Los `tasa_*_aplicada` quedan congelados al insertar y representan los valores efectivos al momento del cálculo (RF34).

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
- Al confirmar, gatilla descuento de stock FEFO/FIFO en `Lote` y crea filas en `VentaLote` (una por lote consumido), una por cada `DetalleVenta` (RF38).

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

**Clave primaria compuesta:** (venta_id, producto_id).

**Restricciones / extensiones:**
- Un mismo producto no puede aparecer más de una vez por venta (debe consolidarse cantidad).
- La trazabilidad por lote (qué lotes se consumieron por FEFO/FIFO) vive en `VentaLote`, ligada al par (`venta_id`, `producto_id`). Permite revertir el stock con precisión al anular (RF37).

**Trazabilidad:** RF35, RF38, RF44 · CU35, CU38, CU44

---

## 5.2b Relación: `VentaLote` (N:M entre Venta y Lote)

Relación con atributos. Materializa el descuento FEFO/FIFO por lote dentro de cada línea de venta: una venta de un producto se distribuye en uno o varios lotes según el orden de salida correspondiente.

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| venta_id | Identificador | | Sí | **PK + FK** → Venta.venta_id |
| lote_id | Identificador | | Sí | **PK + FK** → Lote.lote_id |
| cantidad_consumida | Entero | > 0 | Sí | |

**Clave primaria compuesta:** (venta_id, lote_id).

**Restricciones / extensiones:**
- Para cada `DetalleVenta` (venta_id, producto_id), la suma de `cantidad_consumida` en `VentaLote` de los lotes asociados a ese producto debe igualar `DetalleVenta.cantidad`.
- Cada `lote_id` referenciado debe pertenecer a `Lote` con `producto_id` consistente con la línea de venta correspondiente.

**Trazabilidad:** RF38 · CU38

---

## 5.3 Entidad: `AnulacionVenta`

**[Inferida]** – RF37 menciona "registrar la anulación con usuario responsable, fecha y hora".

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| anulacion_venta_id | Identificador | Único | Sí | **PK** |
| venta_id | Identificador | Venta a anular, debe estar Confirmada | Sí | **FK** UNIQUE → Venta.venta_id |
| usuario_id | Texto | Trabajador (solo ventas del día) o Dueño (cualquier día) | Sí | **FK** → Usuario.usuario_id |
| fecha_hora | Fecha-hora | = ahora() | Sí | |
| razon | Texto | 10..200 caracteres | Sí | |

**Relaciones:**
- AnulacionVenta **1..1 — 1..1** Venta (0..1 desde la perspectiva de Venta)

**Restricciones / extensiones:**
- Al confirmar la anulación se revierte el stock de cada producto involucrado consultando `VentaLote` para conocer los lotes consumidos y restaurando sus `cantidad_actual`. Se registra un `AjusteInventario` compensatorio por cada lote afectado con `justificacion = "Reversión por anulación venta #<venta_id>"` (RF37).
- Solo el rol Dueño puede anular ventas de días anteriores (RF37, RF56).

**Trazabilidad:** RF37 · CU37

---

## 5.4 Entidad: `CierreCaja`

**Trazabilidad:** RF41 · CU41

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| cierre_caja_id | Identificador | Único | Sí | **PK** |
| fecha_hora_inicio | Fecha-hora | Se llena al primer movimiento del día; única por día (`DATE(fecha_hora_inicio)`) | Sí | |
| estado | Enum {Abierta, Cerrada} | Por defecto "Abierta" al primer movimiento del día | Sí | |
| fecha_hora_fin | Fecha-hora | Se completa al cerrar | No | |
| usuario_cierre_id | Texto | Usuario (Dueño o Trabajador) que cerró | No | **FK** → Usuario.usuario_id |
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
| RF51 (asistencia) | CU51 | Trabajador, Asistencia, Ausencia |
| RF52 (auditoría de movimientos) | CU52 | UNION de DetalleVentaLote (vía Venta), MermaLote (vía Merma), Lote (entradas vía `pedido_origen_id`) y AjusteInventario, con join a Producto y Usuario |
| RF53 (exportación de reportes) | CU53 | (orquestador) — la acción se registra en LogAuditoria |

> Toda exportación de reportes queda registrada como evento en `LogAuditoria` con tipo de acción "Exportación de reportes" (RF57).

---

# MÓDULO 7 – DASHBOARD Y CONTROL DE ACCESO

El RF54 (Dashboard) y RF56 (Control de acceso por rol) son **vistas/políticas** sobre las entidades ya definidas; no requieren entidades nuevas. Las entidades de soporte (Usuario, SesionUsuario, IntentoLogin, Contrasena) ya fueron descritas en el Módulo 4 (Trabajadores y Asistencia) por ser dependientes de Trabajador.

A continuación se describen las entidades **propias** del módulo: `LogAuditoria` y `LogErroresTecnicos`.

## 7.1 Entidad: `LogAuditoria`

**Trazabilidad:** RF57, RNF09 · CU57, CU62

| Atributo | Tipo | Restricciones | Obligatorio | PK/FK |
|---|---|---|---|---|
| log_auditoria_id | Identificador | Único | Sí | **PK** |
| usuario_id | Texto | Usuario que ejecutó la acción | Sí | **FK** → Usuario.usuario_id |
| nombre_usuario_snapshot | Texto | Snapshot al momento de la acción | Sí | |
| rol_snapshot | Enum {dueño, trabajador} | Snapshot al momento de la acción | Sí | |
| fecha_hora | Fecha-hora | | Sí | |
| tipo_accion | Enum {Inicio sesión, Cierre sesión, Registro, Edición, Eliminación, Anulación, Exportación reportes, Consulta rentabilidad, Consulta remuneraciones} | RF57 | Sí | |
| modulo | Texto | Módulo donde ocurrió | Sí | |
| descripcion | Texto | Descripción de la acción | Sí | |

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

Matriz consolidada de **todas** las relaciones del modelo, organizada por módulo. Cada fila corresponde a una relación binaria entre dos entidades (las relaciones N:M se indican con "vía <Relación>"). Las relaciones aparecen una sola vez (no se duplican en ambos sentidos).

### Módulo 1 – Inventario

| Entidad origen | Cardinalidad | Entidad destino | RF/CU |
|---|---|---|---|
| Categoria | 1..1 — 0..N | Producto | RF01 · CU01 |
| Producto | 1..1 — 0..N | Lote | RF05 · CU05 |
| Producto | 1..1 — 0..N | Merma | RF10 · CU10 |
| Producto | 1..1 — 0..N | AjusteInventario | RF11, RF12 · CU11, CU12 |
| Lote | 1..1 — 0..N | AjusteInventario | RF11 · CU11 |
| Merma | N..N — 1..N | Lote (vía `MermaLote`; descuento FEFO/FIFO) | RF10, RF38 · CU10, CU38 |
| Venta | N..N — 0..N | Lote (vía `VentaLote`; descuento FEFO/FIFO) | RF38 · CU38 |
| Usuario | 1..1 — 0..N | Merma (usuario que registra) | RF10 · CU10 |
| Usuario | 1..1 — 0..N | AjusteInventario (responsable) | RF11 · CU11 |

### Módulo 2 – Proveedores

| Entidad origen | Cardinalidad | Entidad destino | RF/CU |
|---|---|---|---|
| Proveedor | N..N — 0..N | Categoria (vía `ProveedorCategoria`) | RF13 · CU13 |
| Proveedor | 0..1 — 0..N | Lote (origen del lote, opcional) | RF05 · CU05 |
| Proveedor | 1..1 — 0..N | PedidoProveedor | RF17 · CU17 |
| PedidoProveedor | N..N — 1..N | Producto (vía `DetallePedido`) | RF17 · CU17 |
| PedidoProveedor | 1..1 — 0..N | Lote (lotes generados al recibir) | RF18 · CU18 |
| PedidoProveedor | 1..1 — 1..N | HistorialAuditoriaPedido | RF18 · CU18, CU18b |
| Usuario | 1..1 — 0..N | HistorialAuditoriaPedido (actor del evento) | RF18 · CU18b |

### Módulo 4 – Trabajadores, Acceso y Asistencia

| Entidad origen | Cardinalidad | Entidad destino | RF/CU |
|---|---|---|---|
| Trabajador | 1..1 — 1..1 | Usuario (todo trabajador tiene una cuenta de acceso) | RF21 · CU21 |
| Trabajador | 1..1 — 0..N | Turno | RF25 · CU25 |
| Trabajador | 1..1 — 0..N | Asistencia | RF29 · CU29 |
| Trabajador | 1..1 — 0..N | Ausencia | RF32 · CU32 |
| Trabajador | 1..1 — 0..N | Remuneracion | RF34 · CU34 |
| Turno | 0..1 — 0..N | Asistencia (vía "registra"; un turno puede ser registrado por varias asistencias del mismo día) | RF29 · CU29 |
| Usuario | 1..1 — 0..N | SesionUsuario | RF55 · CU55 |
| Usuario | 1..1 — 0..N | IntentoLogin | RF55 · CU55 |
| Usuario | 1..1 — 1..N | Contrasena (historial; rol "para") | RF55, RF58 · CU55, CU58 |
| Usuario | 1..1 — 0..N | Contrasena_Temporal (rol "generada_por"; sólo si es Dueño) | RF22, RF58 · CU22, CU58 |
| Usuario | 1..1 — 0..N | Ausencia (dueño que registra) | RF32 · CU32 |
| Usuario | 1..1 — 0..N | Remuneracion (dueño que registra) | RF34 · CU34 |

### Módulo 5 – Ventas

| Entidad origen | Cardinalidad | Entidad destino | RF/CU |
|---|---|---|---|
| Venta | N..N — 1..N | Producto (vía `DetalleVenta`) | RF35 · CU35 |
| Venta | 1..1 — 0..1 | AnulacionVenta | RF37 · CU37 |
| Venta | N..1 — 1..1 | CierreCaja (día operativo) | RF41 · CU41 |
| Usuario | 1..1 — 0..N | Venta (cajero responsable) | RF42 · CU42 |
| Usuario | 1..1 — 0..N | AnulacionVenta (trabajador o dueño que anula) | RF37 · CU37 |
| Usuario | 0..1 — 0..N | CierreCaja (usuario que ejecuta el cierre) | RF41 · CU41 |

### Módulo 7 – Control de Acceso, Dashboard y Logs

| Entidad origen | Cardinalidad | Entidad destino | RF/CU |
|---|---|---|---|
| Usuario | 1..1 — 0..N | LogAuditoria (actor de la acción auditada) | RF57 · CU57, CU62 |
| Usuario | 0..1 — 0..N | LogErroresTecnicos (usuario afectado si había sesión) | RNF08 |

---

# ANEXO – Entidades inferidas (resumen)

| Entidad | Justificación de inferencia | Derivada de |
|---|---|---|
| Categoria | Lista predefinida usada en múltiples RF; necesaria para M:N con Proveedor. La obligatoriedad de `fecha_vencimiento` por categoría se modela como extensión fija | RF01, RF05, RF13 |
| Usuario | Necesaria para autenticación, sesiones y rol de acceso; se separa de Trabajador para concentrar las credenciales. El nivel de acceso lo da `rol ∈ {dueño, trabajador}` | RF21, RF55, RF57 |
| Contrasena (con ISA Definitiva/Temporal) | RF55 (política de contraseña), RF58 (temporal 24h) → requiere persistencia del historial; unifica ambos tipos bajo ISA disjunta total | RF21, RF22, RF55, RF58 |
| SesionUsuario | RF55 menciona expiración por inactividad → requiere estado de sesión persistente | RF55 |
| IntentoLogin | RF55 menciona "5 intentos fallidos consecutivos" → requiere contador persistente | RF55 |
| HistorialAuditoriaPedido | RF18 y CU18b mencionan "historial de auditoría del pedido" | RF18, CU18b |
| AnulacionVenta | RF37 requiere registrar usuario, fecha, hora y razón de la anulación de manera persistente | RF37 |
| CierreCaja | RF41 menciona "registrar fecha, hora y usuario que lo ejecutó" y bloquear ventas posteriores | RF41 |
| LogErroresTecnicos | RNF08 requiere log interno con retención mínima | RNF08 |

> **Nota:** `ProveedorCategoria`, `DetallePedido` y `DetalleVenta` no aparecen en este anexo porque **no son entidades**, sino **relaciones N:M con atributos** (PK compuesta de las FK de las entidades adyacentes, sin identidad propia). Se documentan en sus módulos respectivos.
