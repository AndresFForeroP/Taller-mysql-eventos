**TALLER EVENTOS**

Haciendo uso de las siguientes tablas para la base de datos de pizza realice los siguientes ejercicios de Events centrados en el uso de ON COMPLETION PRESERVE y ON COMPLETION NOT PRESERVE :

```mysql
CREATE TABLE IF NOT EXISTS resumen_ventas (
fecha       DATE      PRIMARY KEY,
total_pedidos INT,
total_ingresos DECIMAL(12,2),
creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id  INT NOT NULL,
  stock_actual    INT NOT NULL,
  fecha_alerta    DATETIME NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
```

1. Resumen Diario Único : crear un evento que genere un resumen de ventas una sola vez al finalizar el día de ayer y luego se elimine automáticamente llamado ev_resumen_diario_unico.

**TABLAS NUEVAS**

```mysql
CREATE TABLE IF NOT EXISTS pedidos(
	id INT AUTO_INCREMENT PRIMARY KEY,
    monto_pedido DECIMAL(10,2)
);
CREATE TABLE IF NOT EXISTS ingrediente(
	id INT AUTO_INCREMENT PRIMARY KEY,
	nombre VARCHAR(40),
    stock INT
);
```

**PROCEDIMIENTO ALMACENADO**

```mysql
DELIMITER //

CREATE PROCEDURE ResumenDiario()
BEGIN
    INSERT INTO resumen_ventas(fecha, total_pedidos, total_ingresos)
	SELECT 
		DATE_SUB(CURDATE(), INTERVAL 1 DAY) AS fecha,
		COUNT(*) AS total_pedidos,
		SUM(monto_pedido) AS total_ingresos
	FROM pedidos
	WHERE DATE(fecha_pedido) = DATE_SUB(CURDATE(), INTERVAL 1 DAY);
END //

DELIMITER ;
```

**SOLUCIÓN EVENTO**

```mysql
DROP EVENT IF EXISTS ev_resumen_diario_unico;
DELIMITER //
CREATE EVENT IF NOT EXISTS ev_resumen_diario_unico
ON SCHEDULE AT ('2025-07-10 00:00:00')
ON COMPLETION NOT PRESERVE
DO
BEGIN
	CALL ResumenDiario();
END //

DELIMITER ;
```

**RESULTADO**

```mysql
mysql> SHOW CREATE EVENT ev_resumen_diario_unico;
+-------------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| Event                   | sql_mode                                                                                                              | time_zone | Create Event                                                                                                                                                                | character_set_client | collation_connection | Database Collation |
+-------------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| ev_resumen_diario_unico | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION | SYSTEM    | CREATE DEFINER=`root`@`localhost` EVENT `ev_resumen_diario_unico` ON SCHEDULE AT '2025-07-10 00:00:00' ON COMPLETION NOT PRESERVE ENABLE DO BEGIN
CALL ResumenDiario();
END | utf8mb4              | utf8mb4_0900_ai_ci   | utf8mb4_0900_ai_ci |
+-------------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
1 row in set (0,00 sec)

```

2.Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, **manteniendo** el evento para que siga ejecutándose cada semana llamado `ev_resumen_semanal`.

**PROCESO ALMACENADO**

```mysql
DELIMITER //

CREATE PROCEDURE ResumenSemanal()
BEGIN
    INSERT INTO resumen_ventas(fecha, total_pedidos, total_ingresos)
	SELECT 
		DATE_SUB(CURDATE(), INTERVAL 7 DAY) AS fecha,
		COUNT(*) AS total_pedidos,
		SUM(monto_pedido) AS total_ingresos
	FROM pedidos
	WHERE fecha_pedido BETWEEN DATE_SUB(CURDATE(), INTERVAL 1 WEEK)
                          AND DATE_SUB(CURDATE(), INTERVAL 1 DAY);
END //

DELIMITER ;
```

**SOLUCIÓN EVENTO**

```mysql
DROP EVENT IF EXISTS ev_resumen_semanal;
DELIMITER //

CREATE EVENT IF NOT EXISTS ev_resumen_semanal
ON SCHEDULE
EVERY 1 WEEK
STARTS ('2025-07-14 01:00:00' )
ON COMPLETION PRESERVE
DO
BEGIN 
	CALL ResumenSemanal();
END //

DELIMITER ;
```

**RESULTADO**

```mysql
mysql> SHOW CREATE EVENT ev_resumen_semanal;
+--------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| Event              | sql_mode                                                                                                              | time_zone | Create Event                                                                                                                                                                          | character_set_client | collation_connection | Database Collation |
+--------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| ev_resumen_semanal | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION | SYSTEM    | CREATE DEFINER=`root`@`localhost` EVENT `ev_resumen_semanal` ON SCHEDULE EVERY 1 WEEK STARTS '2025-07-14 01:00:00' ON COMPLETION PRESERVE ENABLE DO BEGIN 
CALL ResumenSemanal();
END | utf8mb4              | utf8mb4_0900_ai_ci   | utf8mb4_0900_ai_ci |
+--------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
1 row in set (0,00 sec)

```

3.Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (`alerta_stock`) de ingredientes con stock < 5, y luego autodestruir el evento.

**PROCESO ALMACENADO**

```mysql
DELIMITER //

CREATE PROCEDURE ComprobarStock(IN cantidad INT)
BEGIN
	INSERT INTO alerta_stock(ingrediente_id, stock_actual, fecha_alerta)
    SELECT id, stock, NOW()
    FROM ingrediente
    WHERE stock < cantidad;
END //

DELIMITER ;
```

**SOLUCION EVENTO**

```mysql
DROP EVENT IF EXISTS ev_stock_bajo;
DELIMITER //

CREATE EVENT IF NOT EXISTS ev_stock_bajo
ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN 
	CALL ComprobarStock(5);
END //

DELIMITER ;
```

**RESULTADO**

```mysql
mysql> SHOW CREATE EVENT ev_stock_bajo;
+---------------+-----------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| Event         | sql_mode                                                                                                              | time_zone | Create Event                                                                                                                                                        | character_set_client | collation_connection | Database Collation |
+---------------+-----------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| ev_stock_bajo | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION | SYSTEM    | CREATE DEFINER=`root`@`localhost` EVENT `ev_stock_bajo` ON SCHEDULE AT '2025-07-09 22:30:17' ON COMPLETION NOT PRESERVE ENABLE DO BEGIN 
CALL ComprobarStock(5);
END | utf8mb4              | utf8mb4_0900_ai_ci   | utf8mb4_0900_ai_ci |
+---------------+-----------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
1 row in set (0,00 sec)

```

4.Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en `alerta_stock`, **dejando** el evento activo para siempre llamado `ev_monitor_stock_bajo`.

**SOLUCIÓN EVENTO**

```mysql
DROP EVENT IF EXISTS ev_monitor_stock_bajo;
DELIMITER //

CREATE EVENT IF NOT EXISTS ev_monitor_stock_bajo
ON SCHEDULE
	EVERY 30 MINUTE
	STARTS NOW() + INTERVAL 30 MINUTE
ON COMPLETION PRESERVE
DO
BEGIN 
	CALL ComprobarStock(10);
END //

DELIMITER ;
```

**RESULTADO**

```mysql
SHOW CREATE EVENT ev_monitor_stock_bajo;
+-----------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| Event                 | sql_mode                                                                                                              | time_zone | Create Event                                                                                                                                                                                  | character_set_client | collation_connection | Database Collation |
+-----------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| ev_monitor_stock_bajo | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION | SYSTEM    | CREATE DEFINER=`root`@`localhost` EVENT `ev_monitor_stock_bajo` ON SCHEDULE EVERY 30 MINUTE STARTS '2025-07-10 01:31:10' ON COMPLETION PRESERVE ENABLE DO BEGIN 
CALL ComprobarStock(10);
END | utf8mb4              | utf8mb4_0900_ai_ci   | utf8mb4_0900_ai_ci |
+-----------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
1 row in set (0,00 sec)
```

5.Limpieza de Resúmenes Antiguos: una sola vez, eliminar de `resumen_ventas` los registros con fecha anterior a hace 365 días y luego borrar el evento llamado `ev_purgar_resumen_antiguo`.

**PROCESO ALMACENADO**

```mysql
DELIMITER //

CREATE PROCEDURE PurgarResumen()
BEGIN
	DELETE FROM resumen_ventas
	WHERE creado_en < DATE_SUB(CURRENT_DATE(),INTERVAL 365 DAY);
END //

DELIMITER ;
```

**SOLUCION EVENTO**

```mysql
DROP EVENT IF EXISTS ev_purgar_resumen_antiguo;
DELIMITER //

CREATE EVENT IF NOT EXISTS ev_purgar_resumen_antiguo
ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN 
	CALL PurgarResumen();
END //

DELIMITER ;
```

**RESULTADO**

```mysql
mysql> SHOW CREATE EVENT ev_purgar_resumen_antiguo;
+---------------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| Event                     | sql_mode                                                                                                              | time_zone | Create Event                                                                                                                                                                   | character_set_client | collation_connection | Database Collation |
+---------------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
| ev_purgar_resumen_antiguo | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION | SYSTEM    | CREATE DEFINER=`root`@`localhost` EVENT `ev_purgar_resumen_antiguo` ON SCHEDULE AT '2025-07-10 01:15:11' ON COMPLETION NOT PRESERVE ENABLE DO BEGIN 
CALL PurgarResumen();
END | utf8mb4              | utf8mb4_0900_ai_ci   | utf8mb4_0900_ai_ci |
+---------------------------+-----------------------------------------------------------------------------------------------------------------------+-----------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+--------------------+
1 row in set (0,00 sec)

```
