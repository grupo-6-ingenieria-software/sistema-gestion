**MÓDULO 1 – INVENTARIO**

Producto(**Producto\_id**, Producto\_Ean\_13, Producto\_Nombre, Producto\_Precio\_Costo, Producto\_Precio\_Venta, Producto\_Stock\_Minimo, Producto\_Stock\_Actual, Producto\_Estado, Producto\_Fecha\_Registro, Categoria\_id)

Categoria(**Categoria\_id**, Categoria\_Nombre)

Lote(**Lote\_id**, Lote\_Cantidad\_Inicial, Lote\_Cantidad\_Actual, Lote\_Precio\_Costo, Lote\_Fecha\_Hora\_Ingreso, Lote\_Estado, ES\_Lote\_Perecible, Lote\_Perecible, ES\_Lote\_No\_Perecible, Lote\_No\_Perecible, Lote\_Perecible\_Fecha\_Vencimiento, Producto\_id, Proveedor\_id, Pedido\_Proveedor\_id)

Merma(**Merma\_id**, Merma\_Cantidad, Merma\_Motivo, Merma\_Observacion, Merma\_Fecha\_Hora, Producto\_id, Usuario\_id)

MermaLote(**Merma\_id**, **Lote\_id**, Merma\_Lote\_Cantidad\_Descontada)

AjusteInventario(**Ajuste\_Inventario\_id**, Ajuste\_Cantidad, Ajuste\_Justificacion, Ajuste\_Stock\_Resultante, Ajuste\_Fecha\_Hora, Producto\_id, Lote\_id, Usuario\_id)

**MÓDULO 2 – PROVEEDORES**

Prooveedor(**Proveedor\_id**, Proveedor\_Rut, Proveedor\_Nombre\_Razon\_Social, Proveedor\_Nombre\_Contacto, Proveedor\_Telefono, Proveedor\_Correo\_Electronico)

ProveedorCategoria(**Proveedor\_id**, **Categoria\_id**)

PedidoProveedor(**Pedido\_Proveedor\_id**, Pedido\_Proveedor\_Fecha\_Hora\_Emision, Pedido\_Proveedor\_Estado, Pedido\_Proveedor\_Fecha\_Hora\_Recepcion, Pedido\_Proveedor\_Nota\_Recepcion, Proveedor\_id, Usuario\_Emisor\_id, Usuario\_Receptor\_id)

DetallePedido(**Pedido\_Proveedor\_id**, **Producto\_id**, Cantidad\_Solicitada, Cantidad\_Recibida)

HistorialAuditoriaPedido(**Historial\_Auditoria\_Pedido\_id**, Historial\_Ap\_Tipo\_Evento, Historial\_Ap\_Fecha\_Hora, Historial\_Ap\_Nota, Pedido\_Proveedor\_id, Usuario\_id)

**MÓDULO 4 – TRABAJADORES Y ACCESO**

Trabajador(**Trabajador\_id**, Trabajador\_Rut, Trabajador\_Nombre, Trabajador\_Apellido, Trabajador\_Telefono, Trabajador\_Correo\_Electronico, Trabajador\_Fecha\_Ingreso, Trabajador\_Estado)

Usuario(**Usuario\_id**, Usuario\_Rol, Usuario\_Fecha\_Creacion,   
Usuario\_Ultimo\_Login\_Fecha\_Hora, Trabajador\_id)

Contrasena(**Contrasena\_id**, Contrasena\_Hash, Contrasena\_Fecha\_Hora\_Creacion, Contrasena\_Estado, ES\_Contrasena\_Definitiva, Contrasena\_Definitiva, ES\_Contrasena\_Temporal, Contrasena\_Temporal, Contrasena\_Temporal\_Fecha\_Hora\_Expiracion, Usuario\_id, Generada\_Por\_Usuario\_id)

SesionUsuario(**Sesion\_Usuario\_id**, Sesion\_Fecha\_Hora\_Inicio, Sesion\_Fecha\_Hora\_Ultimo\_Acceso, Sesion\_Fecha\_Hora\_Cierre, Sesion\_Motivo\_Cierre, Usuario\_id)

IntentoLogin(**Intento\_Login\_id**, Intento\_Nombre\_Usuario\_Ingresado, Intento\_Fecha\_Hora, Intento\_Exitoso, Usuario\_id)Turno(**Turno\_id**, Turno\_Fecha\_Hora\_Inicio, Turno\_Fecha\_Hora\_Fin, Turno\_Estado, Trabajador\_id)

Asistencia(**Asistencia\_id**, Asistencia\_Fecha\_Hora\_Entrada, Asistencia\_Fecha\_Hora\_Salida, Asistencia\_Horas\_Trabajadas, Trabajador\_id, Turno\_id)

Ausencia(**Ausencia\_id**, Ausencia\_Fecha, Ausencia\_Tipo, Ausencia\_Observacion, Ausencia\_Fecha\_Hora\_Registro, Trabajador\_id, Usuario\_Registrador\_id)  
Remuneracion(**Remuneracion\_id**, Remuneracion\_Mes, Remuneracion\_Anio, Remuneracion\_Monto\_Bruto, Remuneracion\_Tasa\_Afp\_Aplicada, Remuneracion\_Tasa\_Salud\_Aplicada, Remuneracion\_Tasa\_Cesantia\_Aplicada, Remuneracion\_Monto\_Liquido, Remuneracion\_Observacion, Remuneracion\_Fecha\_Hora\_Registro, Trabajador\_id, Usuario\_Registrador\_id)

**MÓDULO 5 – VENTAS**

Venta(**Venta\_id**, Venta\_Fecha\_Hora, Venta\_Subtotal, Venta\_Descuento\_Tipo, Venta\_Descuento\_Valor, Venta\_Descuento\_Razon, Venta\_Total, Venta\_Metodo\_Pago, Venta\_Estado, ES\_Venta\_Efectivo, Venta\_Efectivo, ES\_Venta\_Electronica, Venta\_Electronica, Venta\_Efectivo\_Monto\_Recibido, Venta\_Efectivo\_Monto\_Vuelto, Usuario\_Cajero\_id, Cierre\_Caja\_id)

DetalleVenta(**Venta\_id**, **Producto\_id**, Detalle\_Venta\_Cantidad, Detalle\_Venta\_Precio\_Unitario, Detalle\_Venta\_Subtotal\_Linea, Detalle\_Venta\_Snapshot\_Categoria)

VentaLote(**Venta\_id**, **Lote\_id**, Venta\_Lote\_Cantidad\_Consumida)

AnulacionVenta(**Anulacion\_Venta\_id**, Anulacion\_Fecha\_Hora, Anulacion\_Razon, Venta\_id, Usuario\_id)

CierreCaja(**Cierre\_Caja\_id**, Cierre\_Fecha\_Hora\_Inicio, Cierre\_Estado, Cierre\_Fecha\_Hora\_Fin, Cierre\_Total\_Ventas, Cierre\_Total\_Efectivo, Cierre\_Total\_Debito, Cierre\_Total\_Credito, Cierre\_Total\_Transferencia, Cierre\_Cantidad\_Anulaciones, Cierre\_Monto\_Anulaciones, Usuario\_Cierre\_id)

**MÓDULO 7 – LOGS Y AUDITORÍA**

LogAuditoria(**Log\_Auditoria\_id**, Log\_Nombre\_Usuario\_Snapshot, Log\_Rol\_Snapshot, Log\_Fecha\_Hora, Log\_Tipo\_Accion, Log\_Modulo, Log\_Descripcion, Usuario\_id)

LogErroresTecnicos(**Log\_Error	Tecnicos\_id**, Log\_Errores\_Fecha\_Hora, Log\_Errores\_Tipo\_Error, Log\_Errores\_Modulo, Log\_Errores\_Descripcion\_Tecnica, Usuario\_id)