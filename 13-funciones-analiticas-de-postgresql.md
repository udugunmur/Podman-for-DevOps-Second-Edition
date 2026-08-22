## Capítulo 13: Funciones Analíticas de PostgreSQL

Postgres proporciona un amplio conjunto de funciones analíticas que ayudan a los usuarios a realizar análisis de datos avanzados directamente dentro de la base de datos. Pueden extraer información rápidamente, crear informes y realizar cálculos avanzados, como totales acumulados (*running totals*), clasificaciones (*rankings*), medias móviles (*moving averages*) y resúmenes complejos.

Idealmente, las funciones analíticas se aplican a un modelo de datos optimizado para analítica, como un esquema en estrella (*star schema*), que se replicó desde uno o más sistemas transaccionales y se desnormalizó para admitir consultas eficientes.

Las funciones analíticas generalmente no modifican los datos, excepto para producir resultados intermedios, y siguen el mismo formato que las consultas `SELECT` simples, pero ofrecen capacidades más potentes de agrupación, agregación, subconsultas y ventanas (*windowing*).

En este capítulo, nos basaremos en el esquema en estrella desarrollado en el [Capítulo 12](https://subscription.packtpub.com/book/data/9781806028474/12), *Transformación de Datos para Analítica*, y te guiaremos paso a paso a través de grupos, agregaciones, funciones de ventana y expresiones de tabla comunes (*common table expressions* o CTE).

El capítulo cubre los siguientes temas:

- Funciones analíticas de PostgreSQL y cómo ayudan con la generación de informes/información
- Agrupación y cálculos agregados (como totales, promedios y recuentos)
- Agrupación avanzada para subtotales: `GROUPING SETS`, `ROLLUP`, `CUBE`
- Funciones de ventana (clasificación, totales acumulados, percentiles y comparaciones)
- CTEs y CTEs recursivas para simplificar consultas complejas y trabajar con jerarquías

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Requisitos técnicos

Todos los ejemplos del capítulo están disponibles en el repositorio de GitHub en `psql_scripts/sample_scripts/chapter_13.sql`. Para acceder al enlace del repositorio, sigue los pasos en la sección "Download the example code files" en el Prefacio.

---

### Funciones analíticas de PostgreSQL

Las funciones analíticas de PostgreSQL nos ayudan a reunir y organizar conjuntos extensos de registros en grupos y grupos jerárquicos, calcular agregados como sumas o promedios, trabajar con ventanas sobre los datos y ejecutar cálculos y recursiones complejos:

- Los **grupos y agregados** nos permiten reunir registros con valores de datos compartidos en una o más columnas (por ejemplo, todas las camisas azules) y calcular agregados (por ejemplo, ventas totales).
- Los **conjuntos de agrupación (*grouping sets*)** facilitan la especificación de múltiples grupos, por ejemplo, todas las camisas, todas las camisas y colores, y todas las camisas y tallas y colores, y calcular los mismos agregados en una sola consulta.
- Los **rollups** toman una lista de grupos, por ejemplo, categoría, color y talla, y generan automáticamente una jerarquía de conjuntos de agrupación y calculan agregados sobre esos conjuntos.
- Los **cubos (*cubes*)** son como rollups, excepto que crean múltiples jerarquías y permutaciones, y no solo una.
- Las **funciones de ventana (*window functions*)** ubican registros individuales en el contexto del conjunto de datos y permiten el cálculo de percentiles, clasificaciones (*rankings*), etc.
- Las **expresiones de tabla comunes (*common table expressions* o CTE)** nos ayudan a simplificar consultas complejas descomponiéndolas en bloques de construcción simples. Las CTE recursivas nos permiten trabajar con conjuntos de datos jerárquicos, como listas de materiales o estructuras organizativas.

Todos nuestros ejemplos analíticos se basan en el esquema en estrella que definimos en el [Capítulo 12](https://subscription.packtpub.com/book/data/9781806028474/12), *Transformación de Datos para Analítica* (ver Figura 13.1). Esto hará que las consultas sean más cortas, ya que no tenemos que repetir todas las uniones (*joins*) en cada consulta, y mucho más rápidas, ya que aprovechamos las dimensiones y los hechos precalculados y sus índices centrados en la analítica.

> [!TIP]
> **Descarga las imágenes a color**  
> Tu compra incluye una copia en PDF a color sin DRM de este libro, ideal para ver imágenes a color, capturas de pantalla y diagramas. Consulta la sección de beneficios gratuitos con tu libro al final del Prefacio para desbloquear tu copia en PDF.

---

### Grupos y agregaciones

Los grupos y los agregados son bloques de construcción analíticos fundamentales. Los conjuntos de filas se definen por valores compartidos en columnas (o expresiones en columnas), y los agregados (`SUM`, `MIN`, `MAX`, etc.) se calculan en función de esos grupos de filas.

Esta consulta de muestra muestra qué marca tiene productos en rangos de precios específicos, cuánto se vendió y el volumen de negocios de ventas total, pero solo para aquellos productos que se originan en los EE. UU. y cuyas ventas totales superan los 10.000. Esta consulta utiliza dos dimensiones (`dim_product` y `dim_date`) para el filtrado y una dimensión (`brand` y `category` de `dim_product`) para la agregación:

```sql
SELECT brand, category, FORMAT ('%2s - %s', DIV(current_price, 10) *10, (DIV(current_price, 10)+1)*10) AS price_range, COUNT(*) nbr_products, ROUND(AVG(current_price),2) AS avg_price, SUM (fact_sales.sales_amount) AS total FROM dim_product NATURAL JOIN fact_sales JOIN dim_date ON fact_sales.date = dim_date.date_key WHERE co_alpha3_code = 'USA' AND year = 2025 GROUP BY brand, category, price_range HAVING SUM(fact_sales.sales_amount) > 10000 ORDER BY brand, category, price_range ASC;
```

Cuando ejecutamos esa consulta en nuestra base de datos de analítica central, obtenemos un informe que aprovecha los hechos en el centro del esquema en estrella y utiliza las dimensiones para el filtrado y la agregación:

```text
      brand       |  category   | price_range | nbr_products | avg_price |   total   
------------------+-------------+-------------+--------------+-----------+-----------
 Calvin Klein     | Accessories | 80 - 90     |          304 |     82.16 |  24976.64
 Diesel           | Pants       | 90 - 100    |         3556 |     98.58 | 350550.48
 Gap              | Pants       | 90 - 100    |         1259 |     98.44 | 123935.96
 Gap              | Polos       | 130 - 140   |         1070 |    131.36 | 140555.20
 Gap              | Shirts      | 30 - 40     |         6013 |     31.72 | 190737.50
 Nike             | Polos       | 130 - 140   |          334 |    131.14 |  43800.76
 Nike             | Sportswear  | 20 - 30     |          814 |     27.44 |  22336.16
 Polo Ralph Laure | Pants       | 90 - 100    |          299 |     98.41 |  29424.59
 Tommy Hilfiger   | Accessories | 80 - 90     |          302 |     82.06 |  24782.12
 Tommy Hilfiger   | Shirts      | 30 - 40     |          579 |     31.72 |  18365.88
 …
```

La cláusula `WHERE` filtra los registros (`alpha3_code = 'USA' AND year = 2025`). La cláusula `HAVING` filtra los grupos. `HAVING SUM (fact_sales.sales_amount) > 10000` filtra todos los grupos que no tienen suficientes ventas totales.

PostgreSQL proporciona varias funciones para agregados aritméticos:

- `AVG`: Devuelve el valor promedio.
- `COUNT`: Devuelve el número de valores.
- `COUNT DISTINCT`: Devuelve el número de valores únicos.
- `MAX`: Devuelve el valor máximo.
- `MIN`: Devuelve el valor mínimo.
- `SUM`: Devuelve la suma de todos los valores o de valores distintos (`SUM (DISTINCT ...)`).

Otras funciones agregadas, como `ARRAY_AGG` o `JSONB_OBJECT_AGG`, compilan valores en matrices o documentos, pero rara vez se utilizan para analítica.

En nuestro ejemplo anterior, utilizamos la cláusula `GROUP BY` con tres columnas individuales que definían una agrupación de los datos: `brand`, `category` y `price_range`.

La cláusula ofrece varios refinamientos que nos permiten crear grupos jerárquicos con subtotales para cada grupo:

- `GROUP BY GROUPING SETS`: Grupos arbitrarios de columnas
- `GROUP BY ROLLUP`: Grupos jerárquicos de columnas
- `GROUP BY CUBE`: Todas las permutaciones de un conjunto de columnas

Los subtotales son una capacidad estándar en los informes empresariales. Antes de que se introdujera esta capacidad en PostgreSQL 9.5, los usuarios tenían que recurrir a consultas complejas o herramientas de terceros para crearlos. Echemos un vistazo a cada uno y mostremos por qué han ayudado a hacer de PostgreSQL una herramienta potente para la analítica.

#### Agrupación por conjuntos de agrupación (GROUP BY GROUPING SETS)

`GROUP BY GROUPING SETS` nos permite crear varios grupos en la misma consulta. Por ejemplo, podemos modificar la consulta anterior para agrupar por marca y rango de precios, por marca y por todos. Para cada agrupación, obtendremos los agregados correspondientes:

```sql
SELECT brand, category, FORMAT ('%2s - %s', DIV(current_price, 10) *10, (DIV(current_price, 10)+1)*10) AS price_range, COUNT(*) nbr_products, ROUND(AVG(current_price),2) as avg_price, SUM (fact_sales.sales_amount) as total FROM dim_product NATURAL JOIN fact_sales WHERE co_alpha3_code = 'USA' GROUP BY GROUPING SETS ((brand, category, price_range), (brand, category), (brand), ()) HAVING SUM(fact_sales.sales_amount) > 5000 ORDER BY brand, category, price_range ASC NULLS LAST;
```

Esta consulta nos proporciona el mismo informe, pero esta vez, con un conjunto jerárquico de subtotales para `brand`, `category` y `price_range`, los grupos definidos en la cláusula `GROUPING SETS`:

```text
    brand     |  category   | price_range | nbr_products | avg_price |   total   
--------------+-------------+-------------+--------------+-----------+-----------
 Calvin Klein | Accessories | 80 - 90     |          304 |     82.16 |  24976.64
 Calvin Klein | Accessories |             |          304 |     82.16 |  24976.64
 Calvin Klein | Shirts      | 30 - 40     |          304 |     31.79 |   9664.16
 Calvin Klein | Shirts      |             |          304 |     31.79 |   9664.16
 Calvin Klein |             |             |          608 |     56.98 |  34640.80
 Diesel       | Pants       | 90 - 100    |         3556 |     98.58 | 350550.48
 Diesel       | Pants       |             |         3556 |     98.58 | 350550.48
 Diese        |             |             |         3556 |     98.58 | 350550.48
 …
```

(En la parte inferior, `…` indica que este es un subconjunto de la salida de la consulta).

La cláusula `NULLS LAST` en `ORDER BY` nos ayuda a asegurarnos de que los subtotales aparezcan como la última fila de esa agrupación.

#### Agrupación por Rollup (GROUP BY ROLLUP)

El rollup simplifica la definición de los conjuntos de agrupación. Cuando definimos un rollup para A, B y C, obtenemos automáticamente los conjuntos de agrupación (A, B, C), (A, B), (A) y (), donde () denota todos los registros:

```sql
SELECT brand, category, FORMAT ('%2s - %s', DIV(current_price, 10) *10, (DIV(current_price, 10)+1)*10) AS price_range, COUNT(*) nbr_products, ROUND(AVG(current_price),2) as avg_price, SUM (fact_sales.sales_amount) as total FROM dim_product NATURAL JOIN fact_sales WHERE co_alpha3_code = 'USA' GROUP BY ROLLUP (brand, category, price_range) HAVING SUM(fact_sales.sales_amount) > 10000 ORDER BY brand, category, price_range ASC NULLS LAST;
```

Esta consulta nos proporciona el mismo informe, también con un conjunto jerárquico de subtotales para `brand`, `category` y `price_range`. Aún así, esta vez, los grupos fueron definidos automáticamente por la cláusula `ROLLUP` en lugar de especificarse manualmente con la cláusula `GROUPING SETS`:

```text
    brand     |  category   | price_range | nbr_products | avg_price |   total   
--------------+-------------+-------------+--------------+-----------+-----------
 Calvin Klein | Accessories | 80 - 90     |          304 |     82.16 |  24976.64
 Calvin Klein | Accessories |             |          304 |     82.16 |  24976.64
 Calvin Klein |             |             |          608 |     56.98 |  34640.80
 Diesel       | Pants       | 90 - 100    |         3556 |     98.58 | 350550.48
 Diesel       | Pants       |             |         3556 |     98.58 | 350550.48
 Diesel       |             |             |         3556 |     98.58 | 350550.48
 Gap          | Pants       | 90 - 100    |         1259 |     98.44 | 123935.96
 Gap          | Pants       |             |         1259 |     98.44 | 123935.96
 Gap          | Polos       | 130 - 140   |         1070 |    131.36 | 140555.20
 Gap          | Polos       |             |         1070 |    131.36 | 140555.20
 Gap          | Shirts      | 30 - 40     |         6013 |     31.72 | 190737.50
 Gap          | Shirts      |             |         6013 |     31.72 | 190737.50
 Gap          |             |             |         8342 |     54.57 | 455228.66
 …
```

(En la parte inferior, `…` indica que este es un subconjunto de la salida de la consulta).

Los rollups se utilizan con mayor frecuencia para agregar datos en relaciones jerárquicas, por ejemplo, por año, trimestre y mes:

```sql
SELECT year, quarter, month, SUM (fact_sales.sales_amount) AS total FROM dim_date JOIN fact_sales ON fact_sales.date = dim_date.date_key WHERE year IN (2025) GROUP BY ROLLUP (year, quarter, month) ORDER BY year, quarter, month ASC NULLS LAST;
```

Esta consulta proporciona un conjunto jerárquico de subtotales para año, trimestre y mes, con la jerarquía de agrupación definida por la cláusula `ROLLUP`:

```text
 year | quarter | month |   total   
------+---------+-------+-----------
 2025 |       1 |     1 | 154197.24
 2025 |       1 |     2 | 121519.18
 2025 |       1 |     3 | 148577.30
 2025 |       1 |       | 424293.72
 2025 |       2 |     4 | 139753.92
 2025 |       2 |     5 | 150402.01
 2025 |       2 |     6 | 136852.22
 2025 |       2 |       | 427008.15
 ...
```

(En la parte inferior, `…` indica que este es un subconjunto de la salida de la consulta).

#### Agrupación por Cubo (GROUP BY CUBE)

Mientras que los rollups nos ayudan a analizar datos en relaciones jerárquicas, los cubos generan todas las permutaciones.

El rollup de (A, B, C) nos da cuatro conjuntos de agrupación:

- (A, B, C)
- (A, B)
- (A)
- ()

Los cubos nos dan las ocho permutaciones posibles:

- (A, B, C)
- (A, B)
- (A, C)
- (B, C)
- (A)
- (B)
- (C)
- ()

```sql
SELECT brand, category, FORMAT ('%2s - %s', DIV(current_price, 10) *10, (DIV(current_price, 10)+1)*10) AS price_range, COUNT(*) nbr_products, ROUND(AVG(current_price),2) as avg_price, SUM (fact_sales.sales_amount) as total FROM dim_product NATURAL JOIN fact_sales WHERE co_alpha3_code = 'USA' GROUP BY CUBE (brand, category, price_range) HAVING SUM(fact_sales.sales_amount) > 10000 ORDER BY brand, category, price_range ASC NULLS LAST;
```

Esto proporciona grupos y agregados adicionales, por ejemplo, para `category` y `price_range`:

```text
      brand       |  category   | price_range | nbr_products | avg_price |   total   
------------------+-------------+-------------+--------------+-----------+-----------
                  | Accessories | 80 - 90     |          666 |     82.11 |  54685.46
                  | Accessories |             |          666 |     82.11 |  54685.46
                  | Pants       | 90 - 100    |         5105 |     98.53 | 503017.02
                  | Pants       |             |         5105 |     98.53 | 503017.02
                  | Polos       | 130 - 140   |         1438 |    131.31 | 188825.06
                  | Polos       |             |         1438 |    131.31 | 188825.06
                  | Shirts      | 30 - 40     |         6922 |     31.72 | 219590.93
                  | Shirts      |             |         6922 |     31.72 | 219590.93
                  | Sportswear  | 20 - 30     |         1069 |     27.42 |  29312.85
                  | Sportswear  |             |         1069 |     27.42 |  29312.85
                  |             | 130 - 140   |         1438 |    131.31 | 188825.06
                  |             | 20 - 30     |         1069 |     27.42 |  29312.85
                  |             | 30 - 40     |         6922 |     31.72 | 219590.93
                  |             | 80 - 90     |          666 |     82.11 |  54685.46
                  |             | 90 - 100    |         5105 |     98.53 | 503017.02
                  |             |             |        15200 |     65.49 | 995431.32
```

(Cerca de la parte superior, `…` indica que este es un subconjunto de la salida de la consulta).

---

### Funciones de ventana

Las funciones de ventana difieren significativamente de las agrupaciones. Una función de ventana realiza un cálculo en un conjunto de filas de la tabla que están relacionadas de alguna manera con la fila actual. Las funciones de ventana no agrupan filas en una sola fila de salida, a diferencia de las funciones agregadas. Ponen una fila en un contexto definido por una ventana sobre algunas o todas las filas de la tabla.

Por ejemplo, si queremos comparar el precio de un producto con el precio promedio de todos los productos de la misma marca, colocamos la fila actual, con el precio del producto actual, en el contexto de todas las filas para esa marca.

En el siguiente ejemplo, `OVER (PARTITION BY brand)` define la ventana de filas (el contexto) con la que comparamos la fila actual; en ese caso, todas las filas que se refieren a la misma marca. Esto nos ayuda a poner el precio de un producto individual en el contexto de los precios promedio para esa marca:

```sql
SELECT DISTINCT brand, label, current_price AS price, ROUND(AVG(current_price) OVER (PARTITION BY brand),2) AS avg_brand FROM dim_product ORDER by brand, label ASC;
```

La salida nos muestra el precio de los productos individuales en comparación con el precio promedio de todos los productos en la ventana definida por la marca del producto:

```text
       brand       |                  label                  | price  | avg_brand 
-------------------+-----------------------------------------+--------+-----------
 Adidas            | Adidas Lifestyle Sneakers               | 131.32 |     79.37
 Adidas            | Adidas Track Pants                      |  27.42 |     79.37
 Aéropostale       | Casual Leather Jacket by Aeropostale    |  81.96 |     81.96
 Boss              | Casual Leather Jacket by Boss           |  82.16 |     90.14
 Boss              | Chinos - casual, by Boss                |  98.59 |     90.14
 Boss              | Dress shirt by Boss                     |  31.93 |     90.14
 Boss              | Polo shirt, Boss                        | 131.35 |     90.14
 Boss              | Sports coat - business casual, by Boss  |  98.57 |     90.14
 Boss              | Trench coat - always perfect, by Boss   |  98.37 |     90.14
 Brioni            | Suit coat - business perfect, by Brioni |  98.43 |     98.43
 …
```

Las funciones de ventana también pueden tener en cuenta el orden de las filas, por ejemplo, para calcular la clasificación (*rank*), los totales acumulados (*running totals*) y los percentiles.

En el siguiente ejemplo, definimos una ventana, `w`, que incluye todas las filas desde el principio hasta la fila actual en orden descendente por ventas de marca. La ventana, `w`, se utiliza luego para calcular el total acumulado (línea 4), el porcentaje acumulado (líneas 5–8) y ordenar los totales de ventas (última línea), mostrando el porcentaje acumulado de las ventas totales (ver Figura 13.2):

```sql
SELECT RANK() OVER (ORDER BY bs.total_brand_sales DESC) AS rank, bs.brand, bs.total_brand_sales AS brand_sales, SUM(bs.total_brand_sales) OVER w AS running_total, ROUND( (SUM(bs.total_brand_sales) OVER w / SUM(bs.total_brand_sales) OVER ()) * 100, 2) AS pct_sales FROM (SELECT d.brand, SUM(f.sales_amount) AS total_brand_sales FROM dim_product d NATURAL JOIN fact_sales f GROUP BY d.brand) AS bs -- define a window 'w' to be used in the calculations -- the window orders by total_brand_sales in descending order -- and includes all rows from the start of the result set to the current row WINDOW w AS (ORDER BY bs.total_brand_sales DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) ORDER BY bs.total_brand_sales DESC;
```

*Figura 13.2: Ventana deslizante de la función de ventana de PostgreSQL*

Esto nos proporciona una vista del porcentaje de las ventas que un producto específico (y todos los anteriores en la lista) constituye del total:

```text
 rank |       brand       | total_brand_sales | running_total | pct_sales 
------+-------------------+-------------------+---------------+-----------
    1 | Boss              |         488713.41 |     488713.41 |     28.07
    2 | Diesel            |         410599.62 |     899313.03 |     51.65
    3 | Gap               |         376663.70 |    1275976.73 |     73.28
    4 | Zara              |          70783.50 |    1346760.23 |     77.34
    5 | Nike              |          64346.92 |    1411107.15 |     81.04
    6 | Brioni            |          58861.14 |    1469968.29 |     84.42
    7 | Adidas            |          49023.24 |    1518991.53 |     87.23
    8 | Tommy Hilfiger    |          44856.10 |    1563847.63 |     89.81
    9 | Calvin Klein      |          32934.02 |    1596781.65 |     91.70
   10 | Polo Ralph Lauren |          32376.89 |    1629158.54 |     93.56
   11 | Uniqlo            |          28461.06 |    1657619.60 |     95.19
   12 | Aéropostale       |          28112.28 |    1685731.88 |     96.81
   13 | Lacoste           |          27504.32 |    1713236.20 |     98.39
   14 | Eaton             |          19848.78 |    1733084.98 |     99.53
   15 | Under Armour      |           8211.00 |    1741295.98 |    100.00
```

Las funciones de ventana son herramientas analíticas convincentes, ya que nos permiten poner las filas en contexto, algo que de otro modo es muy difícil de hacer.

---

### Expresiones de tabla comunes (CTEs)

Las consultas SQL pueden volverse muy complejas, a menudo involucrando múltiples niveles de subconsultas anidadas, que son difíciles de entender y mantener.

Las CTEs ayudan al desarrollador a simplificar consultas complejas descomponiéndolas y reutilizando resultados previos. Son especialmente útiles cuando la misma consulta base (o resultado intermedio) se referencia varias veces dentro de una consulta o función más grande. Las CTEs permiten la definición de consultas base que crean conjuntos de resultados temporales con nombre que fluyen hacia una consulta principal, comparables a vistas que existen solo en el contexto de una consulta.

La consulta mostrada anteriormente se puede reescribir como una CTE, y la subconsulta se puede mover a una consulta base con nombre (`brand_sales`):

```sql
WITH brand_sales AS ( -- the temp table expression to calculate total sales per brand SELECT brand, sum(sales_amount) AS total_brand_sales FROM dim_product dp NATURAL JOIN fact_sales GROUP BY brand ORDER BY total_brand_sales DESC ) -- main query to rank brands by sales and calculate running totals and percentages SELECT RANK() OVER (ORDER BY total_brand_sales DESC) AS rank, brand, total_brand_sales, SUM(total_brand_sales) OVER w AS running_total, ROUND( (SUM(total_brand_sales) OVER w / SUM(total_brand_sales) OVER ()) * 100, 2) AS pct_sales FROM brand_sales WINDOW w AS (ORDER BY total_brand_sales DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) ORDER BY total_brand_sales DESC;
```

¡Esto no solo mejora la legibilidad del código, sino que también nos permite probar las consultas base por separado!

La Tabla 13.1 describe la estructura general de una CTE:

| Componente | Descripción |
| :--- | :--- |
| `WITH` | Crea el contexto para la CTE. |
| `aux1 AS ( SELECT … FROM … WHERE … ),` | Define la primera declaración auxiliar, que da como resultado una tabla temporal, `aux1`, visible solo en el contexto de la CTE. |
| `aux2 AS ( SELECT … FROM … WHERE … ),` | Segunda declaración auxiliar (opcional). |
| `…` | Más declaraciones auxiliares (opcional). |
| `SELECT … FROM aux1, … WHERE …` | Declaración principal que se refiere a las tablas temporales resultantes de las declaraciones auxiliares. |

*Tabla 13.1: Los bloques de construcción de una CTE recursiva*

Las CTEs son herramientas poderosas que los desarrolladores experimentados utilizan con frecuencia. Además de simplificar consultas complejas, también se pueden utilizar para escribir consultas recursivas que iteran a través de listas de materiales o jerarquías organizativas.

Las versiones recientes de PostgreSQL han abordado algunas preocupaciones históricas sobre el rendimiento con las CTEs. Antes de la versión 12, las CTEs a menudo eran problemáticas porque cada consulta auxiliar se planificaba por separado, lo que conducía a un plan general subóptimo para la CTE. Hoy en día, el planificador es lo suficientemente inteligente como para generar planes optimizados para toda la CTE. El desarrollador puede usar la cláusula `MATERIALIZED` para controlar ese comportamiento.

#### CTEs recursivas

El modificador opcional `RECURSIVE` transforma `WITH` de una característica de sintaxis simple a una herramienta que facilita la creación de consultas recursivas. Con `RECURSIVE`, una consulta `WITH` puede hacer referencia a su propia salida. Si bien la recursión también se puede implementar mediante funciones SQL/PL, `WITH RECURSIVE` mantiene la lógica dentro de una sola consulta.

Exactamente como una función matemática recursiva, la CTE recursiva consta de un caso base y un paso recursivo que se basa inicialmente en el caso base y luego en sus propios resultados sucesivos.

Esta es una CTE recursiva muy simple. El caso base añade una fila a la columna `n` en la tabla temporal `sum_to_10`. El paso recursivo añade `n+1` (inicialmente = 2) a la tabla temporal y continúa hasta que `n > 10`. El resultado es un conjunto de registros con 10 filas:

```sql
WITH RECURSIVE sum_to_10 AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n+1 FROM sum_to_10 WHERE n < 10
)
SELECT sum(n) FROM sum_to_10;
```

La Tabla 13.2 muestra los bloques de construcción para una CTE recursiva:

| Componente | Descripción |
| :--- | :--- |
| `WITH RECURSIVE cte_name` | Crea el contexto de la CTE, la declara como recursiva y define su tabla de trabajo temporal (correspondiente al nombre de la CTE). |
| `AS ( SELECT columns FROM table1 WHERE condition` | Término base no recursivo evaluado primero. El resultado se coloca en la tabla de trabajo. |
| `UNION [ALL]` | Incluye todas las filas restantes en el resultado de la consulta recursiva y las añade a la tabla de trabajo. `UNION` elimina duplicados; `UNION ALL` mantiene duplicados. |
| `SELECT columns FROM cte_name WHERE recursive_condition )` | Término recursivo. La condición recursiva y los cálculos para las columnas pueden hacer referencia a la tabla de trabajo. |
| `SELECT columns FROM cte_name;` | Declaración principal ejecutada contra la tabla de trabajo temporal. |

*Tabla 13.2: Los bloques de construcción de una CTE recursiva*

La siguiente consulta ilustra las CTEs recursivas. Utiliza la estructura organizativa jerárquica definida en la tabla `auxiliary.sales_organization` (ver también Figura 13.3) para calcular el nivel organizativo de cada empleado.

*Figura 13.3: Muestra de la organización de ventas definida en la tabla auxiliary.sales_organization*

La definición de una CTE recursiva comienza con el término base no recursivo. En nuestro ejemplo, el término base identifica a los directores superiores (aquellos sin un gerente) y los agrega a la tabla de trabajo temporal llamada `sales_hierarchy`.

El término recursivo se une a la tabla temporal para identificar a todos los empleados que reportan a gerentes que ya están en esa tabla. Se agregan nuevos registros a la tabla temporal y el proceso continúa hasta que el término recursivo no devuelve más registros nuevos:

```sql
WITH RECURSIVE sales_hierarchy AS ( -- identify top managers (those without a manager) SELECT employee_id, first_name, last_name, territory_id, manager_id, 1 AS level FROM auxiliary.sales_organization WHERE manager_id IS NULL UNION ALL -- recursively join to find employees under each manager SELECT so.employee_id, so.first_name, so.last_name, so.territory_id, so.manager_id, sh.level + 1 AS level FROM auxiliary.sales_organization so JOIN sales_hierarchy sh ON so.manager_id = sh.employee_id ) SELECT employee_id, first_name, last_name, manager_id FROM sales_hierarchy ORDER BY level, employee_id;
```

La salida muestra cómo la consulta atraviesa la jerarquía. La primera fila (empleado E001) es el resultado del término base no recursivo. Las siguientes cuatro filas son las recursiones iniciales basadas en el término base:

```text
 employee_id | first_name | last_name | sales_target | manager_id 
-------------+------------+-----------+--------------+------------
 E001        | John       | Smith     |            0 | 
 E002        | Jane       | Doe       |            0 | E001
 E003        | Jim        | Brown     |            0 | E001
 E004        | Emily      | Davis     |            0 | E001
 E005        | Michael    | Wilson    |            0 | E001
 E006        | Sarah      | Johnson   |            0 | E002
 E007        | David      | Lee       |       150007 | E002
 E008        | Laura      | Garcia    |       150008 | E003
 E009        | Robert     | Martinez  |       150009 | E003
 E010        | Linda      | Rodriguez |       150010 | E004
 E011        | James      | Hernandez |       150011 | E004
 E012        | Barbara    | Lopez     |       150012 | E005
 E013        | William    | Gonzalez  |       150013 | E005
 E014        | Elizabeth  | Wilson    |       150014 | E006
 E015        | Richard    | Anderson  |       150015 | E006
```

Ten en cuenta que la parte recursiva se refiere a registros que se agregaron previamente a la tabla temporal (en este ejemplo, el término recursivo se une con la tabla temporal e incrementa el valor de `sh.level` en 1).

En el ejemplo anterior, atravesamos la estructura de arriba hacia abajo (*top down*). Algunas preguntas, como calcular el objetivo de ventas acumulativo del gerente, requieren un enfoque de abajo hacia arriba (*bottom-up*). Los objetivos de ventas en los niveles inferiores se agregan primero en una tabla temporal antes de la agregación final:

```sql
WITH RECURSIVE bottom_up AS ( -- leaves of the hierarchy (those without subordinates) SELECT employee_id, first_name, last_name, manager_id, sales_target FROM auxiliary.sales_organization WHERE NOT EXISTS (SELECT 1 FROM auxiliary.sales_organization so WHERE so.manager_id = auxiliary.sales_organization.employee_id) UNION ALL -- aggregate sales targets up the hierarchy SELECT so.employee_id, so.first_name, so.last_name, so.manager_id, so.sales_target + bu.sales_target FROM auxiliary.sales_organization so JOIN bottom_up bu ON so.employee_id = bu.manager_id ) SELECT DISTINCT employee_id, first_name, last_name, manager_id, sum(sales_target) OVER (PARTITION BY employee_id) AS total_sales_target FROM bottom_up ORDER BY total_sales_target DESC;
```

En este caso, el término base no recursivo identificó a ocho empleados sin subordinados directos (E007, E008, …, E015), y luego la recursión se acumuló a partir de ahí:

```text
 employee_id | first_name | last_name | manager_id | total_target 
-------------+------------+-----------+------------+--------------
 E001        | John       | Smith     |            |      1350099
 E002        | Jane       | Doe       | E001       |       450036
 E006        | Sarah      | Johnson   | E002       |       300029
 E005        | Michael    | Wilson    | E001       |       300025
 E004        | Emily      | Davis     | E001       |       300021
 E003        | Jim        | Brown     | E001       |       300017
 E015        | Richard    | Anderson  | E006       |       150015
 E014        | Elizabeth  | Wilson    | E006       |       150014
 E013        | William    | Gonzalez  | E005       |       150013
 E012        | Barbara    | Lopez     | E005       |       150012
 E011        | James      | Hernandez | E004       |       150011
 E010        | Linda      | Rodriguez | E004       |       150010
 E009        | Robert     | Martinez  | E003       |       150009
 E008        | Laura      | Garcia    | E003       |       150008
 E007        | David      | Lee       | E002       |       150007
```

Las CTEs ayudan a los desarrolladores a simplificar consultas complejas dividiéndolas en declaraciones auxiliares que se pueden desarrollar y probar individualmente. ¡Esa característica por sí sola ya es muy útil!

Lo que hace que las CTEs sean verdaderamente emocionantes es que pueden ayudarnos a navegar por estructuras jerárquicas (y de grafos), algo que antes requería una codificación compleja. En PostgreSQL 14, se agregó la cláusula `SEARCH` para dar a los desarrolladores control sobre la búsqueda en profundidad (*depth-first*) frente a la búsqueda en anchura (*breadth-first*), y `CYCLE` permite la detección de ciclos cuando los grafos no son estrictamente jerárquicos.

La cláusula `SEARCH` se puede utilizar para cambiar el recorrido del árbol del valor predeterminado en anchura a en profundidad. Por ejemplo, la adición de `SEARCH DEPTH FIRST BY employee_id SET ordercol` cambia la secuencia en la que se visitan los nodos y nos da una nueva columna `ordercol` para rastrear el recorrido:

```sql
WITH RECURSIVE hierarchy AS (
    …
)
SEARCH DEPTH FIRST BY employee_id SET ordercol
SELECT employee_id, first_name, last_name, manager_id, ordercol
FROM hierarchy
ORDER BY employee_id;
```

Esto cambia el recorrido de la jerarquía a una búsqueda en profundidad (subconjunto de la salida):

```text
 employee_id | first_name | last_name | manager_id |        ordercol        
-------------+------------+-----------+------------+------------------------
 E001        | John       | Smith     |            | {(E001)}
 E002        | Jane       | Doe       | E001       | {(E001),(E002)}
 E003        | Jim        | Brown     | E001       | {(E001),(E003)}
 E004        | Emily      | Davis     | E001       | {(E001),(E004)}
 E005        | Michael    | Wilson    | E001       | {(E001),(E005)}
 E006        | Sarah      | Johnson   | E002       | {(E001),(E002),(E006)}
 ...
```

Un ejemplo simple ilustra el uso de la cláusula `CYCLE`. Agregar `CYCLE employee_id SET is_cycle USING path` garantiza que se detecten los ciclos:

```sql
-- sample table for a cyclic data structure
CREATE TABLE test_cycle (
    employee_id TEXT PRIMARY KEY,
    manager_id TEXT
);

INSERT INTO test_cycle VALUES ('E001', 'E002');
INSERT INTO test_cycle VALUES ('E002', 'E003');
INSERT INTO test_cycle VALUES ('E003', 'E001');

-- CTE to detect cycles in the hierarchy using the CYCLE clause
WITH RECURSIVE hierarchy AS (
    SELECT employee_id, manager_id
    FROM test_cycle
    --WHERE manager_id IS NULL
    UNION ALL
    SELECT tc.employee_id, tc.manager_id
    FROM test_cycle tc
    JOIN hierarchy h ON tc.manager_id = h.employee_id
)
CYCLE employee_id SET is_cycle USING path
SELECT employee_id, manager_id, is_cycle, path
FROM hierarchy
ORDER BY path;
```

Esto nos da dos columnas adicionales, `is_cycle` y `path`, que podemos usar para obtener visibilidad sobre el recorrido:

```text
 employee_id | manager_id | is_cycle |             path              
-------------+------------+----------+-------------------------------
 E001        | E002       | f        | {(E001)}
 E003        | E001       | f        | {(E001),(E003)}
 E002        | E003       | f        | {(E001),(E003),(E002)}
 E001        | E002       | t        | {(E001),(E003),(E002),(E001)}
 E002        | E003       | f        | {(E002)}
 E001        | E002       | f        | {(E002),(E001)}
 E003        | E001       | f        | {(E002),(E001),(E003)}
 E002        | E003       | t        | {(E002),(E001),(E003),(E002)}
 E003        | E001       | f        | {(E003)}
 E002        | E003       | f        | {(E003),(E002)}
 E001        | E002       | f        | {(E003),(E002),(E001)}
 E003        | E001       | t        | {(E003),(E002),(E001),(E003)}
```

---

### Resumen

Grupos, agregaciones, funciones de ventana, expresiones de tabla comunes y recursiones: PostgreSQL nos proporciona una gran cantidad de capacidades para analizar datos. Todas ellas aprovechan esquemas en estrella bien diseñados con sus hechos y dimensiones para facilitar la redacción de consultas altamente efectivas.

Si bien un esquema bien diseñado centrado en la analítica hace que el análisis de datos sea más fácil y eficiente, las técnicas y capacidades que revisamos aquí también se pueden aplicar directamente a esquemas normalizados. Esto significará que tendremos declaraciones de unión (*join*) más complejas, pero los grupos, agregados, CTEs, funciones de ventana y recursiones también tienen su lugar en el mundo transaccional.

En el siguiente capítulo, ampliaremos nuestro enfoque de datos tabulares a datos de texto. Aprenderemos a indexar, buscar y consultar datos en columnas de datos centradas en texto.
