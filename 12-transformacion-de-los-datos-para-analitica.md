## Capítulo 12: Transformación de Datos para Analítica

Ahora que hemos replicado los datos de los sistemas transaccionales en nuestro entorno analítico, debemos transformar el modelo de datos de una forma normalizada y optimizada para transacciones a un esquema en estrella (*star schema*), optimizado para analítica.

Como recordatorio, diferenciamos entre los enfoques de extracción, transformación y carga (*extract, transform, and load* o ETL) y extracción, carga y transformación (*extract, load, and transform* o ELT) para mover datos de sistemas transaccionales a sistemas analíticos, y para optimizar el modelo de datos y permitir consultas más sencillas y eficientes.

PostgreSQL es una excelente solución para transacciones y analítica, proporcionando herramientas para la replicación de datos y capacidades robustas de transformación de datos, lo que nos permite implementar un proceso ELT sin depender de herramientas de terceros. Podemos aprovechar estas capacidades para hacerlo todo con PostgreSQL, reduciendo la complejidad operativa y utilizando licencias de código abierto.

El capítulo anterior mostró cómo mover datos del sistema transaccional al sistema analítico mediante un proceso de extracción y carga (los pasos "E" y "L"). En este capítulo, profundizaremos en las mejores formas de transformar y optimizar esos datos para la analítica (el paso "T"). Revisaremos tres enfoques para crear un esquema en estrella: vistas (*views*), vistas materializadas (*materialized views*), y disparadores (*triggers*) y tablas. Terminaremos con una rápida comparación de rendimiento y orientación para el desarrollador.

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Requisitos técnicos

El código de muestra para las tres transformaciones está disponible en el repositorio de GitHub. El enfoque de solo vistas se implementa en `psql_scripts/database_definitions/central_analytics_stars/vo_analytics.sql`. El código para el enfoque de vistas materializadas está en `psql_scripts/database_definitions/central_analytics_stars/mv_analytics.sql`.

El código para el enfoque de tablas y disparadores se encuentra en el archivo `psql_scripts/database_definitions/central_analytics_stars/tt_analytics.sql`. Para acceder al enlace del repositorio, sigue los pasos en la sección "Download the example code files" en el Prefacio.

---

### Tres formas de crear un esquema en estrella centrado en la analítica

Describiremos tres enfoques para utilizar las capacidades nativas de PostgreSQL para transformar datos normalizados (ver Figura 12.1) en un esquema en estrella (ver Figura 12.2):

- El enfoque de solo vistas utiliza vistas de PostgreSQL para mejorar la experiencia de consulta y permitir un acceso fácil al esquema en estrella.
- El enfoque de vistas materializadas se basa en el modelo de solo vistas y permite consultas más rápidas en conjuntos de datos más grandes.
- El enfoque de disparadores y tablas crea un nuevo esquema basado en tablas y utiliza disparadores de inserción/actualización/eliminación para mantener los datos analíticos sincronizados con el esquema normalizado.

*Figura 12.1: El esquema de datos normalizado optimizado para transacciones*

Los tres enfoques se basan en el esquema de datos normalizado, replicado desde la base de datos de referencia y los dos sistemas de comercio electrónico (ver Figura 12.1), y todos producen el mismo esquema en estrella (ver Figura 12.2). Agregamos dos tablas auxiliares para simular la entrada de un sistema de gestión de ventas que nos ayude a detallar la información sobre las regiones de ventas y las regiones geográficas.

*Figura 12.2: El esquema en estrella para la analítica de ventas*

Si bien los tres enfoques producen el mismo esquema en estrella que se muestra en la Figura 12.2, utilizan métodos diferentes para implementar la transformación y exhiben características de rendimiento diferentes.

Para que los enfoques y los resultados sean comparables, creamos tres esquemas:

- El esquema de solo vistas, `vo_analytics`
- El esquema de vistas materializadas, `mv_analytics`
- El esquema basado en tablas y disparadores, `tt_analytics`

Los tres esquemas se definen en la base de datos `central_analytics`, y mostraremos cómo poblarlos y mantenerlos utilizando las tres técnicas diferentes.

#### Uso de vistas

Las vistas son un concepto fundamental de SQL. Nos permiten encapsular consultas complejas, darles nombres, almacenarlas en la base de datos bajo esos nombres y hacerlas accesibles y reutilizables para otros usuarios. El siguiente ejemplo muestra una consulta simple (`SELECT …`) que encapsulamos y nombramos, para que podamos llamarla con `SELECT * FROM transactions_per_month` en lugar de especificar la consulta completa cada vez:

```sql
CREATE OR REPLACE VIEW transactions_per_month AS
SELECT
    EXTRACT(YEAR FROM transaction_date) AS year,
    EXTRACT(MONTH FROM transaction_date) AS month,
    COUNT(id) AS total_transactions
FROM sales_transaction
WHERE transaction_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY year, month
ORDER BY year, month;
```

Las vistas son excelentes herramientas para reducir la complejidad y fomentar la reutilización, pero son solo "consultas con nombre"; ¡no tienen un mejor rendimiento que la consulta subyacente, que se ejecuta cada vez que se accede a la vista!

Las vistas facilitan enormemente la creación de una perspectiva virtual alternativa sobre datos normalizados.

Primero, definimos una dimensión de fecha y hora que descompone la fecha en las transacciones de ventas en año, trimestre, mes, fin de semana, día de la semana y día del mes, lo que admite consultas basadas en fechas en múltiples niveles de agregación. Para permitir consultas de ventas en una fecha específica, incluso si no hubo ninguna en esa fecha, generamos esta dimensión a partir de una serie de datos:

```sql
CREATE OR REPLACE VIEW vo_analytics.vw_dim_date AS
SELECT
    d::DATE AS date_key,
    EXTRACT(DAY FROM d) AS day,
    EXTRACT(MONTH FROM d) AS month,
    EXTRACT(QUARTER FROM d) AS quarter,
    EXTRACT(YEAR FROM d) AS year,
    EXTRACT(DOW FROM d) AS day_of_week,
    CASE WHEN EXTRACT(DOW FROM d) IN (0, 6) THEN true ELSE false END AS is_weekend
FROM GENERATE_SERIES('2020-01-01'::DATE, '2030-12-31'::DATE, INTERVAL '1 day') AS d;
```

La dimensión de cliente y ubicación integra datos de la tabla `customer` con datos de referencia de tablas auxiliares para admitir el desglose (*drill-down*) o consolidación (*roll-up*) por país, región geográfica, territorio de ventas, estado, ciudad y código postal (*ZIP code*):

```sql
CREATE VIEW vo_analytics.vw_dim_customer_location AS
SELECT
    c.id AS customer_id,
    auxiliary.parse_zipcode_postalcode(c.postal_code) AS zipcode,
    city,
    state_code,
    state_name,
    st.territory_name sales_territory,
    ds.region geographic_region,
    c.country
FROM customer c
JOIN auxiliary.us_state ds ON ds.state_code = auxiliary.parse_state_postalcode(c.postal_code)
JOIN auxiliary.sales_territory st ON st.us_state_code = ds.state_code
WHERE c.country = 'US';
```

La dimensión del producto integra el país de origen, la marca, la categoría, la etiqueta, el color, el tamaño y los atributos:

```sql
CREATE OR REPLACE VIEW vo_analytics.vw_dim_product AS
SELECT
    pv.id as product_variant_id,
    attributes,
    pvp.price as current_price,
    pv.attributes->>'size' as size,
    pv.attributes->>'color' as color,
    p.label,
    c.label AS category,
    b.label AS brand,
    co.name AS co_name,
    co.alpha3_code AS co_alpha3_code
FROM product_variant pv
JOIN product_variant_price pvp ON pv.id = pvp.product_variant_id AND pvp.current = true
JOIN product p ON pv.product_id = p.id
JOIN category c ON p.category_id = c.id
JOIN brand b ON p.brand_id = b.id
JOIN country_of_origin co ON co.brand_id = b.id;
```

Finalmente, los hechos reflejan los datos detallados de ventas con dos métricas: precio y cantidad:

```sql
CREATE VIEW vo_analytics.vw_fact_sales AS
SELECT
    stl.id sales_transaction_line_id,
    st.transaction_date AS date,
    stl.product_variant_id,
    st.customer_id,
    stl.qty AS quantity,
    stl.price_at_sale AS unit_price,
    (stl.qty * stl.price_at_sale) AS sales_amount
FROM sales_transaction_line stl
JOIN sales_transaction st ON stl.sales_transaction_id = st.id;
```

Los hechos se pueden unir directamente con las dimensiones, lo que facilita la creación de consultas analíticas avanzadas sin tener que repetir todas las uniones (*joins*) subyacentes cada vez. Por ejemplo, esta consulta agrupa las ventas por estado y ciudad con subtotales en cada nivel:

```sql
SELECT
    state_name,
    COALESCE(city, 'Total'),
    SUM(sales_amount) AS total_sales_amount
FROM vo_analytics.vw_fact_sales
JOIN vo_analytics.vw_dim_customer_location AS cl ON vw_fact_sales.customer_id = cl.customer_id
JOIN vo_analytics.vw_dim_date AS dd ON vw_fact_sales.date = dd.date_key
WHERE year = 2024
GROUP BY ROLLUP (state_name, city)
ORDER BY state_name, city ASC NULLS LAST;
```

¡Imagínate la complejidad de esta consulta si hubiera que repetir las definiciones de las dimensiones y los hechos!

Si bien las vistas son fáciles de escribir, cómodas de usar y nos ayudan a encapsular una lógica de negocio compleja, no aceleran la ejecución de las consultas. Las vistas se regeneran al acceder, cada vez.

Un enfoque de solo vistas no es adecuado para producción cuando se trabaja con grandes conjuntos de datos o cuando se presta servicio a muchos usuarios concurrentes.

#### Uso de vistas materializadas

Las vistas materializadas se definen igual que las vistas, pero entre bastidores almacenan físicamente el resultado de la consulta. Deben actualizarse periódicamente; de lo contrario, sus datos se vuelven obsoletos (*stale*). Dado que una vista materializada almacena el resultado físico de una consulta en lugar de generarlo cada vez, es mucho más rápida que una vista normal.

Las vistas materializadas también se pueden indexar por separado de las tablas base subyacentes. Se pueden actualizar de forma concurrente, siempre que tengan un índice único, lo que permite que la vista permanezca accesible mientras se regenera el almacenamiento físico. ¡La opción de actualización concurrente mantiene la vista legible durante la actualización!

Generar vistas materializadas a partir de las vistas que acabamos de definir es sencillo. Los índices se crean de la misma manera que los índices de tablas.

Aquí están las definiciones para la vista materializada para los hechos de ventas. Las vistas materializadas deben tener un índice único para permitir actualizaciones concurrentes. Se recomienda encarecidamente crear índices en las claves foráneas para las tablas de hechos (y vistas materializadas) para mejorar el rendimiento de las consultas:

```sql
-- create the sales fact materialized view
CREATE MATERIALIZED VIEW mv_analytics.mv_fact_sales AS
SELECT * FROM vo_analytics.vw_fact_sales;

-- create an index on sales_transaction_line_id for faster joins and concurrent refreshes
CREATE UNIQUE INDEX idx_mv_fact_sales ON mv_analytics.mv_fact_sales (sales_transaction_line_id);

-- indexes on foreign keys for faster joins
CREATE INDEX idx_mv_fact_sales_date ON mv_analytics.mv_fact_sales (date);
CREATE INDEX idx_mv_fact_sales_product_variant ON mv_analytics.mv_fact_sales (product_variant_id);
CREATE INDEX idx_mv_fact_sales_customer ON mv_analytics.mv_fact_sales (customer_id);
```

Actualizar vistas materializadas de forma concurrente es sencillo y se puede hacer de forma programada. El siguiente comando se puede invocar desde `psql` y ejecutarse mediante una herramienta de programación, como cron:

```sql
-- refresh the materialized view
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_analytics.mv_fact_sales;
```

Las vistas materializadas son similares a las tablas de bases de datos y funcionan significativamente mejor que las vistas simples, pero ten en cuenta que las vistas materializadas de PostgreSQL deben actualizarse periódicamente.

#### Uso de tablas y disparadores

Esta opción, el uso de tablas y disparadores (*triggers*), nos ayuda a abordar las deficiencias de las vistas materializadas. Utiliza tablas para las dimensiones y los hechos, y disparadores en las tablas base para mantener los datos en las tablas de hechos y dimensiones.

Primero, definimos el esquema en estrella especificando las tablas de dimensiones (fecha, producto y ubicación del cliente) y la tabla de hechos (ventas). Se corresponden exactamente con las definiciones de vistas que creamos anteriormente:

```sql
CREATE TABLE tt_analytics.dim_customer_location (
    customer_id UUID,
    zipcode CHAR(5),
    city VARCHAR(100),
    state_code CHAR(2),
    state_name VARCHAR(50),
    sales_territory VARCHAR(100),
    geographic_region VARCHAR(50),
    country VARCHAR(50)
);
```

Luego definimos funciones de disparador en las tablas base. Observa que hacemos referencia a la variable `NEW`, que es de tipo `RECORD`, y nos permite acceder a los valores del registro nuevo o actualizado en la base de datos:

```sql
CREATE OR REPLACE FUNCTION tt_analytics.sf_insert_customer ()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO tt_analytics.dim_customer_location (
        customer_id,
        zipcode,
        city,
        state_code,
        state_name,
        sales_territory,
        geographic_region,
        country)
    VALUES (
        NEW.id,
        auxiliary.parse_zipcode_postalcode(NEW.postal_code),
        NEW.city,
        auxiliary.parse_state_postalcode(NEW.postal_code),
        (SELECT state_name FROM auxiliary.us_state WHERE state_code = auxiliary.parse_state_postalcode(NEW.postal_code)),
        (SELECT territory_name FROM auxiliary.sales_territory WHERE us_state_code = auxiliary.parse_state_postalcode(NEW.postal_code)),
        (SELECT region FROM auxiliary.us_state WHERE state_code = auxiliary.parse_state_postalcode(NEW.postal_code)),
        NEW.country
    );
    RETURN NEW;
END;
$$ LANGUAGE PLPGSQL;
```

Vinculamos las funciones de disparador a la tabla base como un disparador posterior a la inserción (*after-insert trigger*) que se ejecuta para cada nueva fila:

```sql
CREATE TRIGGER tr_insert_customer
AFTER INSERT ON customer.customer
FOR EACH ROW
EXECUTE FUNCTION tt_analytics.sf_insert_customer();
```

Luego habilitamos los disparadores como disparadores de réplica para asegurarnos de que se activen en eventos de replicación (asumiendo que usamos replicación lógica en lugar de `COPY` para replicar los datos):

```sql
ALTER TABLE customer.customer ENABLE REPLICA TRIGGER tr_insert_customer;
```

De manera similar, definimos funciones y disparadores para actualizaciones y eliminaciones.

Esto también debe hacerse para todas las tablas de componentes. Por ejemplo, cuando se actualiza un registro en la tabla `brand`, cada registro en la dimensión del producto que hace referencia a esa marca también debe actualizarse:

```sql
-- create the triggers and functions to handle updates to the brand table
CREATE OR REPLACE FUNCTION tt_analytics.sf_update_brand ()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE tt_analytics.dim_product
    SET brand = NEW.label
    WHERE product_variant_id IN (SELECT pv.id
                                 FROM product.product_variant pv
                                 JOIN product.product p ON pv.product_id = p.id
                                 WHERE p.brand_id = NEW.id);
    RETURN NEW;
END;
$$ LANGUAGE PLPGSQL;
```

Para nuestro ejemplo, definimos disparadores de inserción/actualización/eliminación para las siguientes entidades: producto, marca, categoría, variante de producto, precio de variante de producto, cliente, transacción de ventas y línea de transacción de ventas. Algunas tablas no requieren disparadores de eliminación (por ejemplo, `brand`) porque son tablas principales en una relación de clave foránea, y eliminar una fila habría causado una infracción de clave foránea en las tablas secundarias. Las funciones `UPSERT` también pueden reducir la cantidad de código personalizado.

En comparación con el enfoque de solo vistas o el método de vistas materializadas, el enfoque de tablas y disparadores requiere más código personalizado para mantener los datos actualizados. El método de disparadores también crea una copia explícita de los datos y, al igual que el enfoque de vistas materializadas, aumenta los requisitos de almacenamiento. Además, ten en cuenta que los disparadores se ejecutan como parte de la transacción que insertó, actualizó o eliminó los datos, y esa transacción principal no se confirmará (*commit*) hasta que se haya completado la función del disparador.

Sin embargo, los datos en el esquema en estrella siempre están actualizados (pueden estar unos segundos por detrás de la base de datos transaccional debido a la latencia de replicación). Además, el enfoque de tablas y disparadores se puede utilizar para escalar la plataforma analítica horizontalmente mediante la replicación lógica (las vistas y las vistas materializadas no se pueden utilizar para definir publicaciones para la replicación lógica; solo las tablas pueden cumplir ese propósito).

---

### Comparación de rendimiento

Los tres enfoques producen el mismo esquema en estrella, con los mismos datos. Para ayudar al desarrollador a tomar una decisión informada, ejecutamos un conjunto de consultas para comparar el rendimiento de los tres enfoques.

Utilizando el punto de referencia `pg_bench` descrito en el [Capítulo 8](https://subscription.packtpub.com/book/data/9781806028474/8), aumentamos el tamaño de los datos a más de 88.000 transacciones de ventas, que comprenden más de 238.000 líneas de ventas. Las vistas materializadas y los esquemas basados en tablas utilizan definiciones de índices idénticas. El almacenamiento total de datos para este punto de referencia (tablas base más tres esquemas) es de aproximadamente 78 GB.

El punto de referencia comparativo incluye una consulta para sumar el valor total de ventas en todas las líneas de ventas, así como una consulta de consolidación (*rollup*) que examina las ventas en el noreste por estado y ciudad en el primer trimestre de 2025:

```sql
SELECT
    state_name,
    COALESCE(city, 'Total'),
    SUM(sales_amount) AS total_sales_amount
FROM tt_analytics.fact_sales
JOIN tt_analytics.dim_customer_location AS cl ON fact_sales.customer_id = cl.customer_id
JOIN tt_analytics.dim_date AS dd ON fact_sales.date = dd.date_key
WHERE year = 2025 AND month IN(1,2,3) AND cl.geographic_region = 'Northeast'
GROUP BY ROLLUP (state_name, city)
HAVING SUM(sales_amount) > 250
ORDER BY state_name, city ASC NULLS LAST;
```

| Esquema en estrella | Tiempo de ejecución (promedio de 5 iteraciones) |
| :--- | :--- |
| Solo vista (*View only*) | 216.48 ms |
| Vista materializada (*Materialized view*) | 60.54 ms |
| Tablas y disparadores (*Tables and triggers*) | 47.90 ms |

*Tabla 12.1: Evaluación comparativa del rendimiento de los tres enfoques de transformación del esquema en estrella*

Como era de esperar, el enfoque de solo vistas es significativamente más lento (aproximadamente 6 veces), mientras que los otros dos enfoques exhiben un rendimiento comparable. Para tu información: una actualización de las cuatro vistas materializadas para el mismo conjunto de datos tardó unos 3 segundos.

Esto demuestra claramente que un enfoque de solo vistas no es adecuado para conjuntos de datos más grandes. El enfoque de vistas materializadas funciona bien, pero debe usarse con precaución, ya que las operaciones de actualización en conjuntos de datos realistas y más grandes pueden requerir mucho tiempo y recursos.

---

### Aprovechamiento del particionamiento

El particionamiento de tablas de PostgreSQL divide una tabla grande en partes físicas más pequeñas. El particionamiento a menudo proporciona beneficios cuando se utiliza en conjuntos de datos grandes que no caben en los búferes compartidos (*shared buffers*) del servidor:

- El rendimiento de las consultas se puede mejorar drásticamente cuando la mayoría de las filas a las que se accede intensamente en la tabla se encuentran en una sola partición o en un número reducido de particiones. El particionamiento sustituye eficazmente los niveles superiores de los índices, lo que hace más probable que las partes de los índices de uso intensivo quepan en la memoria.
- Las cargas masivas y las eliminaciones masivas se pueden lograr agregando o quitando particiones. Desasociar particiones (*detaching partitions*) es mucho más rápido que las operaciones de eliminación masiva y no causa hinchazón de tabla (*table bloat*) (ver [Capítulo 8](https://subscription.packtpub.com/book/data/9781806028474/8), *Comprendiendo las Tablas; Comprendiendo los Índices*), como lo hacen las operaciones de eliminación.
- El proceso de aspirado (*vacuum*) puede centrarse en aquellas particiones que se han actualizado.
- Las particiones de archivo, o particiones raramente utilizadas, se pueden migrar a medios de almacenamiento más rentables.

PostgreSQL admite el particionamiento por lista (*list*), por hash (*hash*) y por rango (*range*):

- **Particiones por lista**: Definen explícitamente qué registros deben aparecer en qué particiones. Por ejemplo, se podría particionar nuestra tabla de hechos de ventas por estado, con una partición para cada estado (Alabama, Alaska, Arizona, etc.).
- **Particiones por hash**: Distribuyen datos a través de un número fijo de particiones según una clave hash. Esta técnica se analizó en el [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), *Adición, Actualización y Manejo de Puntos Calientes de MVCC*.
- **Particiones por rango**: Agrupan datos en rangos que crecen, como números o fechas. Por ejemplo, podríamos agrupar nuestros datos de hechos de ventas por mes o año si esperamos que la mayoría de las consultas se centren en datos de un mes o año.

Para crear nuestra tabla de hechos como una tabla particionada por rango, usamos el comando `PARTITION BY RANGE`, y definimos un conjunto de particiones para los diferentes rangos y una partición `DEFAULT`, en caso de que un registro no coincida con ninguna de las particiones existentes:

```sql
CREATE TABLE part_analytics.fact_sales (
    sales_transaction_line_id UUID,
    date DATE,
    product_variant_id INTEGER,
    customer_id UUID,
    quantity INTEGER,
    unit_price NUMERIC(10,2),
    sales_amount NUMERIC(12,2)
)
-- partition the table by RANGE on the date column
PARTITION BY RANGE (date);

-- create partitions for the years 2024, 2025, 2026
CREATE TABLE part_analytics.fact_sales_2024 PARTITION OF part_analytics.fact_sales
FOR VALUES FROM ('2024-01-01') TO ('2024-12-31');

CREATE TABLE part_analytics.fact_sales_2025 PARTITION OF part_analytics.fact_sales
FOR VALUES FROM ('2025-01-01') TO ('2025-12-31');

CREATE TABLE part_analytics.fact_sales_2026 PARTITION OF part_analytics.fact_sales
FOR VALUES FROM ('2026-01-01') TO ('2026-12-31');

-- create a default partition for future dates
CREATE TABLE part_analytics.fact_sales_default PARTITION OF part_analytics.fact_sales DEFAULT;
```

Las consultas que analizan datos dentro de una partición pueden centrarse en buscar, agrupar y agregar dentro de esa partición, reduciendo la E/S (*I/O*) y acelerando significativamente las consultas. Las búsquedas en múltiples particiones también se pueden paralelizar, lo que potencialmente resulta en un rendimiento mucho mayor.

---

### Orientación para el desarrollador

El desarrollador debe considerar lo siguiente al decidir qué enfoque de transformación utilizar:

- **El tamaño del conjunto de datos**: Pequeño (10 MB – 50 MB), mediano (50 GB – 500 GB) o grande (> 500 GB).
- **El número de usuarios concurrentes**: Pequeño (<10), mediano (<100) o grande (> 100).
- **La complejidad de las consultas**: ¿Se requieren subconsultas, ordenaciones grandes y cálculos complejos sobre grandes conjuntos de datos?
- **Requisito de casi tiempo real (*Near-real-time*)**: ¿Se espera que el sistema analítico esté actualizado (menos de 60 segundos de retraso con respecto al sistema de transacciones), o basta con una actualización programada periódicamente?
- **La necesidad de escalado horizontal**: ¿Puede un solo servidor manejar todas las solicitudes de analítica o es necesario planificar el escalado horizontal?
- **Complejidad de implementación y mantenimiento**: La cantidad de código personalizado que se necesita para mantener sincronizados los datos en el esquema en estrella con el esquema transaccional.

Las agrupaciones pequeño/mediano/grande tienen como objetivo proporcionar un marco conceptual.

| Característica | Solo vistas (*View Only*) | Vistas materializadas (*Materialized Views*) | Tablas y disparadores (*Tables and Triggers*) |
| :--- | :--- | :--- | :--- |
| **Tamaño del conjunto de datos** | Solo conjuntos de datos muy pequeños | Conjunto de datos mediano | Conjunto de datos grande |
| **Número de usuarios concurrentes** | Bajo | Grande | Grande |
| **Complejidad de las consultas** | Baja | Alta | Alta |
| **Requisito de "casi tiempo real"** | Sí | No | Sí |
| **Necesidad de escalado horizontal** | No | No | Sí |
| **Complejidad de implementación y mantenimiento** | Baja | Baja | Alta |
| **Impacto en el rendimiento de la replicación** | Ninguno | Ninguno | Potencialmente, ya que los disparadores se ejecutan como parte del proceso de aplicación de la suscripción. |

*Tabla 12.2: Decidir qué enfoque de transformación usar y cuándo*

Como se muestra en la Tabla 12.2, el enfoque de tablas y disparadores es el más escalable. Aun así, tiene un costo de implementación inicial relativamente alto porque necesitamos crear y mantener una cantidad considerable de disparadores, y aumenta los requisitos de almacenamiento.

Ten en cuenta que esto no tiene que ser una decisión de todo o nada. Algunos de los datos pueden estar disponibles a través de una vista materializada, ya que los usuarios aceptan actualizaciones periódicas. Otras partes del modelo se implementan como vistas, ya que rara vez se utilizan y es aceptable una mayor latencia de consulta. Otras partes que requieren respuestas rápidas se implementan mediante transformaciones basadas en disparadores.

Los tres enfoques también se pueden utilizar si se opta por replicar datos mediante el enfoque basado en copias (*copy-based*) en lugar de la replicación lógica. Sin embargo, en ese caso, los disparadores deben agregarse a las tablas como disparadores regulares en lugar de disparadores de réplica.

---

### Resumen

En este capítulo, repasamos el tercer y último paso del proceso de Extracción, Carga y Transformación (ELT). PostgreSQL proporciona múltiples formas de transformar datos de un modelo normalizado y optimizado para transacciones a un modelo desnormalizado y optimizado para analítica: vistas, vistas materializadas, tablas y disparadores. Si bien las vistas son fáciles de implementar, no escalan a mayores volúmenes de datos o recuentos de usuarios. Según un análisis de rendimiento, recomendamos dos opciones: vistas materializadas o tablas y disparadores. Son el camino a seguir al crear un modelo de datos enfocado en analítica del mundo real. La elección está impulsada por la complejidad de la implementación y la necesidad de información casi en tiempo real.

El siguiente capítulo presenta las capacidades de consulta analítica nativas de PostgreSQL que se basan en el modelo de datos analítico: `GROUPS`, `GROUPING SETS`, `ROLLUPS`, `CUBES`, `WINDOW FUNCTIONS` y `COMMON TABLE EXPRESSIONS` (CTE). Esas son las construcciones SQL que usamos para crear informes y analizar datos.
