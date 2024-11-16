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




EXPLICAR ESTO. Básicamente son las tablas

postgres@servidor-postgre1:~$ psql
psql (15.8 (Debian 15.8-0+deb12u1))
Digite «help» para obtener ayuda.

postgres=# CREATE USER pablo WITH PASSWORD 'pablo';
CREATE ROLE
postgres=# CREATE DATABASE prueba OWNER pablo;
CREATE DATABASE
postgres=# GRANT ALL PRIVILEGES ON DATABASE prueba TO pablo;
GRANT


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
