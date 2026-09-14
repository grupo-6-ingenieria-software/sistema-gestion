# Árbol de navegación — Incremento 2

Sistema de Gestión Operativa — Minimarket y Panadería Huáscar.

## Alcance y lectura

Transcripción del árbol general visible en `Arbol_de_Navegacion_Incremento_2.drawio.png`, extraída de los datos originales de draw.io incluidos en el PNG. Conserva nombres, rutas, códigos visibles, incrementos, roles y dirección de las flechas.

- Roles: `D` = DUEÑO; `T` = TRABAJADOR; `D/T` = ambos roles.
- `—` = dato no indicado en el árbol visible.
- `Desde` enumera los nodos con una flecha directa hacia esa pantalla.
- Los códigos repetidos se conservan tal como aparecen. Las rutas mantienen literalmente `:ean13`, `:rut` y `:id`.
- Una flecha sin etiqueta no especifica condiciones adicionales ni una conexión de regreso.

## Acceso

| Pantalla | Código | Ruta | Incremento | Roles |
| --- | --- | --- | --- | --- |
| Login | V01 | `/login` | 1 | — |
| Cambio de contraseña | — | `/cambiar-contrasena` | 1 | D/T |
| Shell autenticado | — | `/app` | — | D/T |

Conexiones de acceso:

- Login → Cambio de contraseña.
- Sistema de Gestión Operativa → Shell autenticado.
- Login → Shell autenticado: Login válido.
- Cambio de contraseña → Shell autenticado: Cambio completado.

## Módulos del shell autenticado

El shell `/app` tiene una flecha directa hacia cada uno de estos módulos: Inicio, Inventario, Proveedores, Personal, Ventas, Caja y Administración. Los primeros seis muestran DUEÑO y TRABAJADOR; Administración muestra solo DUEÑO. Las cabeceras de módulo no indican una ruta propia.

### Inicio

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Dashboard | V05 | `/app/inicio` | 1 | D/T | Inicio |

### Inventario

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Productos | V06 | `/app/inventario/productos` | 1 | D/T | Inventario |
| Detalle de producto | V20 | `/app/inventario/productos/:ean13/detalle` | 2 | D/T | Productos |
| Historial de movimientos | V22 | `/app/inventario/movimientos` | 2 | D/T | Detalle de producto; Inventario |
| Nuevo producto | — | `/app/inventario/productos/nuevo` | 1 | D | Productos |
| Editar producto | — | `/app/inventario/productos/:ean13/editar` | 1 | D | Productos |
| Cambiar estado de producto | — | `/app/inventario/productos/:ean13/estado` | 1 | D/T | Productos |
| Eliminar producto | — | `/app/inventario/productos/eliminar` | 1 | D/T | Productos |
| Registrar lote | — | `/app/inventario/lotes/nuevo` | 1 | D | Inventario |
| Registrar merma | — | `/app/inventario/mermas/nueva` | 1 | D/T | Inventario |
| Ajuste manual de stock | V21 | `/app/inventario/ajustes` | 2 | D/T | Inventario |
| Reabastecimiento | V25 | `/app/inventario/reabastecimiento` | 2 | D/T | Inventario; Proveedores |
| Valor de inventario | V28 | `/app/inventario/valorizacion` | 2 | D/T | Inventario |

### Proveedores

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Listado de proveedores | V23 | `/app/proveedores/listado` | 2 | D/T | Proveedores |
| Registrar proveedor | V24 | `/app/proveedores/nuevo` | 2 | D/T | Listado de proveedores |
| Editar proveedor | V24 | `/app/proveedores/:rut/editar` | 2 | D/T | Listado de proveedores |
| Registrar pedido | V26 | `/app/proveedores/pedidos/nuevo` | 2 | D/T | Proveedores |
| Recepciones e historial de pedidos | V27 | `/app/proveedores/pedidos` | 2 | D/T | Proveedores |

### Personal

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Trabajadores | V15 | `/app/personal/trabajadores` | 1 | D/T | Personal |
| Registrar trabajador | V16 | `/app/personal/trabajadores/nuevo` | 1 | D | Trabajadores |
| Editar trabajador | V16 | `/app/personal/trabajadores/:rut/editar` | 1 | D | Trabajadores |
| Estado del trabajador | V29 | `/app/personal/trabajadores/:rut/estado` | 2 | D | Trabajadores |
| Calendario de turnos | V17 | `/app/personal/turnos` | 1 | D/T | Personal |
| Crear turno | V18 | `/app/personal/turnos/nuevo` | 1 | D | Calendario de turnos |
| Editar turno | V18 | `/app/personal/turnos/:id/editar` | 1 | D | Calendario de turnos |
| Asistencia | V19 | `/app/personal/asistencia` | 1 | D/T | Personal |
| Registrar remuneración | V30 | `/app/personal/remuneraciones/nueva` | 2 | D | Personal |
| Configuración previsional | V31 | `/app/personal/configuracion-previsional` | 2 | D | Personal |

### Ventas

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Registrar venta | V12 | `/app/ventas/registrar` | 1 | D/T | Ventas |
| Ventas del día | V13 | `/app/ventas/dia` | 1 | D/T | Ventas |
| Consulta de ventas | V33 | `/app/ventas/consulta` | 2 | D/T | Ventas |
| Anular venta | V32 | `/app/ventas/anular` | 2 | D/T | Ventas; Ventas del día; Consulta de ventas |

### Caja

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Cierre de caja | — | `/app/caja/cierre` | 1 | D/T | Caja |

### Administración

| Pantalla | Código | Ruta | Incremento | Roles | Desde |
| --- | --- | --- | --- | --- | --- |
| Usuarios | — | `/app/admin/usuarios` | 1 | D | Administración |
| Auditoría | — | `/app/admin/auditoria` | 1 | D | Administración |

## Información no especificada en el árbol visible

- Condición para pasar de Login a Cambio de contraseña.
- Pantalla de destino inicial al entrar en `/app`.
- Comportamiento ante acceso denegado, sesión expirada o cierre de sesión.
- Si cada destino se presenta como página, modal u otro componente.

Estas definiciones quedan sin completar; el diagrama no permite determinarlas.
