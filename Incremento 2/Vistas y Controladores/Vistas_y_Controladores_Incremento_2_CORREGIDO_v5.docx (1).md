**Sistema de Gestión Operativa**

**Minimarket y Panadería Huáscar**

*Documento de Vistas y Controladores \- Incremento 2 (revisión de acuerdos del 10 de septiembre de 2026)*

# **1\. Alcance y criterio de continuidad**

Este documento define el diseño acordado para los 25 RF del Incremento 2. Su fuente funcional es [Grupo 6 - Incremento 2.docx](<diagramas de secuencias/Grupo 6 - Incremento 2.docx>), actualizado con los acuerdos de la revisión. La base técnica vigente de I1 es `implementacion-sistema-gestion/`, commit `9f790f6`, inspeccionado sin modificar código. La presencia de una función acredita código existente, no aceptación funcional ni cobertura completa del RF.

Los identificadores Vxx y Cxx se conservan para trazabilidad. No son posiciones del arreglo `controllers` ni exigen un archivo o registro IPC por identificador. C26-C48 identifican responsabilidades del diseño de I2: unas se crean y otras se amplían dentro de los controladores de I1. Los participantes C_ son alias técnicos de los diagramas, no clases envoltorio cuya existencia se dé por probada.

Criterio de vista: igual que en Incremento 1, solo se numeran como Vxx las pantallas navegables o subpantallas principales. Widgets, modales, diálogos de confirmación y campos compartidos se documentan como componentes internos, no como vistas independientes. Los RNF se documentan como condiciones transversales; no se crean elementos adicionales por el solo hecho de enumerarlos.

Se crean las responsabilidades C26-C36, C44-C45 y C47. C37-C43, C46 y C48 se reutilizan o amplían en los módulos existentes según las secciones correspondientes. RF43 se reutiliza; RF44 integra las anulaciones y RF57 requiere ampliar los permisos acordados. RF45 sigue fuera del incremento. El cierre de saldo es una variante de RF18/CU18b, no un RF nuevo.

| RF del Incremento 2 cubiertos |
| :---- |
| RF07, RF11, RF12, RF13, RF14, RF15, RF16, RF17, RF18, RF19, RF22, RF23, RF24, RF26, RF27, RF28, RF31, RF34, RF35, RF37, RF38, RF41, RF43, RF44, RF57 |

# **2\. Resumen modular**

| \# | Módulo | Vistas navegables | Controladores del equipo (N: nuevo / R: reutilizado) | CU cubiertos |
| :---- | :---- | :---- | :---- | :---- |
| 8 | Inventario \- Detalle, Ajustes y Trazabilidad | V20-V22 | N: C26-C28 / R: C03, C04, C05, C13, C14 | CU7, CU11, CU12 |
| 9 | Gestión de Proveedores y Reabastecimiento | V23-V27 | N: C29-C35 / R: C03, C04, C05, C13 | CU13-C18, CU18b |
| 10 | Inventario \- Valorización | V28 | N: C36 / R: C03, C05 | CU19 |
| 11 | Gestión de Personal (extendido) | V15-V18, V29 | Adaptar/reutilizar C37-C42 dentro de C21/C22; apoyo C03-C05, C23 | CU22-CU24, CU26-CU28 |
| 12 | Remuneraciones y Configuración Previsional | V19, V30, V31 | Crear C44-C45; reutilizar/verificar C43 dentro de C23; apoyo C03-C05 | CU31, CU34, CU35 |
| 13 | Ventas - Descuentos, Anulación y Consulta | V12, V32, V33 | Crear C47; adaptar/reutilizar C46 en C16 y C48 en C18; apoyo C03-C05, C09, C19-C20 | CU37, CU38, CU41 |
| \- | Cobertura transversal heredada | V05, V12, V15-V19 | R: C03, C04, C05, C09, C13, C14, C16, C18, C21-C23 | CU43, CU44, CU57 y soporte |

# **3\. Catálogo de componentes no navegables**

| Cód. | Componente | Dónde se usa | Responsabilidad | RF/CU |
| :---- | :---- | :---- | :---- | :---- |
| UI01\* | CampoEAN13Input (reutilizado) | V21, V22 y V26 | Componente de Incremento 1 para captura manual o por lector, normalización y validación de formato/checksum EAN-13 antes de invocar el canal IPC de la vista. | RF11, RF12, RF17 / CU11, CU12, CU17 |
| UI02\* | ResumenVentasDashboard (reutilizado) | V05 | Widget del dashboard que consume C09 y se refresca cuando C16 registra o C47 anula una venta. No define ruta propia. | RF44 / CU44 |
| UI04 | DescuentoVentaModal | V12 | Organiza el monto y la razón usando la funcionalidad de descuento ya existente. El total es provisional hasta que C16/C46 lo revalida al registrar la venta. Sin ruta propia. | RF37 / CU37 |
| UI05 | ListaReabastecimientoPrintView | C33 | Componente React de impresión sin ruta de navegación, usado exclusivamente por webContents.printToPDF para generar el reporte PDF de reabastecimiento. | RF16 / CU16 |

# **4\. Controladores técnicos de la arquitectura usados en las secuencias**

El diagrama de componentes y los diagramas de secuencia muestran controladores técnicos que encapsulan seguridad, persistencia, eventos, exportación y sincronización. Se nombran directamente con el prefijo C\_ y se integran o configuran; no se contabilizan como nuevas responsabilidades funcionales Cxx. Preload/contextBridge, ipcMain.handle y Electron Main se consideran infraestructura de comunicación y ejecución, no controladores.

| Controlador técnico de arquitectura | Tecnología encapsulada | Función y condición de implementación |
| :---- | :---- | :---- |
| C\_jsonwebtoken | jsonwebtoken | Emite y verifica JWT. C01 lo usa al autenticar; C03 y C05 lo usan para autorización y sesión. |
| C\_bcryptjs | bcryptjs | Compara de forma segura la contraseña durante la autenticación gestionada por C01. |
| C\_cryptoNode | node:crypto | Genera UUID solicitados por los controladores funcionales; las credenciales temporales conservan el mecanismo existente de autenticación. |
| C\_webContentsSend | webContents.send | Emite eventos desde main para sesión expirada y actualización de dashboard o listados. |
| C\_webContents.printToPDF | API nativa webContents.printToPDF | Genera la exportación PDF orquestada por C33 para RF16, usando el identificador solicitado para el controlador técnico. |
| C\_exceljs | exceljs | Genera la exportación Excel orquestada por C33 para RF16. |
| C\_drizzle | Drizzle ORM | Encapsula el acceso SQL tipado utilizado por los controladores funcionales. |
| C\_libsql | @libsql/client \+ libSQL local | Ejecuta consultas contra la base local y comunica la capa de persistencia. |
| C\_tursoSync | Turso Cloud / sincronización | Referencia arquitectónica: no se acredita sincronización local/remota por tener una conexión libSQL. Verificar por separado; no es requisito para ejecutar cada CU local. |

## 4.1 Separación entre elementos propios y arquitectura

Esta clasificación evita contar bibliotecas, procesos Electron y adaptadores del modelo como si fueran controladores de caso de uso. La columna Estado distingue lo existente de las integraciones por desarrollar; no afirma que las bibliotecas tengan una clase envoltorio propia.

| Tipo | Identificadores | Estado | Responsabilidad |
| :---- | :---- | :---: | :---- |
| Responsabilidades nuevas | C26-C36, C44-C45, C47 | Crear | Consultas, operaciones de dominio y exportaciones descritas en las secciones 5-10. |
| Responsabilidades existentes y ampliaciones | C03-C05, C09, C13-C16, C18-C23; C37-C43, C46 y C48 como responsabilidades internas | Reutilizar, verificar o extender | Conservar un único propietario por canal; no volver a registrar operaciones existentes. |
| Controladores técnicos de arquitectura | C\_jsonwebtoken, C\_bcryptjs, C\_cryptoNode, C\_webContentsSend, C\_webContents.printToPDF, C\_exceljs, C\_drizzle, C\_libsql, C\_tursoSync | Integrar/configurar | Encapsulan seguridad, UUID, eventos, exportación, persistencia y sincronización. |
| Mecanismos/procesos de infraestructura | Preload/contextBridge, ipcMain.handle, Electron Main | Usar como infraestructura | Transportan la solicitud y alojan la ejecución; no son controladores ni se registran en la matriz como tales. |

## 4.2 Patrón objetivo y correspondencia con la base vigente

**Autenticación (CU56).** V01 \-\> mecanismo IPC de Electron \-\> C01 AuthHandler. C01 consulta el modelo a través de C\_drizzle/C\_libsql, compara la contraseña mediante C\_bcryptjs y emite JWT mediante C\_jsonwebtoken; después registra el intento y crea SesionUsuario.

**Operación protegida.** Vista → preload/IPC → `authorizeRequest` (token, permiso por operación y sesión activa) → controlador de dominio → persistencia. Este chequeo de sesión ya existe en el commit inspeccionado. C03/C05 son responsabilidades reutilizadas, no una segunda cadena obligatoria de IPC entre handlers. RF22 exige que el rol efectivo de la sesión se conserve hasta el siguiente login; deben alinearse con ello los helpers que actualmente leen el rol de la cuenta. RF23 revoca sesiones al desactivar, aunque el token aún no haya expirado.

**Operación auditada.** El controlador funcional delega en C04 AuditoriaHandler. C04 usa C\_cryptoNode para el identificador y C\_drizzle/C\_libsql para persistir LogAuditoria; esos controladores técnicos no generan controladores funcionales Cxx adicionales.

# **5\. Módulo 8: Inventario \- Detalle, Ajustes y Trazabilidad**

**RF cubiertos: RF07, RF11, RF12**

## **Vistas navegables**

| Cód. | Vista | Entrada / navegación | Responsabilidad y datos requeridos | RF/CU | Rol(es) | Resultado esperado |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| V20 | DetalleProductoView | Desde V06: Ver detalle. | EAN-13, nombre, categoría, stock derivado, mínimo, precio de venta y costo vigente. Lista lotes con cantidad > 0, costo, proveedor y vencimiento aplicable; orden por vencimiento o ingreso. Ambos roles ven costos y proveedor. | RF07, RF57 / CU7 | Dueño y Trabajador | Ficha completa y lotes con existencias; lista vacía si no hay existencias. |
| V21 | AjusteStockView | Menú Inventario > Ajuste manual. | Buscar por EAN-13 o nombre con UI01. Seleccionar lote existente con existencias, agotado o vencido. Mostrar cantidad anterior, ajuste y resultado. Exigir ajuste entero distinto de cero, justificación y confirmación. | RF11 / CU11 | Dueño y Trabajador | Cantidad no negativa y ajuste trazable. Desechos se registran como merma. |
| V22 | HistorialMovimientosView | Menú Inventario > Historial, o V20. | UI01; ingresos, ventas, restituciones por anulación, mermas y ajustes con lote, fecha/hora, cantidad firmada, stock resultante y responsable. Calcular sobre toda la secuencia antes de filtrar. | RF12 / CU12 | Dueño y Trabajador | Historial reconciliado con stock actual; sin coincidencias: tabla vacía. |

## **Responsabilidades funcionales de Incremento 2 (crear o adaptar)**

| Cód. | Responsabilidad / propietario y canal IPC | Responsabilidad | Entradas principales | Salidas y errores esperados | RF/CU |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C26 | Crear DetalleProductoHandler: producto:detalle | Consultar Producto, Categoria, HistorialPrecioProducto, Lote, LotePerecible y Proveedor. Derivar stock por suma de cantidades actuales. No ocultar costos ni proveedor al Trabajador. | EAN-13, JWT | Ficha completa para ambos roles; producto inexistente: error; sin lotes con existencias: lista vacía. | RF07, RF57 / CU7 |
| C27 | Crear AjusteStockHandler: inventario:lotes-ajustables, inventario:ajustar-stock | Consultar producto y todos sus lotes, incluidos agotados/vencidos. En transacción revalidar pertenencia, cantidad actual, ajuste entero no cero y justificación; impedir resultado negativo, actualizar lote y registrar ajuste manual y C04. Conservar identidad, costo y vencimiento. Evento tras commit. | Producto/lote, ajuste, justificación, confirmación, JWT | Anterior, ajuste, resultado y stock derivado; errores de referencia, cantidad o justificación. | RF11 / CU11 |
| C28 | Crear HistorialMovimientosHandler: inventario:historial-movimientos | Unir Lote (ingreso una vez), VentaLote+Venta, AnulacionVenta+VentaLote (restitución en la fecha de anulación), MermaLote+Merma y ajustes manuales. Distinguir los ajustes de ingreso heredados para no sumarlos otra vez. Responsable desde el evento de recepción o ingreso estructuralmente asociado. Calcular antes de filtrar. | EAN-13, tipo, rango, JWT | Movimientos firmados y saldo reconciliado; datos heredados ambiguos se identifican sin inventar responsables. Ver sección 14. | RF12 / CU12 |

# **6\. Módulo 9: Gestión de Proveedores y Reabastecimiento**

**RF cubiertos: RF13, RF14, RF15, RF16, RF17, RF18**

## **Vistas navegables**

| Cód. | Vista | Entrada / navegación | Responsabilidad y datos requeridos | RF/CU | Rol(es) | Resultado esperado |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| V23 | ListadoProveedoresView | Menú Proveedores \> Listado. | Tabla con RUT, razón social, contacto, teléfono, correo y categorías. Búsqueda por nombre/RUT y filtro por categoría. Da acceso a V24-V27 según permiso. | RF15 / CU15 | Dueño y Trabajador | Listado filtrado o mensaje sin coincidencias. |
| V24 | FormularioProveedorView | Desde V23: Registrar o Editar. | Formulario reutilizado para alta y edición. En edición mantiene el RUT bloqueado; gestiona las categorías suministradas. | RF13, RF14 / CU13, CU14 | Dueño y Trabajador | Proveedor creado/actualizado o errores específicos de unicidad, obligatoriedad y formato. |
| V25 | ListaReabastecimientoView | Menú Inventario o Proveedores \> Reabastecimiento. | Lista productos con stock derivado menor o igual al mínimo. Cantidad sugerida \= max(0, 2 x stock mínimo \- stock actual). Exporta mediante UI05/C33 a PDF o XLSX. | RF16 / CU16 | Dueño y Trabajador | Lista y cantidad a solicitar correctas; archivo generado o mensaje de error. |
| V26 | RegistrarPedidoProveedorView | Menú Proveedores \> Registrar pedido, o desde V25. | Selecciona proveedor y una o más líneas. Reutiliza UI01; cada producto debe existir y estar activo, la cantidad es entera mayor a 0 y no se repite el producto. | RF17, RF04 / CU17 | Dueño y Trabajador | Pedido pendiente o errores por proveedor/producto, estado, duplicidad o cantidad. |
| V27 | ConfirmarRecepcionPedidoView | Menú Proveedores > Recepciones e historial de pedidos. | Listar por estado y abrir detalle/historial. Recibir pedidos pendientes o parciales abiertos; mostrar solicitado, recibido acumulado, faltante y cantidad de esta entrega, con costo y vencimiento por lote. Cancelar sin recepción o cerrar saldo parcial con motivo y confirmación. | RF18, RF57 / CU18, CU18b | Dueño y Trabajador | Estado y entregas trazables; cierre/cancelación no modifican inventario. |

## **Responsabilidades funcionales de Incremento 2 (crear o adaptar)**

| Cód. | Responsabilidad / propietario y canal IPC | Responsabilidad | Entradas principales | Salidas y errores esperados | RF/CU |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C29 | RegistrarProveedorHandler: proveedor:registrar | Valida obligatoriedad, RUT único, teléfono de 9 dígitos y correo; inserta Proveedor y ProveedorCategoria en una transacción y audita mediante C04. | RUT, razón social, contacto, teléfono, correo, categorías, JWT | Proveedor creado o errores por unicidad/formato/campo vacío. | RF13 / CU13 |
| C30 | EditarProveedorHandler: proveedor:editar | Mantiene Proveedor\_Rut inmutable, valida los campos y reemplaza de forma transaccional las asociaciones ProveedorCategoria; audita con C04. | RUT existente, campos editables, categorías, JWT | Proveedor actualizado o errores de existencia/formato. | RF14 / CU14 |
| C31 | Crear ConsultaProveedorHandler: proveedor:listar, proveedor:buscar-existente | Consultar proveedor y categorías con búsqueda/filtros. Reutilizar consulta interna de existencia desde C14/C34, sin encadenar canales IPC entre controladores. | Búsqueda, filtros o identidad, JWT | Lista o detalle/validación de proveedor. | RF15 / CU15 |
| C32 | Crear ReabastecimientoHandler: inventario:lista-reabastecimiento | Stock derivado, incluyendo productos sin lotes con stock cero; seleccionar stock <= mínimo y sugerir max(0, 2 x mínimo - stock). C33 reutiliza esta consulta en Main. | JWT | Nombre, EAN-13, categoría, stock, mínimo y sugerencia; lista vacía si no hay resultados. | RF16 / CU16 |
| C33 | Crear ExportacionReporteHandler: reporte:exportar-pdf, reporte:exportar-xlsx | Obtener la lista autorizada desde C32 en Main; generar PDF con UI05/printToPDF o XLSX con exceljs. No confiar en filas arbitrarias del renderer. | Formato, JWT | Archivo guardado, cancelación del guardado o error identificado. | RF16 / CU16 |
| C34 | Crear PedidoProveedorHandler: pedido:registrar | Validar proveedor, producto activo, cantidades enteras > 0 y ausencia de productos repetidos; guardar PedidoProveedor y DetallePedido en estado pendiente y auditar en transacción. Una línea puede originar lotes en varias entregas. | Proveedor, líneas EAN-13/cantidad, JWT | Pedido pendiente o errores de referencia/cantidad/duplicidad. Requiere adaptar enum actual. | RF17, RF04 / CU17 |
| C35 | Crear RecepcionPedidoHandler: pedido:listar, pedido:detalle, pedido:confirmar-recepcion, pedido:cancelar, pedido:cerrar-saldo | Ambos roles. Consultar detalle e historial. Revalidar estado y saldo dentro de la transacción; conservar cada entrega, sus líneas, responsable y lotes; acumular recibido y determinar estado. Cancelar solo sin recepciones; cerrar saldo conserva lo recibido y exige motivo. Auditar y emitir después del commit. | Pedido, identificador de operación, cantidades de esta entrega, costo/vencimiento; motivo/confirmación para cierre; JWT | Detalle, recibido, parcial, cancelado o parcial cerrado. Errores por estado, saldo o datos de lote. Ver sección 14. | RF18, RF57 / CU18, CU18b |

# **7\. Módulo 10: Inventario \- Valorización**

**RF cubiertos: RF19**

## **Vistas navegables**

| Cód. | Vista | Entrada / navegación | Responsabilidad y datos requeridos | RF/CU | Rol(es) | Resultado esperado |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| V28 | ValorizacionInventarioView | Menú Inventario > Valor de inventario. | Suma de cantidad actual x costo para lotes con existencias; desglose por categoría con productos distintos, unidades y valor. Es valorización, no rentabilidad. | RF19, RF57 / CU19 | Dueño y Trabajador | Total CLP; cero y tabla vacía sin existencias. |

## **Responsabilidades funcionales de Incremento 2 (crear o adaptar)**

| Cód. | Responsabilidad / propietario y canal IPC | Responsabilidad | Entradas principales | Salidas y errores esperados | RF/CU |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C36 | Crear ValorizacionHandler: inventario:valorizacion | Autorizar ambos roles. Agregar lotes con cantidad > 0, productos y categorías; calcular productos distintos, unidades y valores. Devolver importes numéricos; la vista formatea CLP. | JWT | Desglose y total; cero sin existencias. | RF19, RF57 / CU19 |

# **8\. Módulo 11: Gestión de Personal (Extendido)**

**RF cubiertos: RF22, RF23, RF24, RF26, RF27, RF28**

## **Vistas extendidas desde Incremento 1**

| Cód. | Vista (extiende Incremento 1\) | Qué agrega en Incremento 2 | RF/CU nuevo |
| :---- | :---- | :---- | :---- |
| V15\* | ListadoTrabajadoresView | Reutilizar listado, filtros y búsqueda presentes en I1. Consulta para ambos roles; alta, edición y estado solo Dueño, con acceso a V16/V29. | RF23, RF24 / CU23, CU24 |
| V16\* | FormularioTrabajadorView | Integrar el modo edición de datos existente en I1 sin duplicar validaciones. RUT bloqueado; registro y edición solo Dueño. Cambio de rol efectivo en el siguiente login. | RF22 / CU22 |
| V17\* | CalendarioTurnosView | Reutilizar semana, filtros y acciones existentes. Ambos roles consultan; solo Dueño abre V18 o confirma eliminación. | RF27, RF28 / CU27, CU28 |
| V18\* | FormularioTurnoView | Reutilizar creación y lógica de edición para turnos no iniciados y sin asistencia. Solo Dueño. | RF26 / CU26 |

## **Vistas navegables nuevas**

| Cód. | Vista | Entrada / navegación | Responsabilidad y datos requeridos | RF/CU | Rol(es) | Resultado esperado |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| V29 | GestionEstadoTrabajadorView | Desde V15: Cambiar estado. | Mostrar identidad y estado; confirmar activación/desactivación. Al desactivar, invalidar sesiones abiertas; al reactivar exigir nuevo login. Reutilizar operación existente de C21. | RF23 / CU23 | Dueño | Estado actualizado, sesiones revocadas cuando corresponda e historial conservado; cancelar no cambia datos. |

## **Responsabilidades funcionales de Incremento 2 (crear o adaptar)**

| Cód. | Responsabilidad / propietario y canal IPC | Responsabilidad | Entradas principales | Salidas y errores esperados | RF/CU |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C37 | Adaptar en C21 TrabajadorHandler: trabajador:actualizar | Solo Dueño. Actualizar datos y Usuario/UsuarioVersion cuando corresponda. Rol efectivo de la sesión hasta siguiente login; alinear guard y helpers que leen rol de base. RUT inmutable; auditar en transacción. | Contrato existente de trabajador, JWT | Datos actualizados o validación. No registrar un canal duplicado trabajador:editar. | RF22 / CU22 |
| C38 | Adaptar en C21 TrabajadorHandler: trabajador:cambiar-estado | Solo Dueño y con confirmación. Desactivar impide login, nuevos turnos/asistencia e invalida sesiones abiertas; Main rechaza operaciones posteriores y notifica al renderer. Conservar historial. Reactivar no reabre sesiones anteriores. Auditar en transacción. | Identidad, estado, confirmación, JWT | Estado actualizado y sesiones invalidadas; cancelación/error sin cambios. | RF23 / CU23 |
| C39 | Reutilizar/ampliar en C21: trabajador:listar | Consulta con nombre/RUT, rol y estado ya existente; incluir fecha de ingreso. Ambos roles consultan; mutaciones se autorizan separadamente. Mantener listado operativo trabajador:listar-activos. | Búsqueda, filtros, JWT | Listado o tabla vacía. | RF24 / CU24 |
| C40 | Reutilizar/verificar en C22: turno:editar | Solo Dueño; revalidar turno no iniciado y sin asistencia, fin posterior a inicio y no solapamiento para el mismo trabajador. Actualización y auditoría atómicas. | Turno, inicio/fin, JWT | Turno actualizado o error; conservar el canal existente. | RF26 / CU26 |
| C41 | Reutilizar/verificar en C22: turno:eliminar | Solo Dueño. Confirmar trabajador/horario y revalidar no iniciado/sin asistencia; eliminar y auditar atómicamente. | Turno, confirmación, JWT | Eliminación o error/cancelación. | RF27 / CU27 |
| C42 | Ampliar en C22: turno:listar | Calendario semanal existente de trabajadores activos, con filtro; permitir consulta a ambos roles sin otorgar edición. La lista de trabajadores para el filtro debe permitir seleccionar todos los activos sin cambiar el alcance de asistencia. | Semana, trabajador opcional, JWT | Calendario o vacío. No crear turno:listar-semana equivalente. | RF28 / CU28 |

# **9\. Módulo 12: Remuneraciones y Configuración Previsional**

**RF cubiertos: RF31, RF34, RF35**

## **Vistas extendidas desde Incremento 1**

| Cód. | Vista (extiende Incremento 1\) | Qué agrega en Incremento 2 | RF/CU nuevo |
| :---- | :---- | :---- | :---- |
| V19\* | RegistroAsistenciaView | Reutilizar cálculo de horas existente en C23 y verificar HH:MM tras salida, Pendiente si no hay salida. | RF31 / CU31 |

## **Vistas navegables nuevas**

| Cód. | Vista | Entrada / navegación | Responsabilidad y datos requeridos | RF/CU | Rol(es) | Resultado esperado |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| V30 | RegistrarRemuneracionView | Menú Personal \> Remuneraciones \> Registrar. | Selecciona trabajador activo o con actividad en el mes, mes/año, bruto y observación. Muestra tasas vigentes, descuentos y líquido derivado. | RF34 / CU34 | Dueño | Remuneración registrada o error por duplicidad, elegibilidad, monto o líquido negativo. |
| V31 | ConfigurarPorcentajesPrevisionalesView | Menú Personal > Configuración previsional. | Mostrar y modificar AFP, Salud y Cesantía. Defaults: 11,5%, 7% y 0,6%. Cambios vigentes al confirmar para remuneraciones registradas después. | RF35 / CU35 | Dueño | Configuración actualizada sin alterar históricos. |

## **Responsabilidades funcionales de Incremento 2 (crear o adaptar)**

| Cód. | Responsabilidad / propietario y canal IPC | Responsabilidad | Entradas principales | Salidas y errores esperados | RF/CU |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C43 | Reutilizar/verificar función interna de C23 | La base ya calcula horasTrabajadas. Diferencia entrada/salida de la misma fecha en HH:MM; salida ausente: Pendiente. No crear otro handler IPC. | Asistencia del día | HH:MM o Pendiente. | RF31 / CU31 |
| C44 | Crear RemuneracionHandler: remuneracion:trabajadores-elegibles, remuneracion:calcular, remuneracion:registrar | Solo Dueño. Conservar elegibilidad RF34: activo o actividad del mes, incluso desactivado; no imponer un nuevo tipo de asistencia. Validar período, bruto entero positivo según CU34, unicidad y líquido no negativo. Tasas al registrar; redondear cada descuento al peso más cercano, medio peso hacia arriba. Guardar bruto y referencias RemuneracionTasa con auditoría. | Trabajador, mes/año, bruto, observación, JWT | Elegibles, cálculo o remuneración; validación de monto, duplicidad o elegibilidad. | RF34 / CU34 |
| C45 | Crear ConfigPrevisionalHandler: configuracion:previsional-obtener, configuracion:previsional-actualizar | Solo Dueño. Validar valores numéricos en [0,100]; en transacción cerrar vigencia anterior y crear nueva al confirmar. No modificar porcentajes históricos ni admitir fechas arbitrarias futuras/retroactivas. Defaults por inicialización y auditoría de cambios. | Tipo(s), valor(es), JWT | Tasas vigentes sin alterar remuneraciones previas. | RF35 / CU35 |

# **10\. Módulo 13: Ventas \- Descuentos, Anulación y Consulta**

**RF cubiertos: RF37, RF38 y RF41 (reutiliza RF43, RF44 y RF57 de Incremento 1 \- ver sección 12\)**

## **Vistas extendidas desde Incremento 1**

| Cód. | Vista (extiende Incremento 1\) | Qué agrega en Incremento 2 | RF/CU nuevo |
| :---- | :---- | :---- | :---- |
| V12\* | RegistrarVentaView | Reutilizar descuento existente e integrar UI04; C16/C46 revalida y persiste al confirmar. | RF37 / CU37 |

## **Vistas navegables nuevas**

| Cód. | Vista | Entrada / navegación | Responsabilidad y datos requeridos | RF/CU | Rol(es) | Resultado esperado |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| V32 | AnularVentaView | Menú Ventas, V13 o V33. | Buscar venta del día, mostrar responsable, fecha, productos, descuento, método, total y estado de su caja; solicitar razón y confirmación. | RF38 / CU38 | Dueño y Trabajador | Anulación y restitución exacta solo si la caja asociada está abierta; si está cerrada, rechazo sin modificar datos. |
| V33 | ConsultaVentasView | Menú Ventas > Consulta. | Buscar por rango válido o número; listar vigentes/anuladas y detalle con precio histórico y efectivo/vuelto. Mostrar cantidad y monto de vigentes, y cantidad de anuladas por separado. | RF41 / CU41 | Dueño y Trabajador | Listado/detalle; anuladas conservan importe original pero no suman a vigentes; sin coincidencias: cero y lista vacía. |

## **Responsabilidades funcionales de Incremento 2 (crear o adaptar)**

| Cód. | Responsabilidad / propietario y canal IPC | Responsabilidad | Entradas principales | Salidas y errores esperados | RF/CU |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C46 | Reutilizar/ampliar función interna de C16: venta:registrar | El descuento ya se valida y persiste en sale-service. Revalidar monto entre cero y subtotal y razón del descuento solicitado, con total autoritativo en Main. UI04 usa total provisional. No crear otro IPC obligatorio de validación equivalente. | Venta en curso, monto, razón, JWT | Venta con descuento validado o error de rango/razón. | RF37 / CU37 |
| C47 | Crear AnulacionVentaHandler: venta:anular | En transacción validar venta vigente del día y su CierreCaja asociado abierto; reponer exactamente VentaLote, insertar AnulacionVenta con responsable/fecha/razón, cambiar estado y auditar. Serializar con cierre de caja; eventos tras commit. | Venta, razón, JWT | Anulación o errores de existencia, fecha, estado, caja cerrada o razón. No se rectifican cierres. | RF38 / CU38 |
| C48 | Ampliar en C18 HistorialVentasHandler: venta:buscar, venta:detalle | Mantener venta:historial-dia. Consultar detalle también para V32; usar precio histórico vinculado a DetalleVenta, descuento y VentaEfectivo. Cantidad/monto solo de vigentes y conteo de anuladas separado; conservar detalle original de anuladas. | Rango o número, JWT | Listado, resumen y detalle; error de fechas o lista vacía. | RF41 / CU41 |

# **11\. Consolidado de vistas extendidas desde Incremento 1**

Tabla de referencia con las vistas de Incremento 1 ampliadas, normalizando su código según la navegación y evitando pantallas duplicadas.

| Cód. | Vista (extiende Incremento 1\) | Qué agrega en Incremento 2 | RF/CU nuevo |
| :---- | :---- | :---- | :---- |
| V12 | RegistrarVentaView | Reutiliza descuento existente; UI04 organiza la edición y C16 revalida al guardar. | RF37 / CU37 |
| V15 | ListadoTrabajadoresView | Reutiliza listado/filtros; consulta para ambos roles y acciones administrativas solo Dueño. | RF23, RF24 / CU23, CU24 |
| V16 | FormularioTrabajadorView | Integra edición existente con RUT bloqueado, solo Dueño. | RF22 / CU22 |
| V17 | CalendarioTurnosView | Reutiliza semana/filtro; consulta ambos roles, acciones solo Dueño. | RF27, RF28 / CU27, CU28 |
| V18 | FormularioTurnoView | Integra creación/edición con validaciones existentes, solo Dueño. | RF26 / CU26 |
| V19 | RegistroAsistenciaView | Reutiliza y verifica cálculo/visualización de horas existentes. | RF31 / CU31 |

# **12\. Cobertura transversal heredada y adaptaciones**

| RF | CU | Nombre del CU | Controladores propios reutilizados | Nota |
| :---- | :---- | :---- | :---- | :---- |
| RF43 | CU43 | Registrando usuario responsable en cada venta | C03 \+ C05 \+ C16 (Incremento 1\) | V12 no permite elegir responsable; C03/C05 validan identidad y sesión, y C16 toma Usuario\_Cajero\_id del JWT. |
| RF44 | CU44 | Viendo total de ventas del día en tiempo real | C03 \+ C05 \+ C09 \+ C16; C47 nuevo | V05/UI02 consume C09; C16 y C47 emiten la actualización mediante C\_webContentsSend. |
| RF57 | CU57 | Controlando acceso por roles | C03 + C05, ampliados | Reutilizar comprobación bloqueante de sesión existente; adaptar permisos por operación, acceso a costos, rol de sesión y revocación al desactivar. |

# **13\. Matriz de trazabilidad separada: CU \- RF \- Vistas \- Controladores \- Arquitectura**

| CU | RF | Vista / componente | Responsabilidades I2 (nuevas o adaptadas) | Controladores propios reutilizados (I1) | Controladores técnicos de arquitectura | Entidades / persistencia |
| :---: | :---: | :---- | :---: | :---- | :---- | :---- |
| CU7 | RF07 | V20 | C26 | C03, C05 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Producto, Categoria, Lote, LotePerecible, HistorialPrecioProducto, Proveedor |
| CU11 | RF11 | V21 \+ UI01\* | C27 | C03, C04, C05, C13 | C\_jsonwebtoken, C\_cryptoNode, C\_webContentsSend, C\_drizzle, C\_libsql | Lote, AjusteInventario |
| CU12 | RF12 | V22 \+ UI01\* | C28 | C03, C05; C14/C15/C16 originan movimientos | C\_jsonwebtoken, C\_drizzle, C\_libsql | Lote, VentaLote, Venta, AnulacionVenta, MermaLote, Merma, AjusteInventario; eventos de recepción/ingreso |
| CU13 | RF13 | V23, V24 | C29 | C03, C04, C05 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | Proveedor, ProveedorCategoria |
| CU14 | RF14 | V23, V24 | C30 | C03, C04, C05 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Proveedor, ProveedorCategoria |
| CU15 | RF15 | V23 | C31 | C03, C05 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Proveedor, ProveedorCategoria, Categoria |
| CU16 | RF16 | V25 \+ UI05 | C32, C33 | C03, C05 | C\_jsonwebtoken, C\_webContents.printToPDF, C\_exceljs, C\_drizzle, C\_libsql | Producto, Categoria, Lote |
| CU17 | RF17 | V26 \+ UI01\* | C31, C34 | C03, C04, C05, C13 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | PedidoProveedor, DetallePedido |
| CU18 / CU18b | RF18 | V27 | C35 | C03, C04, C05 | C\_jsonwebtoken, C\_cryptoNode, C\_webContentsSend, C\_drizzle, C\_libsql | PedidoProveedor, DetallePedido, Lote, LotePerecible, HistorialAuditoriaPedido; RecepcionPedido/DetalleRecepcion propuestos |
| CU19 | RF19 | V28 | C36 | C03, C05 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Lote, Producto, Categoria |
| CU22 | RF22 | V16\* | C37 dentro de C21 | C03, C04, C05, C21 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | Trabajador, Usuario, UsuarioVersion |
| CU23 | RF23 | V15\*, V29 | C38 dentro de C21 | C03, C04, C05, C21 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | Trabajador, SesionUsuario; historial conservado |
| CU24 | RF24 | V15\* | C39 dentro de C21 | C03, C05, C21 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Trabajador, Usuario |
| CU26 | RF26 | V17\*, V18\* | C40 dentro de C22 | C03, C04, C05, C22 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | Turno, Asistencia |
| CU27 | RF27 | V17\* | C41 dentro de C22 | C03, C04, C05, C22 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | Turno, Asistencia |
| CU28 | RF28 | V17\* | C42 dentro de C22 | C03, C05, C21, C22 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Turno, Trabajador |
| CU31 | RF31 | V19\* | C43 (interna) dentro de C23 | C03, C05, C23 | Por C23: C\_jsonwebtoken, C\_drizzle, C\_libsql | Asistencia |
| CU34 | RF34 | V30 | C44 | C03, C04, C05 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | Trabajador, actividad del período, Remuneracion, TasaLegal, RemuneracionTasa |
| CU35 | RF35 | V31 | C45 | C03, C04, C05 | C\_jsonwebtoken, C\_cryptoNode, C\_drizzle, C\_libsql | TasaLegal, RemuneracionTasa |
| CU37 | RF37 | V12\* \+ UI04 | C46 dentro de C16 | C03, C05, C16 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Venta |
| CU38 | RF38 | V32 | C47; consulta C48 | C03, C04, C05, C18, C19, C20 | C\_jsonwebtoken, C\_cryptoNode, C\_webContentsSend, C\_drizzle, C\_libsql | Venta, CierreCaja, VentaLote, Lote, AnulacionVenta |
| CU41 | RF41 | V33 | C48 dentro de C18 | C03, C05, C18 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Venta, Usuario, DetalleVenta, Producto, HistorialPrecioProducto, VentaEfectivo |
| CU43 | RF43 | V12\* | \- | C03, C05, C16 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Venta.Usuario\_Cajero\_id |
| CU44 | RF44 | V05 \+ UI02\* | C47 aporta anulaciones | C03, C05, C09, C16 | C\_jsonwebtoken, C\_webContentsSend, C\_drizzle, C\_libsql | Venta |
| CU57 | RF57 | Todas las vistas protegidas | Adaptar permisos y revocación | C03, C05 | C\_jsonwebtoken, C\_drizzle, C\_libsql | Usuario, Trabajador, SesionUsuario |

# **14\. Reglas de consistencia para implementación**

| Criterio | Regla aplicable |
| :---- | :---- |
| Nomenclatura | C01-C48 identifican responsabilidades del equipo, existentes o por desarrollar según su estado en este documento. Los identificadores con prefijo C\_, como C\_jsonwebtoken, C\_bcryptjs, C\_drizzle y C\_libsql, son controladores técnicos de arquitectura; no son nuevos controladores funcionales Cxx. |
| Continuidad de numeración | V20-V33 y C26-C48 conservan referencias del diseño; no significan que toda su lógica sea nueva. V12 y V15-V19 se reutilizan/amplían; C37-C43/C46/C48 tienen base existente. |
| Reutilización transversal | Se reutilizan UI01, UI02 y C03, C04, C05, C09, C13, C14, C16, C18 y C21-C23. C03 valida autorización y C05 valida sesión antes de los canales protegidos. |
| Patrón autenticado | Vista → preload/IPC → authorizeRequest (token, permiso y sesión) → dominio → persistencia. La base 9f790f6 ya comprueba sesión antes del handler; reutilizar y ampliar. |
| Inicio de sesión CU56 | V01 invoca C01 AuthHandler. C01 consulta Usuario/Trabajador/Contrasena mediante C\_drizzle/C\_libsql, usa C\_bcryptjs y C\_jsonwebtoken, registra el intento y crea SesionUsuario. Estos controladores técnicos no agregan controladores funcionales Cxx. |
| Separación MVC e IPC | Las vistas React/TypeScript se comunican mediante el preload/IPC existente y los Cxx se ejecutan en Electron Main. Preload/contextBridge, ipcMain.handle y Electron Main son infraestructura, no controladores. Ninguna vista importa Drizzle, @libsql/client, node:crypto ni APIs Node. |
| Acceso al modelo | Reutilizar Drizzle/libSQL y transacciones de la base. La sincronización no se da por implementada ni se invoca desde las vistas. |
| Stock derivado | El modelo no contiene Producto\_Stock\_Actual. C26, C27, C32, C35, C36 y C47 derivan el stock con Lote\_Cantidad\_Actual; no deben escribir una columna inexistente ni crear una tabla genérica MovimientoInventario. |
| Roles y costo | Ambos roles acceden a costos, proveedores, recepción, ajustes y valorización. Administración de trabajadores, mutación de turnos, registro de remuneraciones y configuración previsional son de Dueño. Calendario y consultas no habilitan mutaciones. Ver matriz acordada en sección 14. |
| Transacciones atómicas | Revalidar y escribir en la misma transacción en C27/C35/C47 y mutaciones de personal/turnos. Agrupar entidades relacionadas y auditoría en altas/ediciones de proveedores, pedidos, remuneraciones y tasas. Eventos después del commit. |
| Historial de inventario | Un ingreso se contabiliza una vez. Distinguir ajustes heredados de ingreso y manuales; sumar restituciones en fecha de anulación. Vincular responsable de ingreso estructuralmente; no depender de texto de auditoría por EAN. Ver adaptaciones. |
| Vigencia previsional | C45 cierra la tasa anterior y crea una nueva. C44 conserva las tasas aplicadas mediante RemuneracionTasa para que el histórico no cambie. |
| Auditoría | Los controladores que modifican datos delegan en C04; C04 usa C\_cryptoNode para UUID y C\_drizzle/C\_libsql para persistir LogAuditoria inmutable. |
| Eventos y exportación | Los cambios de stock/ventas usan C\_webContentsSend. C33 orquesta C\_webContents.printToPDF y C\_exceljs con UI05, conforme al diagrama de componentes. |
| Fuera de alcance | Se mantiene fuera RF45/CU45/V35/C49 porque RF45 no aparece en Grupo 6 - Incremento 2.docx (carpeta diagramas de secuencias) y el diagrama de componentes ubica RF45-RF53 en Reportes. |


## 14.1 Estados y trazabilidad de pedidos

| Estado funcional | Valor propuesto | Condición y operaciones |
|---|---|---|
| Pendiente | `pendiente` | Sin recepciones: recibir o cancelar. |
| Recibido parcialmente | `parcial` | Recibido > 0 y faltante abierto: recibir otra entrega o cerrar saldo. |
| Recibido | `recibido` | Todo completado: consultar. |
| Cancelado | `cancelado` | Terminado antes de recibir: consultar, sin lotes generados por el pedido. |
| Cerrado con recepción parcial | `parcial_cerrado` | Faltante que ya no llegará: consultar; conservar lo recibido. |

Pendiente por línea = solicitado − recibido acumulado. Cada confirmación ingresa cantidades **de esa entrega**, enteras entre cero y pendiente, con al menos una línea positiva. Total completa el saldo restante; Parcial registra menos que ese saldo. El servidor determina el estado a partir de los acumulados, no confía solo en la opción de la vista. Cada entrega genera sus lotes con costo y vencimiento propios; una línea puede originar varios lotes en recepciones sucesivas.

Cerrar saldo exige motivo no vacío y confirmación; conserva solicitado, recibido y faltante no entregado, fecha/hora y responsable. No finge recepción, no elimina lotes ni altera stock. Bloquea nuevas recepciones. Cancelar el diálogo mantiene el estado anterior. La cancelación antes de recibir y el cierre de saldo se documentan como variantes de CU18b, sin agregar RF.

Un identificador único de operación evita duplicar una recepción por reintento. Si faltan datos de lote, se rechaza toda esa nueva entrega; las recepciones anteriores se conservan. Datos incompletos y cantidad recibida parcial son situaciones diferentes.

## 14.2 Permisos acordados

| Operación | Dueño | Trabajador |
|---|---|---|
| Costos, detalle de producto, proveedores y valorización | Sí | Sí |
| Ajustes, movimientos, reabastecimiento y exportación | Sí | Sí |
| Registrar/editar proveedores; registrar, recibir, cancelar y cerrar saldo de pedidos | Sí | Sí |
| Consultar listado de trabajadores | Sí | Sí, sin acciones administrativas |
| Registrar trabajadores, asignar rol inicial, editar datos y cambiar rol/estado | Sí | No |
| Consultar calendario semanal y filtrar por trabajador | Sí | Sí |
| Crear, editar o eliminar turnos | Sí | No |
| Registrar remuneraciones y configurar porcentajes | Sí | No |
| Ventas, descuentos, consulta y anulación bajo las condiciones de RF38 | Sí | Sí |
| Reportes de rentabilidad y consulta del log de auditoría | Sí | No |

La consulta de trabajadores sigue el permiso general RF57; las mutaciones se restringen por operación. La valorización expresa existencias por costo, no rentabilidad. Se mantienen los RF específicos de las demás funciones. Las vistas heredadas que ocultaban costos también requieren adecuación; no se consideran funcionalidades nuevas por ese ajuste.

## 14.3 Cálculo, históricos y fechas

Un lote con existencias tiene cantidad > 0; agotado, cantidad = 0. El vencimiento es una condición separada. Seleccionar un lote para corregir conteo no cambia su identidad, costo ni vencimiento, ni lo habilita automáticamente para venta. Se valora la existencia registrada: el vencimiento no descuenta unidades físicamente; una merma o ajuste registrado modifica su cantidad.

Remuneración: por cada concepto, redondear bruto × porcentaje / 100 al peso más cercano; medio peso hacia arriba. Líquido = bruto − suma de descuentos ya redondeados. Ejemplo: bruto $1.005 con 11,5%, 7% y 0,6% produce descuentos $116, $70 y $6, y líquido $813. Vista y Main deben usar el mismo criterio. La elegibilidad de RF34 se conserva sin exigir un nuevo tipo de asistencia: se puede pagar a un trabajador desactivado que trabajó durante el mes.

Las tasas se resuelven **al registrar**, no según el mes remunerado. RemuneracionTasa conserva las referencias aplicadas y C45 no modifica el porcentaje de filas históricas. Si cambió la configuración desde la previsualización, se muestra el cálculo actualizado antes de confirmar el guardado. No se agregan vigencias futuras/retroactivas elegidas manualmente.

Venta del día se determina según `America/Santiago`, reutilizando el criterio de I1. C47 comprueba la caja **asociada a esa venta**, no otra caja abierta. Si el cierre gana la concurrencia, se rechaza la anulación sin cambios. Las anuladas conservan su importe original en detalle pero no suman a montos/conteos vigentes; cantidad de anuladas separada, igual que RF44.

## 14.4 Adaptaciones del modelo y del código todavía no implementadas

| Área | Adaptación requerida |
|---|---|
| Estados | El enum actual no admite `pendiente` ni `parcial_cerrado`: requieren migración. Cualquier dato heredado `borrador/emitido/enviado` debe revisarse antes de convertirlo, sin equivalencias supuestas. |
| Recepciones | Incorporar RecepcionPedido (pedido, fecha/hora, usuario, operación única) y DetalleRecepcion (entrega, línea de pedido, cantidad y lote(s)). Son entidades propuestas, no tablas existentes. La cabecera actual tiene un único receptor/fecha y no basta como historial. |
| Acumulados/cierre | CantidadRecibida es acumulada y coincide con las entregas. El cierre guarda motivo/fecha/responsable en el historial del pedido y conserva el faltante no entregado. |
| Ingresos/ajustes | Distinguir origen ingreso/manual en AjusteInventario y asociar responsable al ingreso directo del lote. C14 crea hoy lote y ajuste por la misma entrada; C28 cuenta el lote una vez y excluye el ajuste de ingreso de la suma. No eliminar registros para ocultar duplicación. |
| Datos heredados | Recuperar responsable desde el ajuste de ingreso inequívocamente asociado. La auditoría por EAN no basta. Clasificar solo correspondencias comprobables y señalar casos sin responsable recuperable, sin inventar identidades. |
| Orden/saldos | Orden estable además de fecha/hora; conservar orden de registro de nuevos eventos. No afirmar orden causal en empates heredados que la base no distingue. Último saldo reconciliado con stock actual. |
| Sesiones/roles | La base `9f790f6` ya verifica sesión antes del dominio. Ampliar revocación por desactivación y alinear guard/helpers con rol de sesión hasta siguiente login. |
| Canales | Conservar `trabajador:actualizar`, `trabajador:listar`, `trabajador:cambiar-estado`, `turno:listar`, `turno:editar` y `turno:eliminar` en sus propietarios actuales. Corregir llamadas de interfaz que difieran; no duplicar registros. |
| Remuneración | Persistir Remuneracion/RemuneracionTasa existentes, sin columnas derivadas ficticias. La adaptación debe reproducir el cálculo y conservar históricos. |

No se crean tablas genéricas de movimiento: se reconstruye el historial desde sus fuentes, con las asociaciones explícitas requeridas. Las adaptaciones del modelo se distinguen de los datos ya presentes.

## 14.5 Correspondencia con archivos de I1

Rutas relativas a `implementacion-sistema-gestion/src/`. Se conserva el propietario actual de los canales aunque la responsabilidad se nombre con otro Cxx en los diagramas.

| Responsabilidad | Archivos de referencia |
|---|---|
| C03/C05: autorización y sesión | `main/controllers/auth-guard.ts`, `session.ts`, `index.ts`, `auth-context.ts` |
| C13/C14/C15: consulta, lotes y merma | `main/controllers/product-query.ts`, `lot.ts`, `waste.ts` |
| C21/C37-C39: trabajadores | `main/controllers/worker.ts`; `renderer/src/views/WorkerListView.tsx`, `WorkerFormView.tsx` |
| C22/C40-C42: turnos | `main/controllers/shift.ts`; `renderer/src/views/ShiftCalendarView.tsx`, `ShiftCreateView.tsx` |
| C23/C43: horas | `main/controllers/attendance-service.ts`; `renderer/src/views/AttendanceView.tsx` |
| C16/C46: descuentos | `main/controllers/sale.ts`, `sale-service.ts`; `renderer/src/views/SaleRegisterView.tsx` |
| C18/C48: consulta de ventas | `main/controllers/sales-history.ts`; `renderer/src/views/DailySalesView.tsx` |
| Contratos y navegación | `shared/controllers.ts`, `shared/navigation.ts`, `preload/index.ts` |

## 14.6 Árbol de navegación del incremento

Árbol integrado de entradas de I2 y vistas relacionadas de I1. D = operación exclusiva del Dueño. Una vista compartida por dos entradas tiene un solo registro de navegación. Las rutas nuevas se asignarán en el catálogo central al implementar; no se duplican las existentes.

```text
Login V01 → Dashboard V05 (UI02)
├─ Inventario
│  ├─ Productos V06 → Detalle V20 → Historial V22
│  ├─ Ajuste V21 (UI01)
│  ├─ Historial V22 (misma vista)
│  ├─ Reabastecimiento V25 → Pedido V26
│  └─ Valor de inventario V28
├─ Proveedores
│  ├─ Listado V23 → Formulario V24
│  ├─ Reabastecimiento V25 (misma vista)
│  ├─ Pedido V26 (UI01)
│  └─ Recepciones/historial V27
│     └─ Detalle interno: recibir / cancelar / cerrar saldo / entregas
├─ Personal
│  ├─ Trabajadores V15 → Formulario V16 [D] / Estado V29 [D]
│  ├─ Calendario V17 → Formulario V18 [D] / eliminar [D]
│  ├─ Asistencia V19
│  ├─ Remuneración V30 [D]
│  └─ Configuración previsional V31 [D]
└─ Ventas
   ├─ Registro V12 (UI04)
   ├─ Ventas del día V13 → Anular V32
   ├─ Anular V32 (misma vista)
   └─ Consulta V33 → detalle interno / Anular V32
```

Las demás entradas de I1, como lotes, merma, caja y administración, permanecen en su árbol; deben alinear permisos con RF57. UI05 es un render de impresión interno, fuera del árbol navegable. Las confirmaciones de cierre, cancelación, ajuste, estado y eliminación son componentes internos.

# 15. Cambios de esta revisión

Se incorporaron los acuerdos de la sesión sobre costos, permisos por operación, lotes agotados/vencidos, entregas sucesivas, cierre de saldo, revocación de sesiones, anulación con caja abierta, totales de vigentes y redondeo. Se corrigió la clasificación nuevo/reutilizado y se completaron consultas, navegación y dependencias del modelo. El catálogo real de I1 contiene también `product-delete`: no se debe interpretar su posición en el arreglo como el identificador C26; los códigos de diseño no son índices del registro.

Los RF y CU del Word de referencia se actualizan junto con sus repeticiones en tablas. Los diagramas existentes requieren actualización **por el usuario**, según lo acordado: CU7/CU19 y proveedores (costos/roles), CU11/CU12 (ajustes e historial), CU17/CU18/CU18b (entregas/cierre), CU23/CU57 (revocación/permisos), CU28 (consulta), CU34/CU35 (cálculo), CU38 (caja abierta) y CU41 (totales). No se modifican esos archivos ni se afirma que ya estén alineados.

Las versiones anteriores se conservan en el respaldo de revisión. Las reglas vigentes son las de este documento. El código de I1 queda sin cambios; esta revisión es especificación para la planificación posterior, no evidencia de funcionalidades I2 ya implementadas.
