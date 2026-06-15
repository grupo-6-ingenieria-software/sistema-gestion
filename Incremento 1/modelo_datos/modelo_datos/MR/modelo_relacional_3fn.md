# MODELO FÍSICO RELACIONAL EN 3FN
## Sistema de Gestión Minimarket y Panadería Huáscar

> Tipos simples: `INTEGER`, `VARCHAR`, `DECIMAL`, `DATE`, `TIMESTAMP`, `BOOLEAN`.
> Snake_case en minúsculas con prefijo de entidad.

### Convención de IDs

- **Entidades maestras / catálogos** (`categoria`, `producto`, `proveedor`, `trabajador`, `tasa_legal`): PK `INTEGER` secuencial.
- **Eventos con fecha-hora** (ventas, mermas, lotes, sesiones, logs, etc.): PK `VARCHAR(36)` (UUID v4). Se justifica porque dos eventos pueden ocurrir en el mismo instante (concurrencia entre dispositivos / sync offline) y no pueden colisionar en ID.
- **Usuario**: PK `VARCHAR(50)` derivada del RUT del trabajador (regla de negocio, no cambia).
- **Tablas intermedias (M:N)**: PK propia `VARCHAR(36)` (UUID v4) + FKs a las tablas adyacentes + `UNIQUE` sobre el par de FKs. Se prefiere esta convención sobre la PK compuesta clásica porque:
  - Permite ser referenciada desde otras tablas con una sola columna (ej. `lote` apunta a `detalle_pedido.detalle_pedido_id`).
  - Mantiene consistencia con el resto del modelo donde todo registro tiene ID propio.
  - El `UNIQUE` sobre el par natural conserva la semántica de "una fila por par".

---

## MÓDULO 1 – INVENTARIO

```sql
CREATE TABLE categoria (
    categoria_id                  INTEGER      PRIMARY KEY,
    categoria_nombre              VARCHAR(50)  NOT NULL UNIQUE,
    categoria_exige_vencimiento   BOOLEAN      NOT NULL
);

CREATE TABLE producto (
    producto_id              INTEGER       PRIMARY KEY,
    producto_ean_13          VARCHAR(13)   NOT NULL UNIQUE,
    producto_nombre          VARCHAR(100)  NOT NULL,
    producto_precio_venta    INTEGER       NOT NULL,
    producto_stock_minimo    INTEGER       NOT NULL,
    producto_estado          VARCHAR(10)   NOT NULL,
    producto_fecha_registro  TIMESTAMP     NOT NULL,
    categoria_id             INTEGER       NOT NULL REFERENCES categoria(categoria_id)
);

CREATE TABLE historial_precio_producto (
    historial_precio_producto_id         VARCHAR(36)  PRIMARY KEY,
    historial_precio_costo               INTEGER      NOT NULL,
    historial_precio_venta               INTEGER      NOT NULL,
    historial_fecha_hora_vigencia_desde  TIMESTAMP    NOT NULL,
    historial_fecha_hora_vigencia_hasta  TIMESTAMP,
    producto_id                          INTEGER      NOT NULL REFERENCES producto(producto_id)
);

CREATE TABLE lote (
    lote_id                  VARCHAR(36)  PRIMARY KEY,
    lote_cantidad_inicial    INTEGER      NOT NULL,
    lote_cantidad_actual     INTEGER      NOT NULL,
    lote_precio_costo        INTEGER      NOT NULL,
    lote_fecha_hora_ingreso  TIMESTAMP    NOT NULL,
    es_lote_perecible        BOOLEAN      NOT NULL,   -- flag ISA (discriminador)
    es_lote_no_perecible     BOOLEAN      NOT NULL,   -- flag ISA (discriminador)
    producto_id              INTEGER      NOT NULL REFERENCES producto(producto_id),
    proveedor_id             INTEGER               REFERENCES proveedor(proveedor_id),
    pedido_proveedor_id      VARCHAR(36)           REFERENCES pedido_proveedor(pedido_proveedor_id),
    CHECK (es_lote_perecible + es_lote_no_perecible = 1)
);

-- Subtipo ISA de lote (1:1)
CREATE TABLE lote_perecible (
    lote_id                           VARCHAR(36)  PRIMARY KEY REFERENCES lote(lote_id),
    lote_perecible_fecha_vencimiento  DATE         NOT NULL
);

CREATE TABLE merma (
    merma_id           VARCHAR(36)   PRIMARY KEY,
    merma_motivo       VARCHAR(20)   NOT NULL,
    merma_observacion  VARCHAR(200),
    merma_fecha_hora   TIMESTAMP     NOT NULL,
    producto_id        INTEGER       NOT NULL REFERENCES producto(producto_id),
    usuario_id         VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id)
);

-- Intermedia M:N (merma <-> lote) con PK propia
CREATE TABLE merma_lote (
    merma_lote_id                   VARCHAR(36)  PRIMARY KEY,
    merma_id                        VARCHAR(36)  NOT NULL REFERENCES merma(merma_id),
    lote_id                         VARCHAR(36)  NOT NULL REFERENCES lote(lote_id),
    merma_lote_cantidad_descontada  INTEGER      NOT NULL,
    UNIQUE (merma_id, lote_id)
);

CREATE TABLE ajuste_inventario (
    ajuste_inventario_id   VARCHAR(36)   PRIMARY KEY,
    ajuste_cantidad        INTEGER       NOT NULL,
    ajuste_justificacion   VARCHAR(300)  NOT NULL,
    ajuste_fecha_hora      TIMESTAMP     NOT NULL,
    producto_id            INTEGER       NOT NULL REFERENCES producto(producto_id),
    lote_id                VARCHAR(36)   NOT NULL REFERENCES lote(lote_id),
    usuario_id             VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id)
);
```

---

## MÓDULO 2 – PROVEEDORES

```sql
CREATE TABLE proveedor (
    proveedor_id                    INTEGER       PRIMARY KEY,
    proveedor_rut                   VARCHAR(12)   NOT NULL UNIQUE,
    proveedor_nombre_razon_social   VARCHAR(100)  NOT NULL,
    proveedor_nombre_contacto       VARCHAR(80)   NOT NULL,
    proveedor_telefono              VARCHAR(9)    NOT NULL,
    proveedor_correo_electronico    VARCHAR(120)  NOT NULL
);

-- Intermedia M:N (proveedor <-> categoria) con PK propia
CREATE TABLE proveedor_categoria (
    proveedor_categoria_id  VARCHAR(36)  PRIMARY KEY,
    proveedor_id            INTEGER      NOT NULL REFERENCES proveedor(proveedor_id),
    categoria_id            INTEGER      NOT NULL REFERENCES categoria(categoria_id),
    UNIQUE (proveedor_id, categoria_id)
);

CREATE TABLE pedido_proveedor (
    pedido_proveedor_id                    VARCHAR(36)   PRIMARY KEY,
    pedido_proveedor_fecha_hora_emision    TIMESTAMP     NOT NULL,
    pedido_proveedor_estado                VARCHAR(25)   NOT NULL,
    pedido_proveedor_fecha_hora_recepcion  TIMESTAMP,
    pedido_proveedor_nota_recepcion        VARCHAR(500),
    proveedor_id                           INTEGER       NOT NULL REFERENCES proveedor(proveedor_id),
    usuario_emisor_id                      VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id),
    usuario_receptor_id                    VARCHAR(50)            REFERENCES usuario(usuario_id)
);

-- Intermedia M:N (pedido_proveedor <-> producto) con PK propia
CREATE TABLE detalle_pedido (
    detalle_pedido_id     VARCHAR(36)  PRIMARY KEY,
    pedido_proveedor_id   VARCHAR(36)  NOT NULL REFERENCES pedido_proveedor(pedido_proveedor_id),
    producto_id           INTEGER      NOT NULL REFERENCES producto(producto_id),
    cantidad_solicitada   INTEGER      NOT NULL,
    cantidad_recibida     INTEGER,
    UNIQUE (pedido_proveedor_id, producto_id)
);

CREATE TABLE historial_auditoria_pedido (
    historial_auditoria_pedido_id  VARCHAR(36)   PRIMARY KEY,
    historial_ap_tipo_evento       VARCHAR(25)   NOT NULL,
    historial_ap_fecha_hora        TIMESTAMP     NOT NULL,
    historial_ap_nota              VARCHAR(500),
    pedido_proveedor_id            VARCHAR(36)   NOT NULL REFERENCES pedido_proveedor(pedido_proveedor_id),
    usuario_id                     VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id)
);
```

---

## MÓDULO 4 – TRABAJADORES Y ACCESO

```sql
CREATE TABLE trabajador (
    trabajador_id                  INTEGER       PRIMARY KEY,
    trabajador_rut                 VARCHAR(12)   NOT NULL UNIQUE,
    trabajador_nombre              VARCHAR(50)   NOT NULL,
    trabajador_apellido            VARCHAR(50)   NOT NULL,
    trabajador_telefono            VARCHAR(9)    NOT NULL,
    trabajador_correo_electronico  VARCHAR(120),
    trabajador_fecha_ingreso       DATE          NOT NULL,
    trabajador_estado              VARCHAR(10)   NOT NULL
);

CREATE TABLE usuario (
    usuario_id                       VARCHAR(50)  PRIMARY KEY,
    usuario_rol                      VARCHAR(15)  NOT NULL,
    usuario_fecha_creacion           TIMESTAMP    NOT NULL,
    usuario_ultimo_login_fecha_hora  TIMESTAMP,
    trabajador_id                    INTEGER      NOT NULL UNIQUE REFERENCES trabajador(trabajador_id)
);

CREATE TABLE usuario_version (
    usuario_version_id                         VARCHAR(36)   PRIMARY KEY,
    usuario_version_nombre                     VARCHAR(101)  NOT NULL,
    usuario_version_rol                        VARCHAR(15)   NOT NULL,
    usuario_version_fecha_hora_vigencia_desde  TIMESTAMP     NOT NULL,
    usuario_version_fecha_hora_vigencia_hasta  TIMESTAMP,
    usuario_id                                 VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id)
);

CREATE TABLE contrasena (
    contrasena_id                  VARCHAR(36)   PRIMARY KEY,
    contrasena_hash                VARCHAR(255)  NOT NULL,
    contrasena_fecha_hora_creacion TIMESTAMP     NOT NULL,
    es_contrasena_temporal         BOOLEAN       NOT NULL,   -- flag ISA (discriminador)
    es_contrasena_definitiva       BOOLEAN       NOT NULL,   -- flag ISA (discriminador)
    usuario_id                     VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id),
    generada_por_usuario_id        VARCHAR(50)            REFERENCES usuario(usuario_id),
    CHECK (es_contrasena_temporal + es_contrasena_definitiva = 1)
);

-- Subtipo ISA de contrasena (1:1)
CREATE TABLE contrasena_temporal (
    contrasena_id                              VARCHAR(36)  PRIMARY KEY REFERENCES contrasena(contrasena_id),
    contrasena_temporal_fecha_hora_expiracion  TIMESTAMP    NOT NULL
);

CREATE TABLE sesion_usuario (
    sesion_usuario_id                VARCHAR(36)  PRIMARY KEY,
    sesion_fecha_hora_inicio         TIMESTAMP    NOT NULL,
    sesion_fecha_hora_ultimo_acceso  TIMESTAMP    NOT NULL,
    sesion_fecha_hora_cierre         TIMESTAMP,
    sesion_motivo_cierre             VARCHAR(15),
    usuario_id                       VARCHAR(50)  NOT NULL REFERENCES usuario(usuario_id)
);

CREATE TABLE intento_login (
    intento_login_id                  VARCHAR(36)  PRIMARY KEY,
    intento_nombre_usuario_ingresado  VARCHAR(50)  NOT NULL,
    intento_fecha_hora                TIMESTAMP    NOT NULL,
    intento_exitoso                   BOOLEAN      NOT NULL,
    usuario_id                        VARCHAR(50)           REFERENCES usuario(usuario_id)
);

CREATE TABLE turno (
    turno_id                 VARCHAR(36)  PRIMARY KEY,
    turno_fecha_hora_inicio  TIMESTAMP    NOT NULL,
    turno_fecha_hora_fin     TIMESTAMP    NOT NULL,
    turno_estado             VARCHAR(15)  NOT NULL,
    trabajador_id            INTEGER      NOT NULL REFERENCES trabajador(trabajador_id)
);

CREATE TABLE asistencia (
    asistencia_id                  VARCHAR(36)  PRIMARY KEY,
    asistencia_fecha_hora_entrada  TIMESTAMP    NOT NULL,
    asistencia_fecha_hora_salida   TIMESTAMP,
    trabajador_id                  INTEGER      NOT NULL REFERENCES trabajador(trabajador_id),
    turno_id                       VARCHAR(36)           REFERENCES turno(turno_id)
);

CREATE TABLE ausencia (
    ausencia_id                  VARCHAR(36)   PRIMARY KEY,
    ausencia_fecha               DATE          NOT NULL,
    ausencia_tipo                VARCHAR(15)   NOT NULL,
    ausencia_observacion         VARCHAR(200),
    ausencia_fecha_hora_registro TIMESTAMP     NOT NULL,
    trabajador_id                INTEGER       NOT NULL REFERENCES trabajador(trabajador_id),
    usuario_registrador_id       VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id),
    UNIQUE (trabajador_id, ausencia_fecha)
);

CREATE TABLE remuneracion (
    remuneracion_id                   VARCHAR(36)   PRIMARY KEY,
    remuneracion_mes                  INTEGER       NOT NULL,
    remuneracion_anio                 INTEGER       NOT NULL,
    remuneracion_monto_bruto          INTEGER       NOT NULL,
    remuneracion_observacion          VARCHAR(200),
    remuneracion_fecha_hora_registro  TIMESTAMP     NOT NULL,
    trabajador_id                     INTEGER       NOT NULL REFERENCES trabajador(trabajador_id),
    usuario_registrador_id            VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id),
    UNIQUE (trabajador_id, remuneracion_anio, remuneracion_mes)
);

CREATE TABLE tasa_legal (
    tasa_legal_id                    INTEGER      PRIMARY KEY,
    tasa_legal_tipo                  VARCHAR(15)  NOT NULL,
    tasa_legal_valor                 DECIMAL      NOT NULL,
    tasa_legal_fecha_vigencia_desde  DATE         NOT NULL,
    tasa_legal_fecha_vigencia_hasta  DATE
);

-- Intermedia M:N (remuneracion <-> tasa_legal) con PK propia
CREATE TABLE remuneracion_tasa (
    remuneracion_tasa_id  VARCHAR(36)  PRIMARY KEY,
    remuneracion_id       VARCHAR(36)  NOT NULL REFERENCES remuneracion(remuneracion_id),
    tasa_legal_id         INTEGER      NOT NULL REFERENCES tasa_legal(tasa_legal_id),
    UNIQUE (remuneracion_id, tasa_legal_id)
);
```

---

## MÓDULO 5 – VENTAS

```sql
CREATE TABLE cierre_caja (
    cierre_caja_id            VARCHAR(36)  PRIMARY KEY,
    cierre_fecha_hora_inicio  TIMESTAMP    NOT NULL UNIQUE,
    cierre_estado             VARCHAR(10)  NOT NULL,
    cierre_fecha_hora_fin     TIMESTAMP,
    usuario_cierre_id         VARCHAR(50)           REFERENCES usuario(usuario_id)
);

CREATE TABLE venta (
    venta_id                VARCHAR(36)  PRIMARY KEY,
    venta_fecha_hora        TIMESTAMP    NOT NULL,
    venta_descuento_tipo    VARCHAR(15)  NOT NULL,
    venta_descuento_valor   INTEGER,
    venta_descuento_razon   VARCHAR(100),
    venta_metodo_pago       VARCHAR(15)  NOT NULL,
    venta_estado            VARCHAR(15)  NOT NULL,
    es_venta_efectivo       BOOLEAN      NOT NULL,   -- flag ISA (discriminador)
    es_venta_electronica    BOOLEAN      NOT NULL,   -- flag ISA (discriminador)
    usuario_cajero_id       VARCHAR(50)  NOT NULL REFERENCES usuario(usuario_id),
    cierre_caja_id          VARCHAR(36)  NOT NULL REFERENCES cierre_caja(cierre_caja_id),
    CHECK ((venta_metodo_pago = 'efectivo' AND es_venta_efectivo = 1 AND es_venta_electronica = 0)
        OR (venta_metodo_pago <> 'efectivo' AND es_venta_efectivo = 0 AND es_venta_electronica = 1))
);

-- Subtipo ISA de venta (1:1)
CREATE TABLE venta_efectivo (
    venta_id                       VARCHAR(36)  PRIMARY KEY REFERENCES venta(venta_id),
    venta_efectivo_monto_recibido  INTEGER      NOT NULL
);

-- Intermedia M:N (venta <-> producto) con PK propia
CREATE TABLE detalle_venta (
    detalle_venta_id              VARCHAR(36)  PRIMARY KEY,
    venta_id                      VARCHAR(36)  NOT NULL REFERENCES venta(venta_id),
    producto_id                   INTEGER      NOT NULL REFERENCES producto(producto_id),
    detalle_venta_cantidad        INTEGER      NOT NULL,
    historial_precio_producto_id  VARCHAR(36)  NOT NULL REFERENCES historial_precio_producto(historial_precio_producto_id),
    UNIQUE (venta_id, producto_id)
);

-- Intermedia M:N (venta <-> lote) con PK propia
CREATE TABLE venta_lote (
    venta_lote_id                  VARCHAR(36)  PRIMARY KEY,
    venta_id                       VARCHAR(36)  NOT NULL REFERENCES venta(venta_id),
    lote_id                        VARCHAR(36)  NOT NULL REFERENCES lote(lote_id),
    venta_lote_cantidad_consumida  INTEGER      NOT NULL,
    UNIQUE (venta_id, lote_id)
);

CREATE TABLE anulacion_venta (
    anulacion_venta_id    VARCHAR(36)   PRIMARY KEY,
    anulacion_fecha_hora  TIMESTAMP     NOT NULL,
    anulacion_razon       VARCHAR(200)  NOT NULL,
    venta_id              VARCHAR(36)   NOT NULL UNIQUE REFERENCES venta(venta_id),
    usuario_id            VARCHAR(50)   NOT NULL REFERENCES usuario(usuario_id)
);
```

---

## MÓDULO 7 – LOGS Y AUDITORÍA

```sql
CREATE TABLE log_auditoria (
    log_auditoria_id    VARCHAR(36)   PRIMARY KEY,
    log_fecha_hora      TIMESTAMP     NOT NULL,
    log_tipo_accion     VARCHAR(40)   NOT NULL,
    log_modulo          VARCHAR(50)   NOT NULL,
    log_descripcion     VARCHAR(500)  NOT NULL,
    usuario_version_id  VARCHAR(36)   NOT NULL REFERENCES usuario_version(usuario_version_id)
);

CREATE TABLE log_errores_tecnicos (
    log_errortecnicos_id             VARCHAR(36)    PRIMARY KEY,
    log_errores_fecha_hora           TIMESTAMP      NOT NULL,
    log_errores_tipo_error           VARCHAR(60)    NOT NULL,
    log_errores_modulo               VARCHAR(50)    NOT NULL,
    log_errores_descripcion_tecnica  VARCHAR(2000)  NOT NULL,
    usuario_id                       VARCHAR(50)             REFERENCES usuario(usuario_id)
);
```

---

## NOTAS

1. **Módulos 3, 6 y 8** no generan tablas (sólo consultas/vistas o lógica de hardware).
2. **Atributos derivados** (stock_actual, subtotal, total, vuelto, etc.) se calculan en consultas, no se almacenan.
3. **Orden de creación** sugerido: `categoria`, `trabajador`, `usuario`, `proveedor`, `pedido_proveedor`, luego el resto (por referencias FK).
4. **Montos** en `INTEGER` (pesos chilenos sin decimales). **Tasas** en `DECIMAL`.
5. **IDs `VARCHAR(36)`**: corresponden a eventos con fecha-hora donde dos registros podrían generarse en el mismo instante (concurrencia, sync offline). Se recomienda UUID v4 para evitar colisiones.
6. **Tablas intermedias** usan PK propia `VARCHAR(36)` + `UNIQUE` sobre el par de FKs (ver "Convención de IDs" al inicio). Esto las hace referenciables desde otras tablas y mantiene la semántica de unicidad del par.
