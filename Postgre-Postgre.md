# **Interconexión entre dos servidores con PostgreSQL**

Para la realización de esta parte de la práctica, se implementará una interconexión de bases de datos entre dos instancias de PostgreSQL mediante la extensión **dblink**, que permite la comunicación entre diferentes bases de datos PostgreSQL. Para este propósito, se configurarán dos máquinas virtuales Debian 12 con PostgreSQL instalado, con las siguientes especificaciones:

- **servidor-postgre1**: 192.168.1.45
- **servidor-postgre2**: 192.168.1.46

Ambas máquinas estarán conectadas a la interfaz de red **br0** para facilitar la comunicación en una red local.

## **Configuración de Acceso en pg_hba.conf**

Para permitir que cada instancia de PostgreSQL acceda a la otra, es necesario realizar configuraciones en el archivo **pg_hba.conf** en ambas máquinas. Este archivo, ubicado generalmente en **/etc/postgresql/15/main/pg_hba.conf**, controla las reglas de acceso a la base de datos, estableciendo qué usuarios pueden conectarse y desde qué direcciones IP. Se deben añadir las siguientes líneas para permitir el acceso entre **servidor-postgre1** y **servidor-postgre2**:

En **servidor-postgre1**:

```
host    all             all             192.168.1.46/32          md5
```

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXeS5tC8xH-qsiQpQzYnLMxkfMe8fr_JQKSsg7_q2I5s56aCWvHOvvDfVFHo3ouweF30gYNxAghm1qzoqog77KxhtioEsiMDtfJPqZ3pDwpjetq2Fw5T0lYGQd6ghpnrfqWQeiKqAw?key=zND1MNMwb0ABC8sdufG-lxz9)

En **servidor-postgre2**:

```
host    all             all             192.168.1.45/32          md5
```

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXc4-LQ9MeonauYY7v8sk_-XyJwfdRRtE7UeYeQf1lPcqxv0JXKAHEAfAiDMKylbkVjgzRQW4f7DxzT_rbohaRen0CtxFb-5PESk8g_itoEvMm6wvb63Ef2HOzl4sZfYx92YrB1c7A?key=zND1MNMwb0ABC8sdufG-lxz9)

Después de añadir las líneas para permitir el acceso debemos reiniciar el servicio de PostgreSQL → sudo systemctl restart postgresql

## **Verificación de Puertos con netstat**

PostgreSQL, por defecto, escucha en el puerto **5432** para conexiones TCP. Utilizando el comando netstat -tln (sudo apt install net-tools), podemos verificar que PostgreSQL está escuchando en este puerto en ambas máquinas. Al ejecutar este comando, se debería ver una línea similar a la siguiente:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXei2b6YvnR1BLMvApKHPaMRf2So8Rc5Y0FKBq6XJpR1tt8UNvFKzMdDYOwTSPuauwFecdwJduleZseVYrWRs9r-muY7n18nbWwHHHyf1xAjzaac3F2-eEc5VHTSpJvzNsxsxi2DOw?key=zND1MNMwb0ABC8sdufG-lxz9)

La primera línea indica que el puerto **5432** está en estado **LISTEN**, permitiendo que PostgreSQL reciba conexiones en todas las interfaces de red configuradas, incluidas las conexiones que llegarán a través de la interfaz **br0**.

Esta configuración permitirá la correcta interconexión entre las bases de datos de ambas máquinas virtuales usando dblink.

## **Creación de Usuarios con Permisos**

En esta parte, procederemos a la creación de usuarios y tablas en ambas instancias de PostgreSQL. Para ello, utilizaremos el usuario **postgres** en cada servidor y, a partir de ahí, crearemos usuarios específicos para gestionar las tablas en cada uno.

En **servidor-postgre1**:

- Usuario: **pablo1**
- Contraseña: **password**

Comandos SQL para crear el usuario y otorgarle permisos:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdgS9roxdBNR7m97J-UF1WJ9tFCIbglTb5Jxq6yA13iRxHdz5vxKSYKYHAGBhFJJ_OfYHIVvqo0WkveLTylMihkmYY5oygAyeYK0gQZajC3L3ZPmSpxYtwd8JvxxxEZfTlGOJP4mQ?key=zND1MNMwb0ABC8sdufG-lxz9)

En **servidor-postgre2**:

- Usuario: **pablo2**
- Contraseña: **password**

Comandos SQL para crear el usuario y otorgarle permisos:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdJ75WJqgDO8NvgKOxo-Tmd1MS-Gs35YeR4qOaXe46J98vWXzxDjyCi-QAr4LKR6MwmzoIM0bX8yD4WyV8evB5zlSXvnHdGSYUlU_C_NCfAz_gTmzeWW07qYtsBAMezY9Uf_93LDQ?key=zND1MNMwb0ABC8sdufG-lxz9)

Los permisos otorgados a estos usuarios permitirán que ambos tengan control sobre la creación de bases de datos y la gestión de tablas en sus respectivas instancias.

## **Crear las Bases de Datos en Cada Servidor**

A continuación, creamos una base de datos en cada servidor donde se alojarán las tablas y el esquema **scott**:

En **servidor-postgre1**:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcD-EBcoXksoRam0suQPwUr-gB9tV0lhq6-zZa7LdllktZMhS24YbA4Voi4MAQX9dWCIZj40j4beD5jjksFQrTxR56sIRibU7PQRhphRQ2_UtC-nQp4ZTkvXxuzhw6TNGo-fJn9CQ?key=zND1MNMwb0ABC8sdufG-lxz9)

**En servidor-postgre2:**

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfDR90wv0ogI7BCA1_PSGJ9gBbkohXUUN8z1kc5UjVHZtngFIthKXLnJQcjvPgqrPmfl7Vo0XexSU8s_Cemg9bIizVT8F3d8AA2UYQbypmChp4VqJapA6Vh48cA5e_mxlINT-ukJQ?key=zND1MNMwb0ABC8sdufG-lxz9)

Le damos permisos a los usuarios creados anteriormente:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXeYEB5xVI2LVJsZTLLRULeGz7eIPf7ilI0gKITt6X4hIj2PQGMBU6dO8yLbIWDJu_b6t2zx93NHkItISb7rNzJZJr-ijUju25spx011wz9tefIXPp4EWGLQAIDKhslylvzN3R8kwg?key=zND1MNMwb0ABC8sdufG-lxz9)

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcYx4LasaHBhF8K2rK4l236xMtFHeNg1Si91_rpKYLmz9h4UtXvB4HaX2xlcc45bWWU22T-RCWwnqqYHHC2EGIW0V__tthBltuN7M76K9IvKLXRWNqZw1XzlzRCA8a10ygqjpiEuQ?key=zND1MNMwb0ABC8sdufG-lxz9)

## **Creación de Tablas en el Esquema scott**

Una vez creados los usuarios y las bases de datos, nos conectaremos a PostgreSQL usando los usuarios **pablo1** en **servidor-postgre1** y **pablo2** en **servidor-postgre2** para crear las tablas bajo el esquema scott. El esquema scott actuará como un espacio de trabajo organizado para estas tablas.

Primero, probaremos las conexiones en cada servidor:

En **servidor-postgre1**:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXecUKStcuk82cTC4rAie--YEOPZrJM4AKyfW75_FJptsAbsnFxCVuqZMat8_M_147M3K8V-BkYJRnh37rkl_cDPHNITvU3K8cA_9h_J42AvI-1JJOdddAU6tTo-5MUI9_ev5jpEHQ?key=zND1MNMwb0ABC8sdufG-lxz9)

Desglose del comando:

- **U**: Especificamos el rol con el que nos queremos conectar, en este caso, pablo1.
- **d**: Especificamos la base de datos a la que nos queremos conectar, en este caso, db_postgre1.
- **h**: Especificamos el nombre de la máquina a la que nos queremos conectar, en este caso, localhost.

En **servidor-postgre2**:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXeTVKrOEO4_dcfckCHIGaf8Ivb0MeRgLEc8ML6dzOM03ANnHWJ0JSCr_7PA6UwPcDRiy97iL7gAVK_4G9Ol7A9HJTWZz3w7WSwR9iWhc1DlDOh_EQ2kMUsGOEtoD1DX1ZETnbDvCA?key=zND1MNMwb0ABC8sdufG-lxz9)

A continuación, voy a crear las tablas en cada una de las máquinas. La única variación es que el **servidor-postgre2** no tendrá la tabla **bonus**, ya que la crearemos en el **servidor-postgre1**.

Para la creación he utilizado un script que tengo en mi [GitHub](https://github.com/PabloMartin19/schema_scott/blob/main/scott.sql):

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfpA3LNUIJepzKXX_OkgRy97u1dwbl-zEdw1TOro1mAdp8NbloIU8K_XI3pk_A_PKxUjLJU89CjyW1G0-Sw9TJbdxSm11K7QxwrlFgHiIz6B8n5iAyf3IO5PesG72YVTWTutEH2WA?key=zND1MNMwb0ABC8sdufG-lxz9)

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdvck3kLkCNiRH7RGc0HnxMUFXe-V5wXpjDumhSzukzwAyfyaFeYDo0Gf-eg9tWwZotc66CB9TAkWDBphW1n3jqAJ1zMB8USrUMuHo6ugMWz0hQ3dduTUE9PED22mZRLOcWNzlA?key=zND1MNMwb0ABC8sdufG-lxz9)

## **Crear los enlaces**

Para crear los enlaces utilizaremos la extensión dblink, para ello nos conectamos a la base de datos en la que deseamos usarla (en este caso, db_postgre1) y ejecutamos el siguiente comando SQL:

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdvBlJ5zrhzprJ1wTwrcav9Mk6Rt3juNCOgnaaOUGlqAgmpkFHgVk-Vst0vpJEatZZTKF24nJtRgTGFX_5jIzgqxAWLYatSH4XTN2YcgvUBu9h0-lTxgzTYOIiglF3O8qjNv3MHyQ?key=zND1MNMwb0ABC8sdufG-lxz9)

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXenO-Tfuip3Bcsrj6VnaQyOOHsU6GID2yy-byt4gKxpJufTzMtzsfPn7T-HQF8ZgMZLqMui24nSPt0AJ3QvGcA0FmB2QDQJfJwPyQJ7E50Oy1c0Ggx4Gf8IkH-9tLnRpIlVWEKy?key=zND1MNMwb0ABC8sdufG-lxz9)

Bien, una vez tenemos creadas ambas extensiones ya podemos empezar a realizar consultas. Es importante recalcar que no es necesario crear las extensiones en ambos servidores en caso de que solo quisiéramos conectarnos de uno al otro, y no entre ambos.

**Primera llamada a dblink (tabla bonus)**

En esta primera llamada voy a realizar una consulta sencilla a la tabla “bonus” ya que esta es la que falta en el servidor2.

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXf4O8anxLtS3SYjgqQMHDaWY9ISq7EGZq6L5lPBeO24DBqgP9UPyFUZsIYsqh24c2f3JePx1ujvPKSOyJr7sV_8262mgQ-oCLFSqAr5CK3UbTWXkQsdlPGqcJ4pHVgq-jbIyOJZXQ?key=zND1MNMwb0ABC8sdufG-lxz9)

Desglose de consulta con dblink:

FROM dblink(...):

- Utiliza la función dblink para establecer una conexión remota a otro servidor de PostgreSQL y ejecutar una consulta en él.
- dblink requiere dos argumentos principales:
    1. **Cadena de conexión**: Describe cómo conectarse al servidor remoto.
    2. **Consulta remota**: Especifica la consulta SQL que se ejecutará en la base de datos remota.

**Cadena de conexión:**

`'host=192.168.1.45 user=pablo1 password=password dbname=db_postgre1'`

`host=192.168.1.45`: La dirección IP del servidor remoto donde se encuentra la base de datos a la que queremos conectarnos.

`user=pablo1`: El nombre de usuario que se usará para conectarse al servidor remoto.

`password=password`: La contraseña del usuario pablo1.

`dbname=db_postgre1`: El nombre de la base de datos en el servidor remoto a la que queremos conectarnos.

**Segunda y tercera llamada a dblink**

La segunda llamada al **dblink** tiene como objetivo inspeccionar la estructura de la tabla bonus en el servidor remoto (**servidor-postgre1**). Para lograrlo, se conecta al servidor remoto utilizando las credenciales proporcionadas y consulta la vista **information_schema.columns**. Esta vista contiene metadatos sobre las tablas y sus columnas en la base de datos, como nombres de columnas, tipos de datos y si las columnas permiten valores nulos. Filtrando por el nombre de la tabla (bonus) y el esquema (public), se obtiene una tabla virtual en el servidor local que describe cómo está estructurada la tabla bonus en el servidor remoto. Esto proporciona información valiosa para replicar la tabla, como los tipos de datos de las columnas y sus restricciones.

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfVh9Tbq53-Ig32RAcsTMJDt7Axl7FMc7zDkmBVpu2SK4qScPiPIPKx0Ov5ecYiwZaIMQVEMzPo2jpZojdprQXNc4c47sb9hYDscsA1x70MYjBrthngN9Xvce6tNrPzgyF-1Zxkfw?key=zND1MNMwb0ABC8sdufG-lxz9)

En la tercera llamada, se utiliza nuevamente dblink, pero esta vez para transferir los datos de la tabla bonus desde el servidor remoto al servidor local. Aquí, el comando ejecuta una consulta SELECT en la base de datos remota para obtener los registros de la tabla bonus. Luego, con esta información, se crea una nueva tabla llamada bonus en el servidor local. La consulta define explícitamente los tipos de datos de las columnas (empno integer, bonus integer) en el resultado del dblink, asegurando que la estructura de la tabla local coincida con la del servidor remoto. Este proceso permite traer tanto la estructura como los datos de la tabla remota y replicarlos localmente.

Finalmente, se verifican los resultados de ambas operaciones. Con el comando \dt, se confirma que la tabla bonus ha sido creada en el servidor local, y con \d bonus se verifica que su estructura corresponde a lo que se esperaba, es decir, con las columnas empno y bonus, incluyendo las restricciones identificadas previamente (por ejemplo, que empno no permite valores nulos). Este flujo asegura que los datos y la estructura de la tabla bonus del servidor remoto han sido correctamente replicados en el servidor local.

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXda53WVeJhrTdqg2RJ2ElEbmSKBeYR_xhHS-i7efjYtfjvfRjjXM3kBRm-2cyqUBiDYRIsaxOj4WdOqRkUXYim1RJ1yKCRYxskJbYvEApri-tMea7oneB2Ok72CIuMfUzs3Hvgn?key=zND1MNMwb0ABC8sdufG-lxz9)

![image](https://lh7-rt.googleusercontent.com/docsz/AD_4nXep-xyHngMjDk09TQbbFda4x6uq2PFenYuFunGAGLWfrIpigs4_Ne-wq-4VkwALp6I9g1-8IZVEI2lxvnK4fUaODzyFQ7Dg-iknrSOcT96Xmk2_zxALKYI7vNuf401i8Ru9D4B1fQ?key=zND1MNMwb0ABC8sdufG-lxz9)