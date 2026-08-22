## Capítulo 14: Análisis de Documentos de Texto en PostgreSQL

Los textos y documentos se han convertido en una parte cada vez más importante de lo que se almacena y administra en las bases de datos. Históricamente, los desarrolladores recurrían a herramientas especializadas de búsqueda de texto para extraer valor de esos datos textuales. Sin embargo, las crecientes capacidades polimórficas (o multimodelo) de PostgreSQL significan que los desarrolladores no solo pueden almacenar documentos en PostgreSQL, sino también analizarlos sin recurrir a herramientas de terceros.

En este capítulo, revisaremos las capacidades de búsqueda de texto en PostgreSQL:

- Búsqueda simple basada en patrones usando `LIKE`
- Búsqueda de patrones complejos usando expresiones regulares
- Búsqueda de texto completo (*Full-Text Search* o FTS) usando `tsvector` y `tsquery`
- Búsqueda difusa (*fuzzy search*) usando `pg_trgm`

Cada una tiene diferentes fortalezas y debilidades, lo que las convierte en la herramienta de elección para casos de uso específicos. Proporcionaremos orientación sobre cuándo utilizar cada capacidad.

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Requisitos técnicos

El código de muestra para este capítulo está disponible en `psql_scripts/sample_scripts/chapter_14.sql`. ¡Te animamos a conectarte a la base de datos `central_analytics` y seguir los ejemplos! Para acceder al enlace del repositorio, sigue los pasos en la sección "Download the example code files" en el Prefacio.

---

### El texto como un tipo de datos clave en la analítica avanzada

Históricamente, el almacenamiento de texto en bases de datos relacionales se limitaba a cadenas de caracteres cortas, como nombres de personas, lugares o productos. Los textos más largos, como descripciones detalladas de productos, informes de reuniones y protocolos de exámenes clínicos, se almacenaban tradicionalmente en bases de datos especializadas centradas en documentos o en unidades de red. La impresión general era que las bases de datos relacionales no podían manejar el volumen de datos y carecían de las capacidades necesarias de búsqueda de texto.

Los desarrollos recientes cambiaron eso:

- Las bases de datos relacionales, incluida PostgreSQL, se volvieron capaces de manejar conjuntos de datos mucho más grandes. La partición, una E/S más rápida, dispositivos de almacenamiento mejorados, mejor compresión de datos y métodos mejorados de copia de seguridad/recuperación fueron impulsores clave para ese desarrollo. Hace 10 años, una base de datos PostgreSQL con 1 terabyte (TB) de datos se consideraba grande; ¡hoy en día, los usuarios informan de tamaños de datos de más de 100 TB en una sola base de datos!
- Integrar el análisis de datos "más blandos", por ejemplo, documentos de texto completo, se volvió cada vez más importante para el negocio. Encontrar nombres de clientes, nombres de productos, ubicaciones, diagnósticos médicos y otros puntos de datos en documentos de texto dentro de una sola consulta de base de datos ahorra tiempo y reduce la complejidad operativa.
- A las bases de datos y aplicaciones acceden cada vez más usuarios no capacitados o clientes que desean acceder a los datos "a su manera". Por ejemplo, cuando buscan una tienda en una ciudad específica, no quieren verse obstaculizados por problemas de ortografía. Cuando buscan una dirección en "San Frncisco", esperan que la base de datos se encargue del error tipográfico.

---

### Búsqueda de texto en PostgreSQL

PostgreSQL proporciona varias formas de abordar estos puntos:

- Coincidencia de patrones basada en `LIKE` para aplicaciones simples y de bajo volumen
- Coincidencia de expresiones regulares POSIX utilizando el operador `~`
- Búsqueda de texto completo (*Full-Text Search* o FTS) basada en vectores que admite derivación de palabras (*stemming*), patrones complejos, clasificación (*ranking*) y resaltado (*headlining*)
- Búsqueda por trigramas para coincidencia difusa de patrones basada en similitud

Revisaremos y explicaremos cada una y concluiremos con una guía para el desarrollador.

#### Coincidencia básica de patrones con LIKE

El operador `LIKE` en SQL es el operador de búsqueda de texto más utilizado y más fácil de implementar. ¡Pero también es el más lento y el menos capaz!

Este es un ejemplo simple del operador `LIKE`:

```sql
SELECT DISTINCT p.id, p.label AS product, b.label AS brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE longdescription LIKE '%shirt%';
```

El operador `LIKE` coincide con expresiones simples, donde `%` coincide con múltiples caracteres y `_` coincide con un solo carácter:

```sql
SELECT DISTINCT p.id, p.label AS product, b.label AS brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.label LIKE '_olo%';
```

También se admiten patrones más complejos, como este que coincide con cualquier cadena que incluya `Oxford` antes de `Shirt`:

```sql
SELECT DISTINCT p.id, p.label AS product, b.label AS brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.label LIKE '%Oxford%Shirt%';
```

El operador `LIKE` tiene una variante, `ILIKE`, que no distingue entre mayúsculas y minúsculas (*case-insensitive*). `LIKE` también tiene otras capacidades, como la función `casefold()` introducida en PostgreSQL 18, que proporciona una forma global y compatible con Unicode de trabajar con cadenas independientemente de mayúsculas/minúsculas. El operador `LIKE` también puede aprovechar GIN, pero solo para términos de búsqueda alineados a la izquierda, es decir, términos de búsqueda que aparecen al principio de un texto; por ejemplo, `Shirt%`, que coincidiría con `Shirts for business` pero no con `White Shirts for business`.

`LIKE` alcanza rápidamente sus límites. La siguiente consulta proporciona los resultados esperados, ya que el patrón coincide con la secuencia de las palabras en la columna de destino:

```sql
SELECT DISTINCT p.id, p.label, b.label FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.label ILIKE '%Men%Oxford%shirt%';
```

La siguiente, sin embargo, no lo hace, ya que la secuencia de palabras en el patrón de búsqueda no coincide, aunque todas las palabras individuales estén presentes:

```sql
SELECT DISTINCT p.id, p.label, b.label FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.label ILIKE '%Oxford%shirt%Men';
```

`LIKE`/`ILIKE` es fácil de usar, pero sus capacidades son limitadas. Debido a que utiliza índices solo en casos excepcionales, no escala.

#### Coincidencia de expresiones regulares

PostgreSQL admite la coincidencia de expresiones regulares POSIX utilizando el operador `~`:

- `~` se utiliza para la coincidencia de expresiones regulares que distingue entre mayúsculas y minúsculas (*case-sensitive*).
- `~*` es como `~`, pero no distingue entre mayúsculas y minúsculas (*case-insensitive*).
- `!~` es verdadero si no hay una coincidencia que distinga entre mayúsculas y minúsculas.
- `!~*` es como `!~`, pero no distingue entre mayúsculas y minúsculas.

Estas expresiones son mucho más potentes que las que se pueden utilizar en la expresión `LIKE` y le dan al desarrollador acceso completo a las expresiones regulares. Las expresiones simples se parecen mucho a la declaración `LIKE`, excepto que el comodín `%` ha sido reemplazado por el comodín de expresiones regulares `.*`:

```sql
SELECT DISTINCT p.id, p.label, b.label FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.label ~* 'Men.*Oxford.*shirt';
```

Además del operador `~`, las expresiones regulares de PostgreSQL también proporcionan una gran cantidad de otras funciones, como `regexp_count`, `regexp_instr`, `regexp_like` y `regexp_match`. Los desarrolladores que se sienten cómodos con regex tienen acceso a una amplia gama de coincidencias de patrones, por ejemplo, la capacidad de hacer coincidir una descripción que comience con `Chinos` y termine con `Boss` o `The Gap`:

```sql
SELECT DISTINCT p.id, p.label, b.label FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.label ~* '^Chinos .* (Boss|The Gap)$';
```

Las expresiones regulares en PostgreSQL no aprovechan los índices generales, excepto los índices funcionales que se centran en expresiones específicas, como URLs, o los índices B-tree para hacer coincidir expresiones ancladas al principio de la cadena. Las expresiones regulares pueden ser lentas y a menudo no escalan para un uso general, pero se utilizan con frecuencia para la limpieza de datos (*data cleansing*).

#### Búsqueda de texto completo con tsvector y tsquery

Tanto `LIKE` como las expresiones regulares POSIX utilizan un enfoque de coincidencia de patrones simple, que es el núcleo de sus limitaciones. Afortunadamente, PostgreSQL FTS, que utiliza un modelo vectorizado, es mucho más capaz y escalable. FTS transforma los datos de texto en vectores de búsqueda de texto (`tsvector`), un formato óptimo para búsquedas complejas que se pueden consultar de manera eficaz y eficiente con `tsquery`, un formato de consulta especializado.

##### Uso de tsvector para convertir los datos de origen

En PostgreSQL, usamos la función `to_tsvector` para transformar un texto en un vector de búsqueda de texto (`tsvector`). `to_tsvector` pasa por dos pasos: derivación (*stemming*) y vectorización (*vectorization*):

- **Derivación (*Stemming*)**: Los textos se transforman en una lista de raíces de palabras (también conocidos como lexemas) que se normalizan para representar diferentes variantes de la misma palabra.
- **Vectorización (*Vectorization*)**: Los textos se representan como vectores de raíces de palabras y sus posiciones.

Por ejemplo, consideremos esta consulta:

```sql
SELECT to_tsvector( 'english', 'A shirt is not like a T-shirt, even though they both use the string shirt');
```

Produce un vector alfabético de raíces de palabras y sus posiciones en el texto de entrada:

```text
to_tsvector -------------------------------------------------------------------------------- 'even':10 'like':5 'shirt':2,9,17 'string':16 't-shirt':7 'though':11 'use':14
```

Esta representación simplificada facilita la coincidencia de patrones complejos, por ejemplo, secuencias de palabras o simplemente ocurrencias de palabras, y consultas más avanzadas, como `shirt`, pero no `T-shirt`.

La función `to_tsvector` utiliza diccionarios específicos del idioma para identificar palabras y extraer sus raíces (lexemas). Se emplean otros diccionarios para reconocer palabras vacías (*stop words*), como `the`, `and` y `or`, que se ignoran al construir el vector. Los usuarios también pueden agregar diccionarios personalizados para áreas especializadas con el fin de manejar sinónimos y abreviaturas que deberían asignarse al mismo lexema.

Los vectores de búsqueda de texto se almacenan en columnas separadas y se pueden indexar mediante GIN o GiST:

- Los **índices GIN** generalmente se prefieren porque tienen un mayor rendimiento de búsqueda, ya que son índices invertidos que asignan lexemas a los documentos donde aparecen. Sin embargo, los índices GIN son más lentos de actualizar que los índices GiST.
- Los **índices GiST** se utilizan para datos dinámicos y que cambian rápidamente donde los requisitos de rendimiento de escritura superan los requisitos de rendimiento de lectura.

Para una revisión en profundidad de los índices, consulta el [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), *Diseño para Altos Volúmenes de Transacciones y Escritura de Código Transaccional Eficiente*.

##### Uso de tsquery para crear consultas

Las consultas de búsqueda de texto se especifican como lexemas y operadores. Por ejemplo, para especificar una búsqueda de camisas de The Gap, la función `to_tsquery` crea un patrón de consulta simple a partir de un conjunto de lexemas:

```sql
SELECT to_tsquery('english', 'Oxford&Shirts&Gap');
```

Convierte las palabras `Oxford`, `Shirts` y `Gap` en sus lexemas. Observa el paso a minúsculas y la omisión de la `s` del plural:

```text
to_tsquery ---------------------------- 'oxford' & 'shirt' & 'gap'
```

`to_tsquery` es la función básica para generar consultas de búsqueda de texto y se puede usar con raíces de palabras y operadores para crear consultas complejas AND (`&`), OR (`|`) y FOLLOWED BY (`<->`).

`phraseto_tsquery` es más flexible y genera la consulta de búsqueda de texto con operadores basados en la secuencia en una frase:

```sql
SELECT phraseto_tsquery('english', 'Oxford Shirt from the Gap');
```

Esto convierte las palabras en lexemas, pero también agrega información de secuencia:

```text
phraseto_tsquery -------------------------------- 'oxford' <-> 'shirt' <3> 'gap'
```

Esto significa que `shirt` debe seguir a `oxford` y que `gap` debe ser el tercer lexema después de `shirt`.

`websearch_to_tsquery` es aún más flexible y potente. Reconoce palabras clave específicas como operadores para construir consultas aún más potentes:

```sql
SELECT websearch_to_tsquery( 'english', 'T-Shirt from the Gap or Oxford Shirt from the gap');
```

Deriva automáticamente secuencias y alternativas y se utiliza a menudo para convertir búsquedas web en consultas SQL:

```text
websearch_to_tsquery ------------------------------------------------------------ 't-shirt' <2> 'shirt' & 'gap' | 'oxford' & 'shirt' & 'gap'
```

Para satisfacer esta consulta, la palabra `t-shirt` debe preceder a la combinación de las palabras `shirt` y `gap`, o las tres palabras `oxford`, `shirt` y `gap` deben estar en el texto.

##### Poniéndolo todo junto

Como se describió anteriormente, usamos `to_tsvector` o una de sus variantes para generar la representación vectorizada que comparamos con la salida de `to_tsquery`. Generar la representación vectorizada del texto cada vez que ejecutamos una consulta sería terriblemente ineficiente.

En su lugar, agregamos una columna generada y almacenada a nuestra tabla, la completamos con datos de vector de búsqueda de texto, la indexamos y ejecutamos nuestras búsquedas contra esa columna.

En nuestro ejemplo, queremos buscar la información textual asociada a nuestros productos: la etiqueta, la descripción corta y la descripción larga. Para ese propósito, agregamos la columna `infotext_tsv` de tipo `tsvector` a la tabla `product`.

Esta columna almacena la información del vector de búsqueda de texto resultante de la combinación de tres campos de texto: `label`, `shortdescription` y `longdescription`. La columna es una columna generada, lo que garantiza que se actualice continuamente, y es una columna almacenada, lo que significa que podemos indexarla:

```sql
ALTER TABLE product ADD COLUMN infotext_tsv tsvector GENERATED ALWAYS AS ( to_tsvector('english', -- cannot use format here as it's not immutable coalesce(label,'') || ' ' || coalesce(shortdescription,'') || ' ' || coalesce(longdescription,'') ) ) STORED;
```

Agregamos un índice GIN a esa columna para mejorar la eficiencia de la consulta:

```sql
-- create an index on the tsvector column CREATE INDEX idx_product_infotext_tsv ON product USING GIN (infotext_tsv);
```

Ahora podemos ejecutar nuestras primeras consultas FTS:

```sql
SELECT DISTINCT p.id, p.label product, b.label brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.infotext_tsv @@ plainto_tsquery('english', 'An Oxford shirt from the Gap');
```

El FTS encontró automáticamente entradas coincidentes en la tabla `product` utilizando el patrón de búsqueda:

```text
id | product | brand ----+--------------------------------------+------- 4 | Mens Classic Oxford Shirt by The Gap | Gap
```

Cuando usamos `websearch_to_tsquery`, podemos ejecutar consultas mucho más complejas:

```sql
SELECT DISTINCT p.id, p.label product, b.label brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE p.infotext_tsv @@ websearch_to_tsquery( 'english', 'T-Shirt from the Gap or Oxford Shirt from the gap');
```

En este caso, se agregó una condición OR (ver arriba), y FTS encontró dos entradas coincidentes:

```text
id | product | brand ----+--------------------------------------+------- 4 | Mens Classic Oxford Shirt by The Gap | Gap 6 | T-Shirt by The Gap | Gap
```

*Figura 14.1: La interacción de los bloques de construcción de PostgreSQL FTS*

##### Ponderación de la entrada (*Weighting the input*)

En nuestro ejemplo, combinamos la etiqueta, la descripción corta y la descripción larga de nuestros productos en un solo vector de texto para que sirva como base para el FTS. ¿Qué pasa si decidimos que la información de la etiqueta es más pertinente que la de las descripciones corta y larga, y que debería tener un mayor peso?

La vectorización de búsqueda de texto nos permite hacer eso ensamblando un vector a partir de varios vectores componentes, cada uno con un peso diferente.

En el siguiente ejemplo, establecemos que la información en la columna `label` es más crítica que la descripción corta, que a su vez es más importante que la descripción larga:

```sql
ALTER TABLE product ADD COLUMN infotext_tsv tsvector GENERATED ALWAYS AS ( setweight (to_tsvector('english', COALESCE(label, '')), 'A') || setweight (to_tsvector('english', COALESCE(shortdescription, '')), 'B') || setweight (to_tsvector('english', COALESCE(longdescription, '')), 'C' ) ) STORED;
```

Los pesos son útiles cuando se trabaja con textos complejos. Por ejemplo, al buscar en blogs o artículos, podríamos dar mayor peso al título, el subtítulo y el resumen que al cuerpo, ya que es más probable que el cuerpo coincida con muchos lexemas. Por el contrario, es más probable que el título, el subtítulo y el resumen se centren en elementos clave.

##### Clasificación de los resultados de búsqueda (*Ranking the search results*)

Los ejemplos anteriores utilizaron FTS en modo binario: una fila está en el conjunto de resultados o no lo está. PostgreSQL FTS también nos permite clasificar los resultados. En el siguiente bloque de código, estamos usando la misma consulta, pero en lugar de usarla en la cláusula `WHERE`, usamos `ts_rank` para clasificar los resultados:

```sql
SELECT DISTINCT p.id, p.label as product_label, b.label as brand_label, ts_rank(p.infotext_tsv, websearch_to_tsquery( 'english', 'T-Shirt from the Gap or Oxford Shirt from the gap')) AS rank FROM product p JOIN brand b ON p.brand_id = b.id ORDER BY rank DESC LIMIT 5;
```

En lugar de simplemente darnos dos resultados (clasificados por encima de una coincidencia de 0.5), obtenemos una lista más larga de coincidencias en orden decreciente:

```text
id | product_label | brand_label | rank ----+--------------------------------------+-------------+------------ 6 | T-Shirt by The Gap | Gap | 0.51621455 4 | Mens Classic Oxford Shirt by The Gap | Gap | 0.51537025 7 | T-Shirt by Diesel | Diesel | 0.34903458 5 | Mens Classic Oxford Shirt from Boss | Boss | 0.34819028 1 | Dress shirt by The Gap | Gap | 0.3431678
```

La clasificación nos ayuda a hacer más que simples búsquedas binarias de verdadero/falso y agrega una flexibilidad increíble a FTS.

##### Resaltado (*Headlining*)

A veces, los usuarios quieren entender por qué se seleccionó un documento de texto. El resaltado (*headlining*) destaca el texto que coincide con una consulta:

```sql
SELECT DISTINCT p.id, p.label as product_label, b.label as brand_label, ts_rank(p.infotext_tsv, websearch_to_tsquery( 'english', 'T-Shirt from the Gap or Oxford Shirt from the gap')) AS rank, ts_headline(p.label || ' '|| p.shortdescription|| ' '|| p.longdescription, websearch_to_tsquery( 'english', 'T-Shirt from the Gap or Oxford Shirt from the gap')) AS snippet FROM product p JOIN brand b ON p.brand_id = b.id ORDER BY rank DESC LIMIT 3;
```

El comando `ts_headline` extrae fragmentos de texto relevantes del texto de origen y resalta todas las palabras que aparecen en la consulta:

```text
id | product_label | brand_label | rank| snippet ----+-----------------+-------------+-----+------------------- 6 | T-Shirt by Th...| Gap |0.51 | T-<b>Shirt</b> by The <b>Gap</b> Short | | | sleeved fitted T-<b>shirt</b> from | <b>Gap</b> 4 | Mens Classic ...| Gap |0.51 | <b>Oxford</b> <b>Shirt</b> by The | <b>Gap</b> Timeless mens <b>Oxford</b> | <b>shirt</b> in 100% cotton 7 | T-Shirt by D... | Diesel |0.34 | T-<b>Shirt</b> by Diesel Short sleeved | fitted T-<b>shirt</b> by Diesel Mens T (3 rows)
```

Ten en cuenta que la salida se ha formateado y abreviado manualmente para facilitar su lectura.

`<b></b>` es el formato predeterminado para los resaltados. Este valor predeterminado, al igual que la longitud y los detalles de los titulares, es totalmente configurable.

La función de resaltado, en combinación con la ponderación y la clasificación, facilita el trabajo con documentos grandes y ayuda a explicar por qué se incluyó un documento en los resultados.

##### Uso de PostgreSQL FTS en combinación con consultas normales

La verdadera fuerza del polimorfismo de PostgreSQL aparece cuando combinamos FTS con consultas convencionales. Ahora obtenemos lo mejor de ambos mundos: un motor FTS flexible con capacidades multilingües, derivación, coincidencia de patrones y vectorización, combinado con la potencia de SQL. La siguiente consulta muestra cómo combinar una consulta de texto (`edgy and cool shirt or t-shirt`) con una condición SQL estándar (`price < 50`):

```sql
SELECT p.id, p.label, FORMAT ('%s-%s', MIN(pvp.price), MAX (pvp.price)) AS price_range FROM product p JOIN product_variant pv ON p.id = pv.product_id JOIN product_variant_price pvp ON pv.id = pvp.product_variant_id WHERE p.infotext_tsv @@ websearch_to_tsquery( 'english', 'edgy and cool shirt or t-shirt') AND pvp.price < 50 GROUP BY p.id, p.label ORDER BY MAX( pvp.price) desc;
```

Los resultados se filtran por ambas condiciones: la búsqueda de texto para la camiseta moderna y genial (*edgy and cool*), y el precio de menos de $50:

```text
id | label | price_range ----+-----------------------+------------- 7 | T-Shirt by Diesel | 29.05-32.69 6 | T-Shirt by The Gap | 29.00-32.64 23 | Nike Athletic T-Shirt | 25.12-28.27
```

##### Resumen de FTS

PostgreSQL FTS es una herramienta poderosa, especialmente cuando se usa junto con consultas SQL estándar.

Los enfoques convencionales podrían haber utilizado dos soluciones diferentes, una para la búsqueda de texto y otra para la consulta SQL, combinadas con código personalizado para fusionar los resultados.

Esto no solo habría sido más difícil de administrar (licencias, parches, mantenimiento y actualizaciones de datos), sino que también habría resultado en un peor rendimiento y habría requerido cantidades significativas de código personalizado.

El [Capítulo 17](https://subscription.packtpub.com/book/data/9781806028474/17), *Razonamiento Multimodal: Combinación de Vectores de IA con SQL Estándar*, presentará otra forma de vectorización utilizando IA y `pgvector`, donde codificamos información semántica (también conocida como *embeddings*) para calcular la similitud. Mientras que PostgreSQL FTS utiliza la derivación de palabras y diccionarios para vectorizar información sintáctica, el enfoque de IA/pgvector aprovecha los LLM para vectorizar información semántica.

---

### Búsqueda de similitud y coincidencia difusa basada en trigramas de PostgreSQL

Las búsquedas de texto que utilizan `LIKE` o `tsvector` siempre se basan en una ortografía correcta; un simple error tipográfico puede hacer que no devuelvan ningún resultado. Las búsquedas de similitud basadas en trigramas nos ayudan a abordar este problema introduciendo la coincidencia difusa (*fuzzy matching*) de cadenas de caracteres o frases.

La extensión `pg_trgm`, parte de la distribución estándar de PostgreSQL, descompone las cadenas en grupos de tres caracteres. Si es necesario, los trigramas se rellenan con espacios en blanco. `pg_trgm` determina la similitud de las cadenas contando cuántos de esos trigramas se encuentran en ambas cadenas.

Este ejemplo descompone la cadena `Uniqlo` y una falta de ortografía de esa marca para ilustrar la coincidencia difusa basada en trigramas:

```sql
SELECT show_trgm('Uniqlo'), show_trgm('Uniclo'), similarity('Uniqlo','Uniclo'), word_similarity('Uniqlo','Uniclo');
```

Podemos ver que `Uniqlo` y `Uniclo` comparten varios trigramas (`u`, `ni`, `lo`, `uni`), lo que ayuda a establecer el grado de similitud:

```text
-[ RECORD 1 ]---+------------------------------------ show_trgm | {" u"," un",iql,"lo ",niq,qlo,uni} show_trgm | {" u"," un",clo,icl,"lo ",nic,uni} similarity | 0.4 word_similarity | 0.42857143
```

La función `similarity` identifica la proporción de cadenas de tres caracteres que coinciden en ambas palabras: número de trigramas coincidentes / (número de trigramas en cadena1 + número de trigramas en cadena2 – número de trigramas coincidentes). La función `word_similarity` comprueba cuántos trigramas de la primera palabra coinciden con los de la segunda.

Cuando usamos `pg_trgm` en consultas, podemos configurar el umbral de similitud requerido para una coincidencia.

Una forma obvia de poner esta capacidad a trabajar es ayudar a los usuarios a lidiar con problemas de ortografía en nombres de personas o productos:

```sql
SELECT p.id, p.label AS product, b.label AS brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE b.label % 'Uniclo';
```

Esta consulta, con un nombre de marca mal escrito (`Uniclo` en lugar de `Uniqlo`), devolverá, no obstante, el resultado esperado:

```text
id | product | brand ----+--------------------+-------- 26 | Uniqlo Down Jacket | Uniqlo
```

Este tipo de coincidencia difusa también es beneficioso para la limpieza de datos, por ejemplo, para identificar nombres o direcciones coincidentes pero mal escritos.

Las búsquedas de autocompletado o de escritura anticipada (*type-ahead searches*) son otro caso de uso para los trigramas. Utilizando la función `similarity()`, podemos proponer fácilmente sugerencias de autocompletado:

```sql
CREATE FUNCTION type_ahead_product_search(search_text TEXT, limit_results INT DEFAULT 5) RETURNS TABLE (product_label TEXT, simi REAL) AS $$ BEGIN RETURN QUERY SELECT label::TEXT, similarity(label, search_text) AS simi FROM product WHERE label ILIKE search_text || '%' ORDER BY simi DESC LIMIT limit_results; END; $$ LANGUAGE plpgsql;
```

Cuando llamamos a la función anterior en una búsqueda web de productos, con la entrada inicial de `Casual Leather Jacket`, nos dirá que, según los productos disponibles en la base de datos, hay dos posibles opciones para completar: de Boss o de Aeropostale:

```sql
SELECT * FROM type_ahead_product_search('Casual Leather Jacket', 2);
```

```text
product_label | simi --------------------------------------+----------- Casual Leather Jacket by Boss | 0.7586207 Casual Leather Jacket by Aeropostale | 0.5945946
```

Las búsquedas por trigramas son extremadamente eficientes cuando están respaldadas por un índice GIN, y este enfoque escala a cientos de miles de filas.

##### Unaccent: una función auxiliar útil

La extensión `unaccent` es una función auxiliar valiosa. Elimina los signos diacríticos, o acentos, de las palabras:

```sql
CREATE EXTENSION IF NOT EXISTS unaccent; SELECT unaccent('Café Münsterländer');
```

Los signos diacríticos `é`, `ü` y `ä` han sido reemplazados para evitar fallos de búsqueda involuntarios:

```text
unaccent -------------------- Cafe Munsterlander (1 row)
```

Los acentos son una causa generalizada de fallos en las búsquedas. En general, vale la pena aplicar la función `unaccent` si hay acentos (por ejemplo, `é` o `à`), contracciones (por ejemplo, `¼`) u otros signos diacríticos (por ejemplo, `ä`, `ç`, `ñ` o `ü`) en el texto de la consulta o en el destino de la consulta.

La función `unaccent()` se puede aplicar fácilmente en una declaración `LIKE` o `ILIKE`. Para vectores de búsqueda de texto, la extensión `unaccent` proporciona un diccionario especial que se puede incluir en la configuración de búsqueda de texto para garantizar que se eliminen todos los signos diacríticos al producir lexemas. ¡Recuerda que los índices también deben aprovechar la función `unaccent` en ese caso!

---

### Orientación para desarrolladores

Las cuatro capacidades de búsqueda de texto tienen casos de uso muy diferentes y complementarios. La Tabla 14.1 ayudará a los desarrolladores a elegir la opción correcta para su caso de uso:

| Capacidad de PostgreSQL | LIKE | Expresiones Regulares | Vectores de Búsqueda de Texto (*tsvector*) | Trigramas (*pg_trgm*) |
| :--- | :--- | :--- | :--- | :--- |
| **Caso de uso principal** | Coincidencia de patrones con comodines (`%`, `_`) | Coincidencia de patrones (`~`, `~*`) | Búsqueda de texto completo basada en palabras (FTS) | Coincidencia difusa (*fuzzy matching*), similitud de texto |
| **Enfoque de coincidencia** | Patrones simples | Coincidencia de patrones complejos | Basado en palabras, relevancia clasificada | Trigramas basados en caracteres; puede encontrar palabras o subcadenas similares |
| **Indexación** | B-tree para búsquedas de prefijo | Soporte de índice limitado | GIN y GiST | GIN y GiST |
| **Rendimiento y escalabilidad** | Rendimiento deficiente en tablas grandes. A menudo lento para coincidencias en medio de cadenas. | Rendimiento medio | Alto rendimiento y altamente escalable | Alto rendimiento y altamente escalable |

*Tabla 14.1: Cuándo utilizar cada capacidad de búsqueda de texto en PostgreSQL*

#### Cuándo utilizar cada enfoque

- **Usa LIKE**: Para una coincidencia de patrones simple y precisa cuando el rendimiento no es una preocupación importante, especialmente para coincidencias de prefijo (`Postgres%`) donde se puede usar un índice B-tree de manera eficiente. *Ejemplo*: Filtrar direcciones de correo electrónico por nombre o dominio.
- **Usa expresiones regulares**: Al buscar patrones complejos durante la limpieza de datos. *Ejemplo*: Búsqueda de direcciones de Internet en textos.
- **Usa tsvector**: Para FTS de alto rendimiento donde deseas encontrar documentos basados en palabras, clasificación de relevancia y derivación de palabras. *Ejemplo*: Búsqueda en blogs y provisión de una clasificación de relevancia.
- **Usa pg_trgm**: Para coincidencias difusas e identificación de texto similar. *Ejemplo*: Usuarios que buscan un producto por nombre o proveedor.

---

### Resumen

En este capítulo, revisamos las sólidas funciones de búsqueda de texto de PostgreSQL. Combinadas con la creciente escalabilidad de PostgreSQL, estas capacidades permiten a los desarrolladores integrar la analítica basada en SQL con la búsqueda de texto, creando una plataforma empresarial convincente y versátil que integra el procesamiento SQL tradicional con diversas formas de analítica de texto. Esto no solo ayuda a mantener la simplicidad y evitar la integración de soluciones de búsqueda de texto de terceros, sino que también permite soluciones empresariales potentes que manejan ortografía multilingüe, errores tipográficos, autocompletado y el análisis de grandes conjuntos de documentos.

Este capítulo se centró en las capacidades de búsqueda sintáctica de texto de PostgreSQL (`LIKE`, expresiones regulares, FTS y trigramas). Encuentran cadenas basadas en la similitud de palabras, no en el significado.

En los siguientes capítulos, exploraremos el uso de IA y *embeddings*. Los *embeddings*, representados como vectores de IA, nos permiten trabajar con la semántica y el significado de datos polimórficos, como texto e imágenes. Aprenderás que, al igual que la búsqueda de texto, los *embeddings* y vectores basados en IA se integran estrechamente con las capacidades SQL de PostgreSQL.
