# **Interconexión entre Oracle 19c y PostgreSQL**

# Oracle 19c a PostgreSQL

Dado que la documentación y los recursos disponibles sobre la interconexión entre Oracle 21c y PostgreSQL son limitados,he optado por realizar la interconexión utilizando Oracle 19c, una versión muy documentada y compatible con muchas herramientas. Además, muchos conceptos y configuraciones entre Oracle 19c y 21c son similares, por lo que no habrá problema.

Bien, para ello debemos tener un servidor Oracle 19c instalado y preparado para interconectar. La instalación es muy parecida  (por no decir igual) a la de Oracle 21c. Como vemos puedo acceder a la base de datos:

```
pablo@oracle19c:~$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Sat Nov 16 18:09:11 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.

SQL>
```

Al acceder a la base de datos después de que la máquina haya sido apagada, nos saltará el mensaje "Connected to an idle instance".
Este mensaje indica que la instancia de Oracle estaba en estado inactivo cuando me he conectado, por lo tanto, para iniciar el proceso para arrancar la instancia de Oracle y abrir la base de datos asociada. Ejecutamos el siguiente comando:

```
SQL> STARTUP;
ORACLE instance started.

Total System Global Area 1644164456 bytes
Fixed Size		    9135464 bytes
Variable Size		 1006632960 bytes
Database Buffers	  620756992 bytes
Redo Buffers		    7639040 bytes
Base de datos montada.
Base de datos abierta.
```

Bien, una vez tenemos todo listo debemos de instalar los siguientes paquetes necesarios para la interconexión:

```
pablo@oracle19c:~$ sudo apt install odbc-postgresql unixodbc -y
```

El primer paso es buscar el archivo de configuración `odbcinst.ini`, que es utilizado para almacenar información relacionada con los controladores ODBC en el sistema operativo. Para encontrar la ubicación de este archivo, se utilizó el siguiente comando:

```
pablo@oracle19c:~$ sudo find / -name odbcinst.ini
/etc/odbcinst.ini
```

El siguiente paso fue asegurarse de que el controlador ODBC de PostgreSQL estuviera instalado en el sistema. Para ello, se utilizó el siguiente comando de búsqueda:

```
pablo@oracle19c:~$ sudo find /usr -name psqlodbcw.so
/usr/lib/x86_64-linux-gnu/odbc/psqlodbcw.so
```

Por último, verificamos la instalación de los paquetes necesarios para el funcionamiento de ODBC:

```
pablo@oracle19c:~$ dpkg -l | grep odbc
ii  libodbc2:amd64               2.3.11-2+deb12u1               amd64        ODBC Driver Manager library for Unix
ii  libodbcinst2:amd64           2.3.11-2+deb12u1               amd64        Support library for accessing ODBC configuration files
ii  odbc-postgresql:amd64        1:13.02.0000-2+b1              amd64        ODBC driver for PostgreSQL
ii  odbcinst                     2.3.11-2+deb12u1               amd64        Helper program for accessing ODBC configuration files
ii  unixodbc                     2.3.11-2+deb12u1               amd64        Basic ODBC tools
ii  unixodbc-common              2.3.11-2+deb12u1               all          Common ODBC configuration files
```

Una vez confirmado que los archivos y paquetes necesarios están instalados, he procedido a editar el archivo `odbcinst.ini` para agregar la configuración del controlador ODBC de PostgreSQL. De forma que mi fichero qudaría así:

```
pablo@oracle19c:~$ cat /etc/odbcinst.ini
[PostgreSQL]
Description     = ODBC for PostgreSQL
Driver          = /usr/lib/x86_64-linux-gnu/odbc/psqlodbcw.so
Setup           = /usr/lib/x86_64-linux-gnu/odbc/libodbcpsqlS.so
Driver64        = /usr/lib/x86_64-linux-gnu/odbc/psqlodbcw.so
Setup64         = /usr/lib/x86_64-linux-gnu/odbc/libodbcpsqlS.so
```

El siguiente paso será crear un **DSN (Nombre de Fuente de Datos)** en el archivo `/etc/odbc.ini`, ya que este archivo se utilizará más adelante para definir cómo establecer la conexión con el gestor de base de datos indicado. El contenido será el siguiente:

```
pablo@oracle19c:~$ cat /etc/odbc.ini 
[PSQLORCL]
Debug           = 0
CommLog         = 0
ReadOnly        = 0
Driver          = PostgreSQL
Servername      = 192.168.122.163
Username        = pablo
Password        = pablo
Port            = 5432
Database        = prueba
Trace           = 0
TraceFile       = /tmp/sql.log
```

Donde:

- **Driver**: Especificamos el nombre del controlador previamente configurado en el archivo `/etc/odbcinst.ini`. En este caso, PostgreSQL.

- **Servername**: Indicamos la dirección IP del servidor PostgreSQL al que queremos establecer la conexión. En este caso, `192.168.122.163`.

- **Username**: Definimos el nombre del usuario con el que accederemos a la base de datos remota. En este caso, `pablo`.

- **Password**: Proporcionamos la contraseña asociada al usuario para acceder a la base de datos remota. En este caso, `pablo`.

- **Port**: Establecemos el puerto en el que el servidor PostgreSQL está escuchando solicitudes. En este caso, `5432`.

- **Database**: Especificamos el nombre de la base de datos remota a la que deseamos conectarnos. En este caso, `prueba`.


Hago aquí un pequeño inciso para mostrar la creación del usuario y de las tablas en PostgreSQL.

El usuario:

```
postgres@servidor-postgre1:~$ psql
psql (15.8 (Debian 15.8-0+deb12u1))
Digite «help» para obtener ayuda.

postgres=# CREATE USER pablo WITH PASSWORD 'pablo';
CREATE ROLE
postgres=# CREATE DATABASE prueba OWNER pablo;
CREATE DATABASE
postgres=# GRANT ALL PRIVILEGES ON DATABASE prueba TO pablo;
GRANT
postgres=# exit
```

Y las tablas con sus respectivos datos:

```
postgres@servidor-postgre1:~$ psql -h localhost -U pablo -d prueba
Contraseña para usuario pablo: 
psql (15.8 (Debian 15.8-0+deb12u1))
Conexión SSL (protocolo: TLSv1.3, cifrado: TLS_AES_256_GCM_SHA384, compresión: desactivado)
Digite «help» para obtener ayuda.

prueba=> CREATE TABLE empleados (
    empleado_id SERIAL PRIMARY KEY,
    nombre_completo VARCHAR(150) NOT NULL,
    puesto VARCHAR(100),
    salario NUMERIC(10, 2),
    fecha_contratacion DATE NOT NULL,
    activo BOOLEAN DEFAULT TRUE
);
CREATE TABLE
prueba=> INSERT INTO empleados (nombre_completo, puesto, salario, fecha_contratacion) VALUES
('Laura Martínez', 'Desarrolladora', 50000.00, '2020-05-15'),
('Pedro Gutiérrez', 'Analista de Datos', 45000.00, '2021-03-01'),
('Lucía Fernández', 'Gerente de Proyectos', 60000.00, '2019-07-10'),
('Miguel Torres', 'Especialista en Marketing', 40000.00, '2022-01-20'),
('Carla López', 'Diseñadora Gráfica', 35000.00, '2021-08-05');
INSERT 0 5
prueba=> CREATE TABLE proyectos (
    proyecto_id SERIAL PRIMARY KEY,
    nombre_proyecto VARCHAR(100) NOT NULL,
    descripcion TEXT,
    fecha_inicio DATE NOT NULL,
    fecha_fin DATE,
    presupuesto NUMERIC(12, 2)
);
CREATE TABLE
prueba=> INSERT INTO proyectos (nombre_proyecto, descripcion, fecha_inicio, fecha_fin, presupuesto) VALUES
('Sistema CRM', 'Desarrollo de un sistema de gestión de relaciones con clientes', '2023-01-01', '2023-12-31', 150000.00),
('Campaña Publicitaria', 'Estrategia de marketing para redes sociales', '2023-06-01', '2023-09-30', 50000.00),
('Diseño de Marca', 'Creación de logotipo y manual de identidad visual', '2023-05-01', NULL, 20000.00),
('Análisis de Mercado', 'Estudio sobre tendencias de consumo', '2024-01-01', '2024-03-31', 30000.00);
INSERT 0 4
prueba=> CREATE TABLE asistencias (
    asistencia_id SERIAL PRIMARY KEY,
    empleado_id INT NOT NULL REFERENCES empleados(empleado_id),
    fecha DATE NOT NULL,
    hora_entrada TIME NOT NULL,
    hora_salida TIME
);
CREATE TABLE
prueba=> INSERT INTO asistencias (empleado_id, fecha, hora_entrada, hora_salida) VALUES
(1, '2024-11-10', '08:00:00', '17:00:00'),
(2, '2024-11-10', '08:30:00', '16:30:00'),
(3, '2024-11-10', '09:00:00', '18:00:00'),
(4, '2024-11-10', '08:15:00', '17:15:00'),
(5, '2024-11-10', '08:45:00', '16:45:00');
INSERT 0 5
```

Ahora, con el comando isql PSQLORCL, voy a conectarme a la base de datos PostgreSQL mediante ODBC.

```
pablo@oracle19c:~$ isql PSQLORCL
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| echo [string]                         |
| quit                                  |
|                                       |
+---------------------------------------+
```

 Una vez establecida la conexión con éxito, podemos ejecutar una consulta para ver los datos de la tabla `empleados` por ejemplo. Con esto, demostramos que la configuración del DSN PSQLORCL en los archivos `odbc.ini` y `odbcinst.ini` fue correcta, permitiendo la comunicación entre Oracle y PostgreSQL a través del driver ODBC.

```
SQL> SELECT * FROM empleados;
+------------+-------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------+-------------+-------------------+-------+
| empleado_id| nombre_completo                                                                                                                                       | puesto                                                                                              | salario     | fecha_contratacion| activo|
+------------+-------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------+-------------+-------------------+-------+
| 1          | Laura Martínez                                                                                                                                       | Desarrolladora                                                                                      | 50000,00    | 2020-05-15        | 1     |
| 2          | Pedro Gutiérrez                                                                                                                                      | Analista de Datos                                                                                   | 45000,00    | 2021-03-01        | 1     |
| 3          | Lucía Fernández                                                                                                                                     | Gerente de Proyectos                                                                                | 60000,00    | 2019-07-10        | 1     |
| 4          | Miguel Torres                                                                                                                                         | Especialista en Marketing                                                                           | 40000,00    | 2022-01-20        | 1     |
| 5          | Carla López                                                                                                                                          | Diseñadora Gráfica                                                                                | 35000,00    | 2021-08-05        | 1     |
+------------+-------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------+-------------+-------------------+-------+
SQLRowCount returns 5
5 rows fetched
```

Como se mencionó anteriormente, aunque el driver ya está configurado, Oracle aún no está preparado para utilizarlo. El siguiente paso consiste en crear un archivo de configuración para habilitar los *Heterogeneous Services*. Este archivo contiene parámetros necesarios para que Oracle pueda interactuar correctamente con el driver configurado.

El archivo debe ubicarse en la ruta `$ORACLE_HOME/hs/admin/` y seguir el formato de nombre `init[DSN].ora`. En este caso, dado que el DSN es `PSQLORCL`, el archivo deberá llamarse `initPSQLORCL.ora`. De forma que el nuevo fichero quedaría así:

```
pablo@oracle19c:~$ cat /opt/oracle/product/19c/dbhome_1/hs/admin/initPSQLORCL.ora
HS_FDS_CONNECT_INFO = PSQLORCL
HS_FDS_TRACE_LEVEL = DEBUG
HS_FDS_SHAREABLE_NAME = /usr/lib64/psqlodbcw.so
HS_LANGUAGE = AMERICAN_AMERICA.WE8ISO8859P1
set ODBCINI=/etc/odbc.ini
```

La siguiente etapa de configuración implica modificar el archivo `listener.ora` para incluir la definición del listener, necesario para que Oracle pueda comunicarse con el servicio heterogéneo configurado previamente. Este archivo se encuentra en la ruta `$ORACLE_HOME/network/admin/`. Por lo tanto:

**Definición del Listener:**
   ```plaintext
pablo@oracle19c:~$ cat /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
# listener.ora Network Configuration File: /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
# Generated by Oracle configuration tools.

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = oracle19c)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
   ```
   - `LISTENER`: Es el nombre del listener.
   - `DESCRIPTION_LIST` y `DESCRIPTION`: Estas secciones configuran las direcciones que el listener utilizará.
   - `ADDRESS` con `PROTOCOL = TCP`: Define la dirección TCP/IP del listener. Aquí se especifica el `HOST` con la dirección IP o con el nombre de host, yo pondré "oracle19c" que es mi nombre de host. Y por último, el `PORT` como `1521`.
   - `ADDRESS` con `PROTOCOL = IPC`: Configura una dirección de comunicación interna mediante el protocolo IPC (Interprocess Communication), con una clave identificadora `EXTPROC1521`.

**Definición del SID (Service Identifier):**
   ```plaintext
SID_LIST_LISTENER=
  (SID_LIST=
      (SID_DESC=
         (SID_NAME=PSQLORCL)
         (ORACLE_HOME=/opt/oracle/product/19c/dbhome_1)
         (PROGRAM=dg4odbc)
      )
  )
   ```
   - `SID_LIST_LISTENER`: Asocia un identificador de servicio (SID) con el listener.
   - `SID_DESC`: Describe el servicio a asociar.
   - `SID_NAME = PSQLORCL`: Define el nombre del SID que usará Oracle para referirse a este servicio heterogéneo.
   - `ORACLE_HOME`: Especifica el directorio base de Oracle, en este caso `/opt/oracle/product/19c/dbhome_1`.
   - `PROGRAM = dg4odbc`: Indica que se utilizará el programa `dg4odbc` (Database Gateway for ODBC), encargado de la comunicación con bases de datos externas mediante ODBC.

Esta configuración asegura que el listener de Oracle pueda reconocer y gestionar solicitudes hacia el servicio heterogéneo asociado al DSN configurado previamente (`PSQLORCL`).

El siguiente paso es modificar el fichero `tnsnames.ora`. Este archivo es un componente clave de la configuración de redes en Oracle, utilizado para definir alias que simplifican el acceso a bases de datos. Este archivo se encuentra en la ruta `$ORACLE_HOME/network/admin/` y permite a Oracle identificar y conectarse con servicios locales o remotos.

```
pablo@oracle19c:~$ cat /opt/oracle/product/19c/dbhome_1/network/admin/tnsnames.ora
# tnsnames.ora Network Configuration File: /opt/oracle/product/19c/dbhome_1/network/admin/tnsnames.ora
# Generated by Oracle configuration tools.

ORCLCDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracle)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLCDB)
    )
  )

LISTENER_ORCLCDB =
  (ADDRESS = (PROTOCOL = TCP)(HOST = oracle)(PORT = 1521))

# Esto es lo que tenemos que añadir, lo demás viene por defecto.
PSQLORCL  =
  (DESCRIPTION=
    (ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521))
    (CONNECT_DATA=(SID=PSQLORCL))
    (HS=OK)
  )

```

Ya tenemos todo configurado, solo nos queda reiniciar el servicio listener desde el usuario `oracle`:

```
pablo@oracle19c:~$ sudo su - oracle
oracle@oracle19c:~$ lsnrctl stop

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 17-NOV-2024 12:47:13

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oracle19c)(PORT=1521)))
TNS-12541: TNS:no listener
 TNS-12560: TNS:protocol adapter error
  TNS-00511: No listener
   Linux Error: 111: Connection refused
Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1521)))
TNS-12541: TNS:no listener
 TNS-12560: TNS:protocol adapter error
  TNS-00511: No listener
   Linux Error: 111: Connection refused
oracle@oracle19c:~$ lsnrctl start

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 17-NOV-2024 12:47:21

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Starting /opt/oracle/product/19c/dbhome_1/bin/tnslsnr: please wait...

TNSLSNR for Linux: Version 19.0.0.0.0 - Production
System parameter file is /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
Log messages written to /opt/oracle/diag/tnslsnr/oracle19c/listener/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracle19c)(PORT=1521)))
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oracle19c)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                17-NOV-2024 12:47:23
Uptime                    0 days 0 hr. 0 min. 0 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/oracle19c/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracle19c)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "PSQLORCL" has 1 instance(s).
  Instance "PSQLORCL", status UNKNOWN, has 1 handler(s) for this service...
The command completed successfully
```

En Oracle, un Database Link permite que una base de datos se comunique con otra, ya sea dentro del mismo sistema Oracle o con bases de datos externas, como PostgreSQL. Para crear un Database Link, es necesario que el usuario tenga privilegios adecuados. En mi caso tengo un usuario llamado "pablolink" al que le he otorgado permisos para que pueda crear un *Database Link*:

```
oracle@oracle19c:~$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Nov 17 12:59:14 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Conectado a:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> GRANT CREATE DATABASE LINK TO pablolink;

Concesion terminada correctamente.

SQL> exit
Desconectado de Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
```

Una vez tenemos los permisos para el usuario "pablolink" ya podremos llevar a cabo la creación del enlace, para ello:

```
oracle@oracle19c:~$ sqlplus pablolink/password

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Nov 17 13:07:50 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Hora de Ultima Conexion Correcta: Dom Nov 17 2024 13:06:55 +01:00

Conectado a:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> create database link kk
  2  connect to "pablo" identified by "pablo"
  3  using 'PSQLORCL';

Enlace con la base de datos creado.
```

Donde:

- `CREATE DATABASE LINK`: Especificamos un nombre identificativo para el enlace.
- `CONNECT TO`: Indicamos las credenciales de acceso a la base de datos remota.
- `USING`: Indicamos el nombre del alias de la conexión que previamente hemos definido en el fichero tnsnames.ora.

Una vez creado el enlace, probamos a realizar una consulta que nos muestre los datos de la tabla `asistencias` del servidor PostgreSQL:
```
SQL> SELECT * FROM "asistencias"@kk;

asistencia_id empleado_id fecha    hora_entrada
------------- ----------- -------- ---------------------------------------------
hora_salida
---------------------------------------------
	    1		1 10/11/24 08:00:00
17:00:00

	    2		2 10/11/24 08:30:00
16:30:00

	    3		3 10/11/24 09:00:00
18:00:00


asistencia_id empleado_id fecha    hora_entrada
------------- ----------- -------- ---------------------------------------------
hora_salida
---------------------------------------------
	    4		4 10/11/24 08:15:00
17:15:00

	    5		5 10/11/24 08:45:00
16:45:00
```

¡Y listo! ya hemos interconectado Oracle 19c con PostgreSQL.