# Análisis de Tercera Forma Normal (3FN) y Normalización Iterativa
## Sistema de Gestión Minimarket y Panadería Huáscar

---

## 0. Marco teórico aplicado

Para que un esquema relacional esté en **3FN** debe cumplir simultáneamente:

1. **1FN** — todos los atributos son atómicos; no hay grupos repetitivos ni listas.
2. **2FN** — está en 1FN y todo atributo no clave depende de la **clave primaria completa** (no de una parte de ella).
3. **3FN** — está en 2FN y **ningún atributo no clave depende transitivamente** de la clave primaria a través de otro atributo no clave. Formalmente: para toda dependencia funcional `X → A`, o bien `X` es superclave, o bien `A` es atributo primo.

Equivalentemente, un atributo viola 3FN si:
- **Es derivable** de otros atributos de la misma fila o de otras tablas (redundancia funcional).
- **Depende de otro atributo no clave** en la misma tabla.
- **Duplica datos** de otra entidad (snapshot/desnormalización).

El modelo original (`modelo_relacional.docx`) declara explícitamente en `modelo_datos.md` §0.3:

> *"Se adopta una visión conceptual relajada (no se exige 2FN/3FN). Algunos atributos pueden ser multivaluados o derivados, según el dominio."*

Por tanto el modelo original **no está en 3FN por diseño**. Este documento identifica cada violación y deriva un esquema equivalente en 3FN estricta mediante normalización iterativa.

---

## 1. Inventario de violaciones de 3FN

### 1.1 Atributos derivados (redundancia funcional)

Un atributo derivado es función de otros atributos del esquema. Su valor se puede recomputar, por lo que almacenarlo introduce posibilidad de inconsistencia y viola 3FN.

| # | Tabla | Atributo | Fórmula de derivación |
|---|---|---|---|
| V1 | `Producto` | `Producto_Stock_Actual` | `Σ Lote.Lote_Cantidad_Actual WHERE Lote.Producto_id = Producto.Producto_id` |
| V2 | `Producto` | `Producto_Precio_Costo` | Última `Lote.Lote_Precio_Costo` recibida (o ponderado) |
| V3 | `Lote` | `Lote_Estado` | Función de `Lote_Cantidad_Actual` y `Lote_Perecible_Fecha_Vencimiento` (Activo / Agotado / Vencido) |
| V4 | `Merma` | `Merma_Cantidad` | `Σ MermaLote.Merma_Lote_Cantidad_Descontada WHERE Merma_id = X` (invariante declarada en §1.4b) |
| V5 | `AjusteInventario` | `Ajuste_Stock_Resultante` | Snapshot del stock total post-ajuste |
| V6 | `Asistencia` | `Asistencia_Horas_Trabajadas` | `Asistencia_Fecha_Hora_Salida − Asistencia_Fecha_Hora_Entrada` |
| V7 | `Remuneracion` | `Remuneracion_Monto_Liquido` | `Monto_Bruto × (1 − Σ Tasas_Aplicadas)` |
| V8 | `DetalleVenta` | `Detalle_Venta_Subtotal_Linea` | `Detalle_Venta_Cantidad × Detalle_Venta_Precio_Unitario` |
| V9 | `Venta` | `Venta_Subtotal` | `Σ DetalleVenta.Detalle_Venta_Subtotal_Linea WHERE Venta_id = X` |
| V10 | `Venta` | `Venta_Total` | `Venta_Subtotal − Venta_Descuento_Valor` |
| V11 | `Venta` | `Venta_Efectivo_Monto_Vuelto` | `Venta_Efectivo_Monto_Recibido − Venta_Total` |
| V12 | `CierreCaja` | `Cierre_Total_Ventas` | `Σ Venta.Venta_Total` del período del cierre |
| V13 | `CierreCaja` | `Cierre_Total_Efectivo` | `Σ Venta.Venta_Total WHERE ES_Venta_Efectivo = TRUE` |
| V14 | `CierreCaja` | `Cierre_Total_Debito`, `Cierre_Total_Credito`, `Cierre_Total_Transferencia` | Análogo, segmentado por `Venta_Metodo_Pago` |
| V15 | `CierreCaja` | `Cierre_Cantidad_Anulaciones`, `Cierre_Monto_Anulaciones` | `COUNT/SUM` sobre `AnulacionVenta` |
| V16 | `Contrasena` | `Contrasena_Estado` | Función de `Contrasena_Temporal_Fecha_Hora_Expiracion` vs. ahora |

### 1.2 Dependencias funcionales transitivas

Un atributo viola 3FN si depende de otro atributo no clave en lugar de la PK.

| # | Tabla | Dependencia transitiva | Explicación |
|---|---|---|---|
| T1 | `Categoria` (implícita) | `Categoria_id → exige_vencimiento → ES_Lote_Perecible` (en `Lote`) | El discriminador ISA de Lote depende de la categoría del producto, no del lote. |

### 1.3 Flattening de ISA (especialización en una sola tabla)

El modelo original aplica *single-table inheritance* con discriminadores `ES_*` y atributos condicionales con NULL. Esto viola 3FN porque los atributos del subtipo dependen del discriminador, no de la PK.

| # | Supertipo | Subtipos | Atributos condicionales (NULL si no aplica) |
|---|---|---|---|
| I1 | `Lote` | `Lote_Perecible` / `Lote_No_Perecible` | `Lote_Perecible_Fecha_Vencimiento` |
| I2 | `Contrasena` | `Contrasena_Definitiva` / `Contrasena_Temporal` | `Contrasena_Temporal_Fecha_Hora_Expiracion` |
| I3 | `Venta` | `Venta_Efectivo` / `Venta_Electronica` | `Venta_Efectivo_Monto_Recibido`, `Venta_Efectivo_Monto_Vuelto` |

### 1.4 Snapshots (duplicación de datos de otra entidad)

Aceptables en el modelo original por requisitos de auditoría e inmutabilidad, pero estrictamente violan 3FN.

| # | Tabla | Atributo snapshot | Fuente original |
|---|---|---|---|
| S1 | `DetalleVenta` | `Detalle_Venta_Precio_Unitario` | `Producto.Producto_Precio_Venta` al momento de la venta |
| S2 | `DetalleVenta` | `Detalle_Venta_Snapshot_Categoria` | `Categoria.Categoria_Nombre` vía `Producto.Categoria_id` |
| S3 | `LogAuditoria` | `Log_Nombre_Usuario_Snapshot` | `Usuario` / `Trabajador.Trabajador_Nombre` |
| S4 | `LogAuditoria` | `Log_Rol_Snapshot` | `Usuario.Usuario_Rol` |
| S5 | `Remuneracion` | `Remuneracion_Tasa_Afp_Aplicada`, `Tasa_Salud_Aplicada`, `Tasa_Cesantia_Aplicada` | Tasas legales vigentes ese mes (no modeladas como entidad) |

---

## 2. Normalización iterativa

Se aplican cuatro rondas. Cada ronda resuelve una categoría de violación y produce un esquema intermedio. La ronda 4 produce el esquema final en 3FN.

### Ronda 1 — Eliminación de atributos derivados

Se eliminan los atributos cuyo valor es función computable de otros atributos del esquema. Las consultas que los necesiten los recalcularán (o bien se materializarán como vistas SQL).

**Cambios:**

| Tabla | Eliminados |
|---|---|
| `Producto` | `Producto_Stock_Actual`, `Producto_Precio_Costo` |
| `Lote` | `Lote_Estado` |
| `Merma` | `Merma_Cantidad` |
| `AjusteInventario` | `Ajuste_Stock_Resultante` |
| `Asistencia` | `Asistencia_Horas_Trabajadas` |
| `Remuneracion` | `Remuneracion_Monto_Liquido` |
| `DetalleVenta` | `Detalle_Venta_Subtotal_Linea` |
| `Venta` | `Venta_Subtotal`, `Venta_Total`, `Venta_Efectivo_Monto_Vuelto` |
| `CierreCaja` | `Cierre_Total_Ventas`, `Cierre_Total_Efectivo`, `Cierre_Total_Debito`, `Cierre_Total_Credito`, `Cierre_Total_Transferencia`, `Cierre_Cantidad_Anulaciones`, `Cierre_Monto_Anulaciones` |
| `Contrasena` | `Contrasena_Estado` |

**Justificación:** las consultas de reporte (RF45–RF53) ejecutarán las agregaciones bajo demanda. Si el desempeño exige materialización, se hará explícitamente vía vistas indexadas (`CREATE MATERIALIZED VIEW`), no vía columnas redundantes en las tablas base.

### Ronda 2 — Resolución de dependencias transitivas

El discriminador de perecibilidad de un lote depende de la `Categoria` del producto, no del `Lote`. Se externaliza la regla a `Categoria`.

**Cambios:**

- `Categoria`: se añade `Categoria_Exige_Vencimiento` (Booleano) para externalizar la regla que determina si un lote de esa categoría es perecible.

Tras esta ronda, la pregunta "¿este lote vence?" se responde con `JOIN Categoria` y la regla deja de ser local al lote.

> **Nota sobre Trabajador:** el modelo vigente eliminó la entidad `Cargo` y la especialización ISA de `Trabajador` (todo trabajador tiene exactamente un `Usuario` y el nivel de acceso vive en `Usuario.rol ∈ {dueño, trabajador}`). Por tanto las antiguas dependencias transitivas vía `Cargo` (los flags `ES_Trabajador_*`) ya no existen y no requieren normalización.

### Ronda 3 — División de ISA flattening en subtablas (patrón híbrido)

Cada especialización con atributos propios se separa en una tabla subentidad con `Lote_id` (o equivalente) como PK y FK al supertipo. **Solo se traslada a la subtabla el atributo propio del subtipo** (p. ej. la fecha de vencimiento).

**Decisión de diseño — los discriminadores `ES_*` se conservan en el supertipo.** A diferencia del patrón ISA puro (donde la pertenencia se infiere por la mera presencia de la fila en la subtabla), el modelo vigente mantiene los flags booleanos `ES_*` en el supertipo. Es un patrón híbrido **supertipo-con-flag + subtabla 1:1** que se conserva en el modelo físico (`modelo_relacional_3fn.md`) y en la implementación (`schema.ts`). Se mantienen porque:

- Permiten validar la pertenencia al subtipo de forma declarativa, sin un JOIN a la subtabla.
- Habilitan reglas que atan la especialización a otros atributos del propio supertipo: en `Venta`, el flag se liga al `Venta_Metodo_Pago` mediante un `CHECK` (`efectivo` ⇒ `ES_Venta_Efectivo`; `débito/crédito/transferencia` ⇒ `ES_Venta_Electronica`).
- En `Lote`, la coherencia del flag con la categoría (regla T1) no se resuelve eliminando el flag, sino enforzándola por **trigger** (`es_lote_perecible` debe coincidir con `Categoria_Exige_Vencimiento`).

**Cambios:**

| Supertipo | Subtablas nuevas | Atributo trasladado a la subtabla | Discriminadores conservados en el supertipo |
|---|---|---|---|
| `Lote` | `LotePerecible(Lote_id, Lote_Perecible_Fecha_Vencimiento)` | `Lote_Perecible_Fecha_Vencimiento` | `ES_Lote_Perecible`, `ES_Lote_No_Perecible` |
| `Contrasena` | `ContrasenaTemporal(Contrasena_id, Contrasena_Temporal_Fecha_Hora_Expiracion)` | `Contrasena_Temporal_Fecha_Hora_Expiracion` | `ES_Contrasena_Temporal`, `ES_Contrasena_Definitiva` |
| `Venta` | `VentaEfectivo(Venta_id, Venta_Efectivo_Monto_Recibido)` | `Venta_Efectivo_Monto_Recibido` (el `Monto_Vuelto` es derivado y ya se eliminó en la ronda 1) | `ES_Venta_Efectivo`, `ES_Venta_Electronica` |

**Notas:**

- `LoteNoPerecible`, `ContrasenaDefinitiva` y `VentaElectronica` no requieren tabla propia porque no aportan atributos adicionales: se identifican por su flag `ES_*` correspondiente (y por la ausencia de fila en la subtabla complementaria).
- La invariante "exactamente uno de los subtipos por instancia" se mantiene como `CHECK` constraint a nivel SQL: `CHECK(ES_<subtipo_a> + ES_<subtipo_b> = 1)`.

### Ronda 4 — Eliminación de snapshots por historización

Los snapshots son datos cuyo valor cambia con el tiempo en la entidad fuente pero deben quedar congelados en el momento del evento. La normalización 3FN exige reemplazarlos por referencias a tablas de historización con vigencia temporal.

**Cambios:**

#### 4.1 Historización de precios y categorías de producto

Se crea una tabla de versiones de precio. `DetalleVenta` referencia la versión vigente al momento de la venta en lugar de duplicar el precio.

```
HistorialPrecioProducto(Historial_Precio_Producto_id, Producto_id,
                        Historial_Precio_Costo, Historial_Precio_Venta,
                        Historial_Fecha_Hora_Vigencia_Desde,
                        Historial_Fecha_Hora_Vigencia_Hasta)
```

`DetalleVenta` cambia `Detalle_Venta_Precio_Unitario` por FK a `HistorialPrecioProducto`. Se elimina `Detalle_Venta_Snapshot_Categoria` porque la categoría se reconstruye trazando `Producto → Categoria` vía la versión histórica del producto si fuera necesario. Si la categoría también cambia con el tiempo, se aplica la misma técnica con `HistorialCategoriaProducto`. Por simplicidad, asumimos que el cambio de categoría de un producto es excepcional y se versiona en la misma `HistorialPrecioProducto` (o se acepta que la categoría se lea de `Producto.Categoria_id` actual).

#### 4.2 Versionado de usuario para auditoría

Se crea una tabla de versiones de usuario. `LogAuditoria` referencia la versión vigente del usuario al emitir el evento.

```
UsuarioVersion(Usuario_Version_id, Usuario_id,
               Usuario_Version_Nombre, Usuario_Version_Rol,
               Usuario_Version_Fecha_Hora_Vigencia_Desde,
               Usuario_Version_Fecha_Hora_Vigencia_Hasta)
```

`LogAuditoria` reemplaza `Log_Nombre_Usuario_Snapshot` y `Log_Rol_Snapshot` por FK a `UsuarioVersion`.

#### 4.3 Tasas legales como entidad

Las tasas (AFP, Salud, Cesantía) son datos legales con vigencia temporal. Se modelan como entidad propia.

```
TasaLegal(Tasa_Legal_id, Tasa_Legal_Tipo, Tasa_Legal_Valor,
          Tasa_Legal_Fecha_Vigencia_Desde,
          Tasa_Legal_Fecha_Vigencia_Hasta)

RemuneracionTasa(Remuneracion_id, Tasa_Legal_id)
```

`Remuneracion` elimina los campos `Remuneracion_Tasa_Afp_Aplicada`, `Remuneracion_Tasa_Salud_Aplicada`, `Remuneracion_Tasa_Cesantia_Aplicada`. La tabla asociativa `RemuneracionTasa` registra qué tasas se aplicaron a cada remuneración (N:M entre `Remuneracion` y `TasaLegal`). El monto líquido se calcula como vista.

---

## 3. Resumen de tablas resultantes (27 → 34 tablas)

> El modelo vigente eliminó la entidad `Cargo`; el conteo y los listados de esta sección ya no la incluyen.

**Tablas modificadas:**

`Producto`, `Lote`, `Merma`, `AjusteInventario`, `Categoria`, `Trabajador`, `Contrasena`, `Asistencia`, `Remuneracion`, `Venta`, `DetalleVenta`, `CierreCaja`, `LogAuditoria` (atributos eliminados y/o relaciones agregadas).

**Tablas nuevas (7):**

| # | Tabla nueva | Motivo |
|---|---|---|
| N1 | `LotePerecible` | Subtipo ISA de `Lote` (ronda 3) |
| N2 | `ContrasenaTemporal` | Subtipo ISA de `Contrasena` (ronda 3) |
| N3 | `VentaEfectivo` | Subtipo ISA de `Venta` (ronda 3) |
| N4 | `HistorialPrecioProducto` | Historización de precio para reemplazar snapshot en `DetalleVenta` (ronda 4) |
| N5 | `UsuarioVersion` | Versionado de usuario para auditoría (ronda 4) |
| N6 | `TasaLegal` | Entidad para tasas legales con vigencia temporal (ronda 4) |
| N7 | `RemuneracionTasa` | Relación N:M entre remuneración y tasas aplicadas (ronda 4) |

**Tablas inalteradas (15):** `Proveedor`, `ProveedorCategoria`, `PedidoProveedor`, `DetallePedido`, `HistorialAuditoriaPedido`, `Usuario`, `SesionUsuario`, `IntentoLogin`, `Turno`, `Ausencia`, `MermaLote`, `VentaLote`, `AnulacionVenta`, `LogErroresTecnicos`.

---

## 4. Verificación de 3FN en el esquema final

Para cada tabla del modelo final se verifica que toda dependencia funcional `X → A` cumpla: o bien `X` es superclave, o bien `A` es atributo primo (3FN de Zaniolo).

| Tabla | PK | Atributos no clave | ¿Hay dependencias transitivas o derivadas? | 3FN |
|---|---|---|---|---|
| `Categoria` | `Categoria_id` | `Categoria_Nombre`, `Categoria_Exige_Vencimiento` | No | ✓ |
| `Producto` | `Producto_id` | `Producto_Ean_13`, `Producto_Nombre`, `Producto_Precio_Venta`, `Producto_Stock_Minimo`, `Producto_Estado`, `Producto_Fecha_Registro`, `Categoria_id` | No | ✓ |
| `Lote` | `Lote_id` | `Lote_Cantidad_Inicial`, `Lote_Cantidad_Actual`, `Lote_Precio_Costo`, `Lote_Fecha_Hora_Ingreso`, `ES_Lote_Perecible`, `ES_Lote_No_Perecible`, `Producto_id`, `Proveedor_id`, `Pedido_Proveedor_id` | No (estado derivado eliminado; discriminadores `ES_*` conservados como atributos de la especialización, coherencia con categoría enforzada por trigger) | ✓ |
| `LotePerecible` | `Lote_id` (PK,FK) | `Lote_Perecible_Fecha_Vencimiento` | No | ✓ |
| `Merma` | `Merma_id` | `Merma_Motivo`, `Merma_Observacion`, `Merma_Fecha_Hora`, `Producto_id`, `Usuario_id` | No (cantidad eliminada, se recupera de MermaLote) | ✓ |
| `MermaLote` | `(Merma_id, Lote_id)` | `Merma_Lote_Cantidad_Descontada` | Depende de la PK completa | ✓ |
| `AjusteInventario` | `Ajuste_Inventario_id` | `Ajuste_Cantidad`, `Ajuste_Justificacion`, `Ajuste_Fecha_Hora`, `Producto_id`, `Lote_id`, `Usuario_id` | No | ✓ |
| `Proveedor` | `Proveedor_id` | 5 atributos directos | No | ✓ |
| `ProveedorCategoria` | `(Proveedor_id, Categoria_id)` | (sin atributos no clave) | n/a | ✓ |
| `PedidoProveedor` | `Pedido_Proveedor_id` | 4 atributos directos + 3 FK | No | ✓ |
| `DetallePedido` | `(Pedido_Proveedor_id, Producto_id)` | `Cantidad_Solicitada`, `Cantidad_Recibida` | Depende de PK completa | ✓ |
| `HistorialAuditoriaPedido` | `Historial_Auditoria_Pedido_id` | 3 atributos directos + 2 FK | No | ✓ |
| `Trabajador` | `Trabajador_id` | 7 atributos directos (Rut, Nombre, Apellido, Telefono, Correo, Fecha_Ingreso, Estado) | No (flags ISA y `Cargo_id` eliminados) | ✓ |
| `Usuario` | `Usuario_id` | `Usuario_Rol` ∈ {dueño, trabajador}, `Usuario_Fecha_Creacion`, `Usuario_Ultimo_Login_Fecha_Hora`, `Trabajador_id` (UNIQUE, 1:1) | No | ✓ |
| `UsuarioVersion` | `Usuario_Version_id` | `Usuario_id`, `Nombre`, `Rol`, vigencia | No | ✓ |
| `Contrasena` | `Contrasena_id` | `Hash`, `Fecha_Creacion`, `ES_Contrasena_Temporal`, `ES_Contrasena_Definitiva`, `Usuario_id`, `Generada_Por_Usuario_id` | No (estado derivado eliminado; discriminadores `ES_*` conservados con `CHECK`) | ✓ |
| `ContrasenaTemporal` | `Contrasena_id` (PK,FK) | `Contrasena_Temporal_Fecha_Hora_Expiracion` | No | ✓ |
| `SesionUsuario` | `Sesion_Usuario_id` | 4 atributos + `Usuario_id` | No | ✓ |
| `IntentoLogin` | `Intento_Login_id` | 3 atributos + `Usuario_id` | No | ✓ |
| `Turno` | `Turno_id` | 3 atributos + `Trabajador_id` | No | ✓ |
| `Asistencia` | `Asistencia_id` | `Fecha_Hora_Entrada`, `Fecha_Hora_Salida`, `Trabajador_id`, `Turno_id` | No (horas eliminadas) | ✓ |
| `Ausencia` | `Ausencia_id` | 4 atributos + 2 FK | No | ✓ |
| `Remuneracion` | `Remuneracion_id` | `Mes`, `Anio`, `Monto_Bruto`, `Observacion`, `Fecha_Registro`, 2 FK | No (tasas y líquido eliminados) | ✓ |
| `TasaLegal` | `Tasa_Legal_id` | `Tipo`, `Valor`, vigencia | No | ✓ |
| `RemuneracionTasa` | `(Remuneracion_id, Tasa_Legal_id)` | (sin atributos) | n/a | ✓ |
| `Venta` | `Venta_id` | `Fecha_Hora`, `Descuento_Tipo`, `Descuento_Valor`, `Descuento_Razon`, `Metodo_Pago`, `Estado`, `ES_Venta_Efectivo`, `ES_Venta_Electronica`, 2 FK | No (subtotal/total/vuelto derivados eliminados; atributo del subtipo movido a `VentaEfectivo`; discriminadores `ES_*` conservados con `CHECK` ligado a `Metodo_Pago`) | ✓ |
| `VentaEfectivo` | `Venta_id` (PK,FK) | `Venta_Efectivo_Monto_Recibido` | No | ✓ |
| `DetalleVenta` | `(Venta_id, Producto_id)` | `Cantidad`, `Historial_Precio_Producto_id` | Depende de PK completa | ✓ |
| `HistorialPrecioProducto` | `Historial_Precio_Producto_id` | `Producto_id`, `Precio_Costo`, `Precio_Venta`, vigencia | No | ✓ |
| `VentaLote` | `(Venta_id, Lote_id)` | `Venta_Lote_Cantidad_Consumida` | Depende de PK completa | ✓ |
| `AnulacionVenta` | `Anulacion_Venta_id` | `Fecha_Hora`, `Razon`, `Venta_id`, `Usuario_id` | No | ✓ |
| `CierreCaja` | `Cierre_Caja_id` | `Fecha_Hora_Inicio`, `Fecha_Hora_Fin`, `Estado`, `Usuario_Cierre_id` | No (todos los totales eliminados, se calculan por agregación) | ✓ |
| `LogAuditoria` | `Log_Auditoria_id` | `Fecha_Hora`, `Tipo_Accion`, `Modulo`, `Descripcion`, `Usuario_Version_id` | No (snapshots reemplazados por FK a UsuarioVersion) | ✓ |
| `LogErroresTecnicos` | `Log_ErrorTecnicos_id` | 4 atributos + `Usuario_id` | No | ✓ |

**Conclusión:** Las 34 tablas del esquema final cumplen 3FN.

---

## 5. Trade-offs del esquema normalizado

| Aspecto | Modelo original (no-3FN) | Modelo normalizado (3FN) |
|---|---|---|
| Cantidad de tablas | 27 | 34 (+7) |
| Joins en reportes | Pocos (datos pre-agregados) | Muchos (recálculo por consulta) |
| Riesgo de inconsistencia | Alto (atributos derivados pueden quedar desactualizados) | Bajo (la fuente única de verdad es atómica) |
| Performance de lectura | Mejor (sin agregaciones) | Peor sin vistas materializadas |
| Performance de escritura | Peor (triggers o transacciones para mantener derivados) | Mejor (inserciones simples sin propagación) |
| Inmutabilidad de auditoría | Resuelta por snapshots inline | Resuelta por versionado temporal explícito |
| Cumplimiento académico de 3FN | No | Sí |

**Recomendación práctica para producción:** mantener el esquema 3FN para la base transaccional y materializar las agregaciones críticas (cierre de caja, stock actual) como **vistas materializadas** con refresh por trigger. Así se obtiene la integridad de 3FN sin sacrificar el desempeño de reportes.

---

## 6. Modelo relacional final

El esquema completo en notación relacional plana (formato `modelo_relacional.docx`) se entrega en el archivo paralelo:

- `modelo_relacional_3fn.docx`
- `modelo_relacional_3fn.md` (versión equivalente en Markdown)

El diagrama físico se entrega en:

- `modelo_relacional_3fn.drawio`
