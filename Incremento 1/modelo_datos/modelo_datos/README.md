# Modelo de Datos — Minimarket y Panadería Huáscar

Documentación del modelo de datos del sistema de gestión, derivado de los requerimientos funcionales (RF01–RF61) y no funcionales (RNF01–RNF21).

## Estructura

```
.
├── README.md
└── modelo_datos/
    ├── modelo_datos.md       # diccionario completo de entidades, atributos, relaciones y restricciones
    ├── requerimientos.md     # tabla de requerimientos RF/RNF (83 requerimientos)
    └── casos_uso.md          # 68 casos de uso extendidos
```

## Contenido de `modelo_datos.md`

Define el modelo conceptual-relacional con trazabilidad completa a RF y CU. Incluye:

- **Convenciones metodológicas** — PK surrogadas, relaciones M:N con atributos, especialización ISA con flattening, cardinalidad
- **8 módulos funcionales** — Inventario, Proveedores, Valorización, Trabajadores/Asistencia, Ventas, Reportes, Dashboard/Control de Acceso, Lector de Código de Barras
- **Entidades principales** — `Producto`, `Categoria`, `Lote`, `Merma`, `AjusteInventario`, `Proveedor`, `PedidoProveedor`, `Trabajador`, `Usuario`, `Contrasena`, `Turno`, `Asistencia`, `Ausencia`, `Remuneracion`, `Venta`, `AnulacionVenta`, `CierreCaja`, `LogAuditoria`, `LogErroresTecnicos`, más entidades de soporte inferidas
- **Relaciones N:M con atributos** — `ProveedorCategoria`, `DetallePedido`, `DetalleVenta`, `VentaLote`, `MermaLote`, `RemuneracionTasa`
- **Especializaciones ISA** (patrón híbrido: flags `ES_*` en el supertipo + subtabla 1:1) — `Lote` (perecible / no perecible), `Contrasena` (temporal / definitiva), `Venta` (efectivo / electrónica)
- **Matriz consolidada de relaciones** por módulo
- **Resumen de entidades inferidas** con su justificación

> **Decisiones de modelado vigentes:** se eliminó la entidad `Cargo` y la especialización ISA de `Trabajador` (todo trabajador tiene un `Usuario` y el acceso lo determina `Usuario.rol`). La antigua entidad `MovimientoInventario` se reemplazó por `AjusteInventario` más las relaciones N:M `VentaLote` y `MermaLote`. La fuente de verdad es el **modelo físico** `MR/modelo_relacional_3fn.md` (34 tablas en 3FN).

## Estado

El modelo de datos está documentado y trazado a requerimientos, normalizado a 3FN (ver `analisis_3fn.md`) y materializado en el modelo físico `MR/modelo_relacional_3fn.md`. Diagramas en `MERE/` (entidad-relación), `MR/` (relacional) y `modelo_fisico/` (físico), en formato `.drawio` y PNG.
