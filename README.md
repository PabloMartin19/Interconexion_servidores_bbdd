# PRÁCTICA : Interconexión de Servidores de Bases de Datos

Las interconexiones de servidores de bases de datos son operaciones que pueden ser muy útiles en diferentes contextos. Básicamente, se trata de acceder a datos que no están almacenados en nuestra base de datos, pudiendo combinarlos con los que ya tenemos.

En esta práctica, aprenderemos a configurar interconexiones entre distintos servidores de bases de datos para facilitar el acceso y la combinación de datos almacenados en distintos sistemas. Exploraremos diversas configuraciones de enlaces entre bases de datos de distintos tipos, permitiendo el acceso mutuo entre servidores y el uso simultáneo de datos en múltiples ubicaciones.

Los enlaces que configuraremos son los siguientes:

1. **Oracle** <------> **Oracle**: Enlace directo entre dos servidores Oracle.
2. **PostgreSQL** <------> **PostgreSQL**: Enlace directo entre dos servidores PostgreSQL.
3. **Oracle** <------> **MySQL**: Enlace bidireccional entre Oracle y MySQL.
4. **Oracle** <------> **PostgreSQL**: Enlace bidireccional entre Oracle y PostgreSQL.

Finalmente, realizaremos una consulta que integre información de dos servidores diferentes, aplicando enlaces simultáneos, para demostrar cómo combinar datos desde distintas bases de datos en una sola operación. Esto permitirá comprender la utilidad de los enlaces de bases de datos para gestionar datos distribuidos.

## **Índice**

1. [Interconexión entre servidores Oracle](Oracle-Oracle.md)

2. [Interconexión entre servidores PostgreSQL](Postgre-Postgre.md)  

3. [Interconexión entre Oracle y PostgreSQL](Oracle-Postgre.md)   

--- 