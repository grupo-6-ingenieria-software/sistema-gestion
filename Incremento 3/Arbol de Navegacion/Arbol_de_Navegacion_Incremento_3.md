# Árbol de navegación

Sistema de Gestión Operativa — Minimarket y Panadería Huáscar.

## Alcance y lectura

Transcripción de la página general del diagrama editable. Las páginas 2 a 8 detallan los módulos y sus controladores.

- Roles: `D` = DUEÑO; `T` = TRABAJADOR; `D/T` = ambos roles.
- `Desde` enumera los nodos con una flecha directa hacia esa pantalla.
- Los códigos Vxx identifican las vistas representadas en el diagrama.

## Acceso

| Pantalla | Código | Ruta | Roles |
| --- | --- | --- | --- |
| Login | V01 | `/login` | — |
| Cambio de contraseña | V02 | `/cambiar-contrasena` | D/T |
| Shell autenticado | — | `/app` | D/T |

Conexiones de acceso:

- Login → Cambio de contraseña (contraseña temporal, incluida la generada por un restablecimiento de CU59).
- Sistema de Gestión Operativa → Shell autenticado.
- Login → Shell autenticado: Login válido.
- Cambio de contraseña → Shell autenticado: Cambio completado.

## Módulos del shell autenticado

El shell `/app` tiene una flecha directa hacia cada uno de estos ocho módulos: Inicio, Inventario, Proveedores, Personal, Ventas, Caja, Reportes y Administración. Los seis primeros muestran DUEÑO y TRABAJADOR; Reportes y Administración muestran solo DUEÑO. Las cabeceras de módulo no tienen ruta propia.

### Inicio

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Dashboard | V05 | `/app/inicio` | D/T | Inicio |

### Inventario

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Productos | V06 | `/app/inventario/productos` | D/T | Inventario |
| Detalle de producto | V20 | `/app/inventario/productos/:ean13/detalle` | D/T | Productos |
| Historial de movimientos | V22 | `/app/inventario/movimientos` | D/T | Inventario; Detalle de producto |
| Nuevo producto | V07 | `/app/inventario/productos/nuevo` | D | Productos |
| Editar producto | V07 | `/app/inventario/productos/:ean13/editar` | D | Productos |
| Cambiar estado de producto | V08 | `/app/inventario/productos/:ean13/estado` | D/T | Productos |
| Eliminar producto | V09 | `/app/inventario/productos/eliminar` | D/T | Productos |
| Registrar lote | V10 | `/app/inventario/lotes/nuevo` | D | Inventario; Productos |
| Registrar merma | V11 | `/app/inventario/mermas/nueva` | D/T | Inventario; Productos |
| Ajuste manual de stock | V21 | `/app/inventario/ajustes` | D/T | Inventario |
| Reabastecimiento | V25 | `/app/inventario/reabastecimiento` | D/T | Inventario; Proveedores |
| Valor de inventario | V28 | `/app/inventario/valorizacion` | D/T | Inventario |

### Proveedores

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Listado de proveedores | V23 | `/app/proveedores/listado` | D/T | Proveedores |
| Registrar proveedor | V24 | `/app/proveedores/nuevo` | D/T | Listado de proveedores |
| Editar proveedor | V24 | `/app/proveedores/:rut/editar` | D/T | Listado de proveedores |
| Registrar pedido | V26 | `/app/proveedores/pedidos/nuevo` | D/T | Proveedores; Listado de proveedores; Reabastecimiento |
| Recepciones e historial de pedidos | V27 | `/app/proveedores/pedidos` | D/T | Proveedores; Listado de proveedores; Registrar pedido |

### Personal

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Trabajadores | V15 | `/app/personal/trabajadores` | D/T | Personal |
| Registrar trabajador | V16 | `/app/personal/trabajadores/nuevo` | D | Trabajadores |
| Editar trabajador | V16 | `/app/personal/trabajadores/:rut/editar` | D | Trabajadores |
| Estado del trabajador | V29 | `/app/personal/trabajadores/:rut/estado` | D | Trabajadores |
| Calendario de turnos | V17 | `/app/personal/turnos` | D/T | Personal |
| Crear turno | V18 | `/app/personal/turnos/nuevo` | D | Calendario de turnos |
| Editar turno | V18 | `/app/personal/turnos/:id/editar` | D | Calendario de turnos |
| Asistencia | V19 | `/app/personal/asistencia` | D/T | Personal |
| Registrar ausencia | V34 | `/app/personal/ausencias/nueva` | D | Personal; Trabajadores (Dueño) |
| Resumen mensual de asistencia | V36 | `/app/personal/asistencia/resumen-mensual` | D | Personal; Trabajadores o Asistencia (Dueño) |
| Registrar remuneración | V30 | `/app/personal/remuneraciones/nueva` | D | Personal |
| Configuración previsional | V31 | `/app/personal/configuracion-previsional` | D | Personal |

### Ventas

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Registrar venta | V12 | `/app/ventas/registrar` | D/T | Ventas |
| Ventas del día | V13 | `/app/ventas/dia` | D/T | Ventas |
| Consulta de ventas | V33 | `/app/ventas/consulta` | D/T | Ventas |
| Anular venta | V32 | `/app/ventas/anular` | D/T | Ventas; Ventas del día; Consulta de ventas |
| Ventas por categoría | V35 | `/app/ventas/categorias` | D/T (T: solo propias) | Ventas |

### Caja

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Cierre de caja | V14 | `/app/caja/cierre` | D/T | Caja |

### Reportes

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Ventas diarias | V37 | `/app/reportes/ventas-diarias` | D | Reportes |
| Ventas mensuales | V38 | `/app/reportes/ventas-mensuales` | D | Reportes |
| Productos más vendidos | V39 | `/app/reportes/mas-vendidos` | D | Reportes |
| Mermas | V40 | `/app/reportes/mermas` | D | Reportes |
| Lotes próximos a vencer | V41 | `/app/reportes/lotes-por-vencer` | D | Reportes |
| Rentabilidad por categoría | V42 | `/app/reportes/rentabilidad` | D | Reportes |
| Asistencia del personal | V43 | `/app/reportes/asistencia-personal` | D | Reportes |
| Movimientos de inventario | V44 | `/app/reportes/movimientos-inventario` | D | Reportes |

### Administración

| Pantalla | Código | Ruta | Roles | Desde |
| --- | --- | --- | --- | --- |
| Usuarios | V03 | `/app/admin/usuarios` | D | Administración |
| Auditoría | V04 | `/app/admin/auditoria` | D | Administración |

## Reglas de navegación

1. El Trabajador no ve el módulo Reportes ni Administración. Si entra por ruta directa: acceso denegado, vuelve al Dashboard y C04 registra el intento (CU58 E1).
2. En Ventas por categoría el Trabajador solo ve sus propias ventas; el filtro lo aplica el proceso principal, no la vista.
3. Exportar (UI06), confirmaciones, gráficos, vistas de impresión UI07–UI15 y UI01 son componentes internos: no tienen ruta ni nodo.
4. Ventas por categoría (V35) y Resumen mensual (V36) son consultas y no exportan; la exportación existe en V06 y en V37–V44.
5. Restablecer contraseña no tiene ruta propia: es una acción dentro de Usuarios; la persona afectada pasa por Cambio de contraseña en su próximo login.
6. Toda ruta bajo `/app` exige sesión vigente (C05) y permiso por operación (C03).
