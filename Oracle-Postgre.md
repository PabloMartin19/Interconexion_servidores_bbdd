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
[PSQLU]
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
HS_FDS_SHAREABLE_NAME = /usr/lib/x86_64-linux-gnu/odbc/psqlodbcw.so
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


# PostgreSQL a Oracle 19c

En esta segunda parte de la interconexión entre Oracle y Postgre, lo haremos al revés de como lo hicimos anteriormente. Para ello utilizaré las mismas máquinas donde:

- Oracle: `host` --> oracle19c | `IP` --> 192.168.122.195

- PostgreSQL: `host` --> servidor-postgre1 | `IP` --> 192.168.122.163 

Primero vamos a instalar unos paquetes que nos servirán tanto para establecer la conexión con Oracle como a la hora de compilar el Makefile que necesitaremos más adelante:
```
pablo@servidor-postgre1:~$ sudo apt install git build-essential libaio1 postgresql-server-dev-all -y
```
Una vez instalados los paquetes, vamos a descargarnos a través de wget los paquetes de Oracle Instant Client:
```
pablo@servidor-postgre1:~$ sudo su - postgres 
postgres@servidor-postgre1:~$ wget https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-basic-linux.x64-21.1.0.0.0.zip
--2024-11-17 16:22:27--  https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-basic-linux.x64-21.1.0.0.0.zip
Resolviendo download.oracle.com (download.oracle.com)... 2.21.140.94
Conectando con download.oracle.com (download.oracle.com)[2.21.140.94]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 79250994 (76M) [application/zip]
Grabando a: «instantclient-basic-linux.x64-21.1.0.0.0.zip»

instantclient-basic-linux.x64-21.1.0.0.0.zi 100%[========================================================================================>]  75,58M  79,7MB/s    en 0,9s    

2024-11-17 16:22:28 (79,7 MB/s) - «instantclient-basic-linux.x64-21.1.0.0.0.zip» guardado [79250994/79250994]

postgres@servidor-postgre1:~$ wget https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sdk-linux.x64-21.1.0.0.0.zip
--2024-11-17 16:22:53--  https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sdk-linux.x64-21.1.0.0.0.zip
Resolviendo download.oracle.com (download.oracle.com)... 2.21.140.94
Conectando con download.oracle.com (download.oracle.com)[2.21.140.94]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 998327 (975K) [application/zip]
Grabando a: «instantclient-sdk-linux.x64-21.1.0.0.0.zip»

instantclient-sdk-linux.x64-21.1.0.0.0.zip  100%[========================================================================================>] 974,93K  --.-KB/s    en 0,06s   

2024-11-17 16:22:54 (15,0 MB/s) - «instantclient-sdk-linux.x64-21.1.0.0.0.zip» guardado [998327/998327]

postgres@servidor-postgre1:~$ wget https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
--2024-11-17 16:23:04--  https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
Resolviendo download.oracle.com (download.oracle.com)... 2.21.140.94
Conectando con download.oracle.com (download.oracle.com)[2.21.140.94]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 936169 (914K) [application/zip]
Grabando a: «instantclient-sqlplus-linux.x64-21.1.0.0.0.zip»

instantclient-sqlplus-linux.x64-21.1.0.0.0. 100%[========================================================================================>] 914,23K  --.-KB/s    en 0,05s   

2024-11-17 16:23:04 (17,1 MB/s) - «instantclient-sqlplus-linux.x64-21.1.0.0.0.zip» guardado [936169/936169]
```

⚠️ Importante: nos descargamos los paquetes .zip con el usuario postgres

Bien, ya tenemos la descarga hecha, para listar los paquetes:
```
postgres@servidor-postgre1:~$ ls -l
total 79296
drwxr-xr-x 3 postgres postgres     4096 oct  9 18:45 15
-rw-r--r-- 1 postgres postgres 79250994 dic  1  2020 instantclient-basic-linux.x64-21.1.0.0.0.zip
-rw-r--r-- 1 postgres postgres   998327 dic  1  2020 instantclient-sdk-linux.x64-21.1.0.0.0.zip
-rw-r--r-- 1 postgres postgres   936169 dic  1  2020 instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
```
Los descomprimimos:
```
postgres@servidor-postgre1:~$ unzip instantclient-basic-linux.x64-21.1.0.0.0.zip
postgres@servidor-postgre1:~$ unzip instantclient-sdk-linux.x64-21.1.0.0.0.zip
postgres@servidor-postgre1:~$ unzip instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
```

En Linux, los binarios y comandos ejecutables se buscan en las rutas especificadas por la variable de entorno `$PATH`. Si un binario no está en una de estas rutas, no podrá ejecutarse simplemente escribiendo su nombre en la terminal.

En este caso, el directorio donde se han descomprimido los binarios (`/home/postgres/instantclient_21_1`) no está incluido en la variable `$PATH`, por lo que, para usar los binarios como sqlplus, tendríamos que proporcionar la ruta completa cada vez. Esto es incómodo y poco práctico.

Por lo tanto, agregamos las siguientes líneas al archivo de configuración `~/.bashrc`:
```
echo 'export ORACLE_HOME=/var/lib/postgresql/instantclient_21_1' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME' >> ~/.bashrc
echo 'export PATH=$PATH:$ORACLE_HOME' >> ~/.bashrc
```
Luego, cargamos el archivo de configuración para aplicar los cambios:
```
source ~/.bashrc
```
Comprobamos que se han hecho correctamente las variables de entorno:
```
postgres@servidor-postgre1:~$ which sqlplus 
/var/lib/postgresql/instantclient_21_1/sqlplus
```

Para comprobar el correcto funcionamiento, vamos a realizar una conexión remota, al igual que haríamos en una situación real:
```
postgres@servidor-postgre1:~$ sqlplus pablolink/password@192.168.122.195:1521/ORCLCDB

SQL*Plus: Release 21.0.0.0.0 - Production on Sun Nov 17 17:44:30 2024
Version 21.1.0.0.0

Copyright (c) 1982, 2020, Oracle.  All rights reserved.

Hora de Ultima Conexion Correcta: Dom Nov 17 2024 17:43:48 +01:00

Conectado a:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL>
```
Y como vemos funciona correctamente.

Ahora, voy a clonar el siguiente repositorio [Laurenz](https://github.com/laurenz/oracle_fdw). En donde se encuentra el código fuente necesario para la compilación de **oracle_fdw**.
```
postgres@servidor-postgre1:~$ git clone https://github.com/laurenz/oracle_fdw.git
Clonando en 'oracle_fdw'...
remote: Enumerating objects: 2870, done.
remote: Counting objects: 100% (968/968), done.
remote: Compressing objects: 100% (95/95), done.
remote: Total 2870 (delta 906), reused 917 (delta 873), pack-reused 1902 (from 1)
Recibiendo objetos: 100% (2870/2870), 1.54 MiB | 6.67 MiB/s, listo.
Resolviendo deltas: 100% (2019/2019), listo.
```

Verificamos que tenemos el contenido y que la clonación es válida:
```
postgres@servidor-postgre1:~$ ls -l
total 16
drwxr-xr-x 3 postgres postgres 4096 oct  9 18:45 15
drwxr-xr-x 4 postgres postgres 4096 nov 17 16:29 instantclient_21_1
drwxr-xr-x 6 postgres postgres 4096 nov 17 17:54 oracle_fdw
drwxr-xr-x 3 postgres postgres 4096 nov 17 16:37 oradiag_postgres
```

Ahora nos moveremos al directorio que se ha creado, y listamos su contenido:
```
postgres@servidor-postgre1:~$ cd oracle_fdw/
postgres@servidor-postgre1:~/oracle_fdw$ ls -l
total 500
-rw-r--r-- 1 postgres postgres  28436 nov 17 17:54 CHANGELOG
drwxr-xr-x 2 postgres postgres   4096 nov 17 17:54 expected
-rw-r--r-- 1 postgres postgres   1059 nov 17 17:54 LICENSE
-rw-r--r-- 1 postgres postgres   1475 nov 17 17:54 Makefile
drwxr-xr-x 2 postgres postgres   4096 nov 17 17:54 msvc
-rw-r--r-- 1 postgres postgres    231 nov 17 17:54 oracle_fdw--1.0--1.1.sql
-rw-r--r-- 1 postgres postgres    240 nov 17 17:54 oracle_fdw--1.1--1.2.sql
-rw-r--r-- 1 postgres postgres   1244 nov 17 17:54 oracle_fdw--1.2.sql
-rw-r--r-- 1 postgres postgres 229598 nov 17 17:54 oracle_fdw.c
-rw-r--r-- 1 postgres postgres    133 nov 17 17:54 oracle_fdw.control
-rw-r--r-- 1 postgres postgres   9156 nov 17 17:54 oracle_fdw.h
-rw-r--r-- 1 postgres postgres  44511 nov 17 17:54 oracle_gis.c
-rw-r--r-- 1 postgres postgres 104953 nov 17 17:54 oracle_utils.c
lrwxrwxrwx 1 postgres postgres     17 nov 17 17:54 README.md -> README.oracle_fdw
-rw-r--r-- 1 postgres postgres  44112 nov 17 17:54 README.oracle_fdw
drwxr-xr-x 2 postgres postgres   4096 nov 17 17:54 sql
-rw-r--r-- 1 postgres postgres    948 nov 17 17:54 TODO
```

Por tanto, para llevar a cabo dicha compilación e instalar el resultado en los correspondientes directorios de la máquina, ejecutaremos los comandos:
```
postgres@servidor-postgre1:~/oracle_fdw$ make
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -I"/var/lib/postgresql/instantclient_21_1/sdk/include" -I"/var/lib/postgresql/instantclient_21_1/oci/include" -I"/var/lib/postgresql/instantclient_21_1/rdbms/public" -I"/var/lib/postgresql/instantclient_21_1/"  -I. -I./ -I/usr/include/postgresql/15/server -I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2   -c -o oracle_fdw.o oracle_fdw.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -I"/var/lib/postgresql/instantclient_21_1/sdk/include" -I"/var/lib/postgresql/instantclient_21_1/oci/include" -I"/var/lib/postgresql/instantclient_21_1/rdbms/public" -I"/var/lib/postgresql/instantclient_21_1/"  -I. -I./ -I/usr/include/postgresql/15/server -I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2   -c -o oracle_utils.o oracle_utils.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -I"/var/lib/postgresql/instantclient_21_1/sdk/include" -I"/var/lib/postgresql/instantclient_21_1/oci/include" -I"/var/lib/postgresql/instantclient_21_1/rdbms/public" -I"/var/lib/postgresql/instantclient_21_1/"  -I. -I./ -I/usr/include/postgresql/15/server -I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2   -c -o oracle_gis.o oracle_gis.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -shared -o oracle_fdw.so oracle_fdw.o oracle_utils.o oracle_gis.o -L/usr/lib/x86_64-linux-gnu  -Wl,-z,relro -Wl,-z,now -L/usr/lib/llvm-14/lib  -Wl,--as-needed  -L"/var/lib/postgresql/instantclient_21_1/" -L"/var/lib/postgresql/instantclient_21_1/bin" -L"/var/lib/postgresql/instantclient_21_1/lib" -L"/var/lib/postgresql/instantclient_21_1/lib/amd64"  -lclntsh 
```
```
postgres@servidor-postgre1:~/oracle_fdw$ sudo make install
/bin/mkdir -p '/usr/lib/postgresql/15/lib'
/bin/mkdir -p '/usr/share/postgresql/15/extension'
/bin/mkdir -p '/usr/share/postgresql/15/extension'
/bin/mkdir -p '/usr/share/doc/postgresql-doc-15/extension'
/usr/bin/install -c -m 755  oracle_fdw.so '/usr/lib/postgresql/15/lib/oracle_fdw.so'
/usr/bin/install -c -m 644 .//oracle_fdw.control '/usr/share/postgresql/15/extension/'
/usr/bin/install -c -m 644 .//oracle_fdw--1.2.sql .//oracle_fdw--1.0--1.1.sql .//oracle_fdw--1.1--1.2.sql  '/usr/share/postgresql/15/extension/'
/usr/bin/install -c -m 644 .//README.oracle_fdw '/usr/share/doc/postgresql-doc-15/extension/'
```

Después de que la compilación se haya completado con éxito y la extensión haya sido instalada en los directorios locales de PostgreSQL, ya está lista para utilizarse. No obstante, será necesario especificar manualmente la ruta a dichas bibliotecas. Para ello: 
```
postgres@servidor-postgre1:~/oracle_fdw$ echo '/home/postgres/instantclient\_21\_1' | sudo tee /etc/ld.so.conf.d/oracle.conf
/home/postgres/instantclient\_21\_1
```

Y por último, tendremos que generar los enlaces necesarios y cargar en memoria las nuevas librerías compartidas:
```
postgres@servidor-postgre1:~/oracle_fdw$ sudo ldconfig
```

⚠️ IMPORTANTE: la ruta que había puesto no era correcta, por lo que la he modificado para que apunte a la verdadera ruta:
```
postgres@servidor-postgre1:~$ cat /etc/ld.so.conf.d/oracle.conf
/var/lib/postgresql/instantclient_21_1
postgres@servidor-postgre1:~$ pwd
/var/lib/postgresql
```

Una vez ya estamos seguros de que está todo correcto, 
```
postgres@servidor-postgre1:~$ psql -d prueba
psql (15.9 (Debian 15.9-0+deb12u1))
Digite «help» para obtener ayuda.

prueba=# CREATE EXTENSION oracle_fdw;
CREATE EXTENSION
```
Verificamos que la extensión `oracle_fdw` esté correctamente creada:
```
prueba=# \dx
                     Listado de extensiones instaladas
   Nombre   | Versión |  Esquema   |              Descripción               
------------+---------+------------+----------------------------------------
 oracle_fdw | 1.2     | public     | foreign data wrapper for Oracle access
 plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 filas)
```

A continuación, crearé un esquema llamado `oracle` en PostgreSQL para almacenar las tablas importadas desde Oracle:
```
prueba=# CREATE SCHEMA oracle;
CREATE SCHEMA
```
Luego, configuraré el servidor Oracle en PostgreSQL utilizando la extensión `oracle_fdw`. Este paso permite conectar PostgreSQL con la base de datos Oracle usando los parámetros de conexión adecuados (en este caso, la IP del servidor Oracle y el nombre del contenedor ORCLCDB):
```
prueba=# CREATE SERVER oracle FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver '//192.168.122.195/ORCLCDB');
CREATE SERVER
```

El siguiente paso será crear un mapeo de usuario entre PostgreSQL y Oracle, proporcionando las credenciales necesarias para acceder a Oracle (usuario y contraseña):
```
prueba=# CREATE USER MAPPING FOR pablo SERVER oracle OPTIONS (user 'pablolink', password 'password');
CREATE USER MAPPING
```

Le otorgamos privilegios a `pablo` sobre el esquema `oracle` y sobre el servidor externo oracle para asegurar que el usuario tenga acceso a las tablas importadas desde Oracle:
```
prueba=# GRANT ALL PRIVILEGES ON SCHEMA oracle TO pablo;
GRANT
prueba=# GRANT ALL PRIVILEGES ON FOREIGN SERVER oracle TO pablo;
GRANT
```

Por último, importamos el esquema de Oracle a PostgreSQL, para ello utilizaré el comando IMPORT FOREIGN SCHEMA. Esto permite que las tablas de Oracle estén disponibles en PostgreSQL bajo el esquema previamente creado (oracle):
```
prueba=> IMPORT FOREIGN SCHEMA "PABLOLINK" FROM SERVER oracle INTO oracle;
IMPORT FOREIGN SCHEMA
```

Ya solo nos queda comprobar que todo ha salido bien, para ello hacemos una consulta a la tabla `dept` del servidor Oracle:
``` 
prueba=> SELECT * FROM oracle.dept;
 deptno |   dname    |   loc    
--------+------------+----------
     10 | ACCOUNTING | NEW YORK
     20 | RESEARCH   | DALLAS
     30 | SALES      | CHICAGO
     40 | OPERATIONS | BOSTON
(4 filas)
```

Y como vemos ha funcionado perfectamente.