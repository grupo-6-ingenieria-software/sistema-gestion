# Arbol de navegacion - Incremento 1

## Proposito

Este documento define el arbol maestro de navegacion para el Incremento 1 del Sistema de Gestion Operativa del Minimarket y Panaderia Huascar. Su objetivo es servir como contrato directo para implementar rutas, menu lateral/superior, permisos por rol y comportamiento de guardas antes de programar las vistas.

El alcance se ajusta a las vistas `V01` a `V19` definidas en `vistas y controladores/Vistas_y_Controladores good.docx` y al documento `documento 0/Grupo 6 - Incremento 1.docx`. El cambio principal frente a la version anterior es la incorporacion del `RF03 / CU3`, que agrega la eliminacion de producto como vista navegable `V09 EliminarProductoView` y controlador `C25 EliminarProductoHandler`.

Los componentes `UI01`, `UI02` y `UI03` no tienen ruta propia.

## Roles

| Rol tecnico | Nombre visible | Alcance |
| --- | --- | --- |
| `dueno` | Dueno | Acceso completo al Incremento 1. |
| `trabajador` | Trabajador | Acceso operativo a inicio, inventario permitido, ventas, caja y asistencia. |

## Arbol maestro

```text
/
|- /login                                      V01 LoginView
|- /cambiar-contrasena                         V02 CambioContrasenaObligatorioView
`- /app                                        Shell autenticado
   |- /app/inicio                              V05 DashboardView                 dueno, trabajador
   |- Inventario
   |  |- /app/inventario/productos             V06 ListadoProductosView           dueno, trabajador
   |  |- /app/inventario/productos/nuevo       V07 FormularioProductoView         dueno
   |  |- /app/inventario/productos/:ean13/editar
   |  |                                        V07 FormularioProductoView         dueno
   |  |- /app/inventario/productos/:ean13/estado
   |  |                                        V08 GestionEstadoProductoView      dueno, trabajador
   |  |- /app/inventario/productos/:ean13/eliminar
   |  |                                        V09 EliminarProductoView           dueno, trabajador
   |  |- /app/inventario/lotes/nuevo           V10 RegistrarLoteView              dueno
   |  `- /app/inventario/mermas/nueva          V11 RegistrarMermaView             dueno, trabajador
   |- Ventas
   |  |- /app/ventas/registrar                 V12 RegistrarVentaView             dueno, trabajador
   |  `- /app/ventas/dia                       V13 ListadoVentasDiaView           dueno, trabajador
   |- Caja
   |  `- /app/caja/cierre                      V14 CierreCajaView                 dueno, trabajador
   |- Personal
   |  |- /app/personal/trabajadores            V15 ListadoTrabajadoresView        dueno
   |  |- /app/personal/trabajadores/nuevo      V16 FormularioTrabajadorView       dueno
   |  |- /app/personal/turnos                  V17 CalendarioTurnosView           dueno
   |  |- /app/personal/turnos/nuevo            V18 CrearTurnoView                 dueno
   |  `- /app/personal/asistencia              V19 RegistroAsistenciaView         dueno, trabajador
   `- Administracion
      |- /app/admin/usuarios                   V03 GestionUsuariosView            dueno
      `- /app/admin/auditoria                  V04 LogAuditoriaView               dueno
```

## Nodos de navegacion

| Codigo | Vista | Ruta | Grupo | Roles | Menu | Entrada |
| --- | --- | --- | --- | --- | --- | --- |
| V01 | LoginView | `/login` | Publico | Sin sesion | No | Pantalla inicial sin sesion |
| V02 | CambioContrasenaObligatorioView | `/cambiar-contrasena` | Publico | Usuario con clave temporal | No | Redireccion posterior a login temporal |
| V05 | DashboardView | `/app/inicio` | Inicio | `dueno`, `trabajador` | Si | Login correcto o accion Inicio |
| V06 | ListadoProductosView | `/app/inventario/productos` | Inventario | `dueno`, `trabajador` | Si | Menu Inventario > Productos |
| V07 | FormularioProductoView | `/app/inventario/productos/nuevo` | Inventario | `dueno` | No | Accion Nuevo producto desde V06 |
| V07 | FormularioProductoView | `/app/inventario/productos/:ean13/editar` | Inventario | `dueno` | No | Accion Editar producto desde V06 |
| V08 | GestionEstadoProductoView | `/app/inventario/productos/:ean13/estado` | Inventario | `dueno`, `trabajador` | No | Accion Cambiar estado desde V06 |
| V09 | EliminarProductoView | `/app/inventario/productos/:ean13/eliminar` | Inventario | `dueno`, `trabajador` | No | Accion Eliminar producto desde V06 |
| V10 | RegistrarLoteView | `/app/inventario/lotes/nuevo` | Inventario | `dueno` | Si | Menu Inventario > Registrar lote, o V06 con `?ean13=` |
| V11 | RegistrarMermaView | `/app/inventario/mermas/nueva` | Inventario | `dueno`, `trabajador` | Si | Menu Inventario > Registrar merma, o V06 con `?ean13=` |
| V12 | RegistrarVentaView | `/app/ventas/registrar` | Ventas | `dueno`, `trabajador` | Si | Menu Ventas > Registrar venta |
| V13 | ListadoVentasDiaView | `/app/ventas/dia` | Ventas | `dueno`, `trabajador` | Si | Menu Ventas > Ventas del dia |
| V14 | CierreCajaView | `/app/caja/cierre` | Caja | `dueno`, `trabajador` | Si | Menu Caja > Cierre de caja, o acceso desde Ventas |
| V15 | ListadoTrabajadoresView | `/app/personal/trabajadores` | Personal | `dueno` | Si | Menu Personal > Trabajadores |
| V16 | FormularioTrabajadorView | `/app/personal/trabajadores/nuevo` | Personal | `dueno` | No | Accion Registrar trabajador desde V15 |
| V17 | CalendarioTurnosView | `/app/personal/turnos` | Personal | `dueno` | Si | Menu Personal > Turnos, o acceso desde V15 |
| V18 | CrearTurnoView | `/app/personal/turnos/nuevo` | Personal | `dueno` | No | Accion Crear turno desde V17 |
| V19 | RegistroAsistenciaView | `/app/personal/asistencia` | Personal | `dueno`, `trabajador` | Si | Menu Personal > Asistencia, dashboard o V15 |
| V03 | GestionUsuariosView | `/app/admin/usuarios` | Administracion | `dueno` | Si | Menu Administracion > Usuarios |
| V04 | LogAuditoriaView | `/app/admin/auditoria` | Administracion | `dueno` | Si | Menu Administracion > Log de auditoria |

## Reglas de navegacion

1. `/` redirige a `/login` si no existe sesion valida.
2. `/` redirige a `/app/inicio` si existe sesion valida y no hay cambio obligatorio de contrasena.
3. Login correcto con contrasena definitiva redirige a `/app/inicio`.
4. Login correcto con contrasena temporal redirige a `/cambiar-contrasena`.
5. Mientras exista cambio obligatorio de contrasena, bloquear toda navegacion excepto `/cambiar-contrasena`.
6. Toda ruta bajo `/app` exige validacion de sesion mediante `C05`.
7. Toda ruta bajo `/app` exige validacion de rol mediante `C03`.
8. Si el rol no tiene permiso, mostrar acceso denegado, registrar evento con `C04` y redirigir a `/app/inicio`.
9. Las rutas con `:ean13` deben validar el parametro antes de ejecutar acciones sensibles.
10. `V10` y `V11` pueden recibir `?ean13=` para abrir el formulario con producto preseleccionado.
11. `V09` debe confirmar eliminacion y bloquearla si el producto tiene ventas, mermas, lotes o movimientos asociados.
12. El shell autenticado no se numera como vista y no debe tener controlador propio.
13. `UI01`, `UI02` y `UI03` no deben aparecer en router ni menu.

## Guardas y controladores

| Regla | Controlador esperado | Uso |
| --- | --- | --- |
| Sesion activa e inactividad | `C05 SesionHandler` | Antes de abrir rutas privadas. |
| Cambio obligatorio de contrasena | `C02 PasswordHandler` | Despues del login y antes de permitir acceso al shell. |
| Permiso por rol | `C03 ControlAccesoMiddleware` | Antes de abrir vistas o ejecutar acciones sensibles. |
| Auditoria de intento denegado | `C04 AuditoriaHandler` | Cuando se bloquea una ruta por rol insuficiente. |
| Eliminacion de producto | `C25 EliminarProductoHandler` | Al confirmar `V09`, validando inexistencia de registros asociados. |

## Componentes sin ruta

| Codigo | Componente | Uso | Controlador |
| --- | --- | --- | --- |
| UI01 | CampoEAN13Input | Embebido en `V06`, `V07`, `V09`, `V10`, `V11` y `V12` | `C24` |
| UI02 | ResumenVentasDashboard | Embebido en `V05` | `C09` |
| UI03 | SeccionesPagoVenta | Embebido en `V12` | `C16` |

## Exclusiones del Incremento 1

No crear rutas para reportes, proveedores, pedidos, remuneraciones, ajustes avanzados de inventario, anulacion operativa, descuentos manuales, busqueda historica extendida, detalle extendido de producto, exportaciones ni funciones avanzadas del lector EAN.

## Criterios de aceptacion

- El Dueno ve los grupos Inicio, Inventario, Ventas, Caja, Personal y Administracion.
- El Trabajador ve Inicio, Inventario operativo, Ventas, Caja y Asistencia.
- `V09 EliminarProductoView` queda incorporada para cubrir `RF03 / CU3`.
- `V07`, `V08`, `V09`, `V10`, `V11`, `V16`, `V18` y `V19` tienen ruta explicita y entrada declarada.
- `UI01`, `UI02` y `UI03` no aparecen en menu ni router.
- Todas las rutas privadas tienen roles declarados y controladores asociados.
- El arbol completo declara vistas navegables `V01` a `V19` y controladores hasta `C25`.
