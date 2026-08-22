## Capítulo 16: Vectores e Indexación para IA con pgvector

A veces, pensamos que la IA es magia. Pero debajo de toda la "magia", hay una idea esencial: la IA entiende el mundo convirtiendo todo en números.

Palabras, imágenes, audio, descripciones de productos, comportamiento de los clientes: todo se convierte en una larga lista de números. Esta conversión se realiza mediante un enfoque especial de IA llamado *embedding*: un agente de IA extrae el significado de un elemento de datos (también conocido como semántica) y lo expresa como una lista organizada de números (el *embedding*). Los números representan una posición en el espacio de todo el significado y ponen el significado de ese elemento de datos en contexto con otros. Nos referimos a estas listas organizadas de números como vectores. Los elementos de datos relacionados (por ejemplo, un Lamborghini y un Ferrari) tendrán vectores similares y estarán cerca unos de otros en el espacio de significados. También puedes imaginar el vector como la huella digital semántica de un elemento de datos y, al igual que las huellas dactilares de parientes humanos, los vectores de elementos relacionados muestran cierta similitud, pero también diferencias significativas.

Si solo recuerdas una cosa de este capítulo, que sea esta: Un vector es una huella digital matemática de algo. Captura significado, patrones y similitudes.

Este capítulo profundiza en los vectores y su uso en aplicaciones de IA basadas en PostgreSQL. Exploraremos los siguientes temas:

- Vectores en general
- Vectores de IA en PostgreSQL
- Indexación para vectores de IA
- Cómo usar vectores de IA para crear aplicaciones de búsqueda semántica
- Cómo construir un motor de recomendaciones en PostgreSQL utilizando vectores de IA

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### ¿Qué es un vector?

Comencemos con lo básico. En su esencia, un vector es solo una lista ordenada de números, como `[0.2, 1.4, -0.8]`, donde cada número representa datos específicos. Los ejemplos incluyen lo siguiente:

- **Una fila en una hoja de cálculo**: Una colección de números en un orden específico (por ejemplo, ingresos, costos y ganancias). Esta lista ordenada de valores forma un vector.
- **Una identidad numérica larga**: Los números estándar internacionales de libros (*International Standard Book Numbers* o ISBN) son identificadores numéricos únicos para libros en un vector de cinco dimensiones. Por ejemplo, `978-0-306-40615-7` nos dice que es un libro (`978`), en inglés (`0`), publicado por Da Capo (`306`), con número de artículo `40615` y un dígito de control. Este es un vector de cinco dimensiones.
- **Coordenadas en un mapa**: Coordenadas bidimensionales en un mapa que identifican un punto en el mapa. Por ejemplo, el Empire State Building está en 40.7484° N, 73.9857° O, representado como dígitos decimales en un vector bidimensional.

Las coordenadas geográficas son un ejemplo directo y práctico de búsqueda vectorial significativa. Para encontrar una cafetería cerca del Empire State Building, busca cafeterías con coordenadas cercanas a 40.7484° N, 73.9857° O.

Exploremos algo más complejo, como una pieza de fruta.

#### Un ejemplo sencillo: Frutas como vectores

Describiremos las frutas a la computadora en términos que nos interesan: un vector tridimensional de dulzura, color y suavidad. Para cada dimensión, definiremos una escala de 0 a 1. La Figura 16.1 es una representación de alto nivel de frutas y verduras en un espacio tridimensional de dulzura, color y suavidad. Las cosas que están más cerca en este espacio tridimensional, como el mango y el plátano, también son más similares en nuestro mundo simplificado de frutas.

*Figura 16.1: Espacio tridimensional de frutas y verduras*

La Tabla 16.1 muestra el vector, o huella digital, de un plátano, que es muy dulce, amarillo y suave. El vector correspondiente es `[0.95, 0.90, 0.80]`. Esa es la "huella digital" numérica del plátano.

| Característica (dimensión) | Valor (0–1) |
| :--- | :--- |
| Dulzura (*Sweetness*) | 0.95 |
| Color | 0.90 |
| Suavidad (*Softness*) | 0.80 |

*Tabla 16.1: Características de un plátano como vector tridimensional*

La Tabla 16.2 nos muestra los vectores para otras frutas y verduras; la Figura 16.2 coloca los vectores en un contexto tridimensional.

| Fruta o verdura | Descripción | Vector de embedding (Gusto, Color, Suavidad) |
| :--- | :--- | :--- |
| Apple | Sweet, Medium red, Medium firm | 0.70, 0.60, 0.50 |
| Banana | Very sweet, Yellow, Soft | 0.95, 0.90, 0.80 |
| Mango | Very sweet, Orange, Very soft | 0.90, 0.80, 0.70 |
| Lemon | Very tart, Yellow, Firm | 0.30, 0.20, 0.40 |
| Orange | Sweet-tart, Orange, Firm | 0.80, 0.70, 0.40 |
| Peach | Sweet, Pinkish, Very soft | 0.85, 0.65, 0.90 |
| Grape | Sweet, Purple/green, Soft-ish | 0.75, 0.50, 0.40 |
| Kiwi | Tart-sweet, Brown/green, Soft | 0.60, 0.35, 0.60 |
| Potato | Starchy, Brown, Firm | 0.20, 0.15, 0.10 |
| Cherry | Sweet, Red, Firm | 0.80, 0.55, 0.50 |

*Tabla 16.2: Vectores (huellas digitales) para frutas y verduras*

Veamos ahora más de cerca:

- **Plátano [0.95, 0.90, 0.80] vs. Mango [0.90, 0.80, 0.70]**: Los números son muy cercanos en todas las dimensiones. La computadora concluye que son "vecinos" porque la distancia entre los vectores es pequeña.
- **Plátano [0.95, 0.90, 0.80] vs. Patata [0.20, 0.15, 0.10]**: Estos números están muy alejados en todas las dimensiones. La computadora concluye que son "distantes" porque la distancia entre los vectores es grande.

*Figura 16.2: Representación tridimensional de vectores de frutas y verduras*

Este simple cálculo se encuentra en el centro de la IA basada en vectores: si dos vectores tienen una distancia pequeña (o una puntuación de similitud alta), entonces las cosas que representan pueden considerarse similares. El ejemplo anterior es extremadamente simplista. Las aplicaciones reales de IA utilizan miles de dimensiones.

---

### Vectores de IA reales

En nuestra analogía simple, elegimos tres características (Dulzura, Color y Suavidad). Un modelo de IA real, después de haber sido entrenado con miles de millones de textos e imágenes, no solo utiliza tres características: encuentra miles de ellas por sí mismo.

En lugar de vectores tridimensionales, los modelos de IA generan vectores enormes, a menudo con 384, 768, 1.536 o incluso 4.096 dimensiones.

Un vector para una oración podría verse así (acortado por cordura): `[0.12, -0.08, 0.45, 0.91, -0.55, ... ]`.

Estos números no corresponden a características que podamos nombrar fácilmente, como la dulzura. Representan miles de conceptos abstractos, como el tono, la formalidad, el tema, el estilo y las complejas relaciones entre las palabras.

Estas huellas digitales gigantes son increíblemente poderosas. Nos permiten hacer lo siguiente:

- **Capturar el significado de una oración**: El vector para *How do I fix a broken car?* estará muy cerca del vector para *My automobile needs repairs*, a pesar de que no comparten una sola palabra.
- **Comprender la similitud de productos**: Una camiseta de cuello en V a rayas y un suéter de punto con ochos están conceptualmente más cerca que una camisa y una licuadora.
- **Representar las preferencias del usuario**: Un cliente que compra los productos A, B y C puede representarse mediante un vector que está muy cerca del de un cliente que compra los productos C y D, si A o C se parecen a D.
- **Coincidir en función de la intención, no de las palabras clave**: Una búsqueda tradicional de *what to wear in cold weather* buscaría esas palabras exactas. Una búsqueda vectorial comprende el propósito de la pregunta y puede relacionarla con un documento que dice: *A guide to winter parkas and thermal layers*, incluso si las palabras originales no están allí. Esta distinción entre coincidencia sintáctica y coincidencia semántica, basada en el significado, es clave.

Almacenamos el significado (también conocido como intención o semántica) como vectores, lo que crea un nuevo y masivo desafío. Cuando tenemos millones o incluso miles de millones de vectores, necesitamos una base de datos adecuada para gestionarlos, con un tipo de datos que admita almacenarlos, indexarlos y recuperarlos. Necesitamos lo siguiente:

- Una base de datos que entienda este nuevo tipo de datos vectorial
- Un sistema de "indexación" eficiente, como una guía telefónica de alta tecnología, para que no tengamos que escanear millones de vectores uno por uno
- Operadores para trabajar con vectores y buscar en la base de datos las coincidencias más cercanas: *Aquí tienes un nuevo vector. Encuéntrame los 10 vectores en mi base de datos que son matemáticamente más cercanos a él*

El ecosistema de PostgreSQL creó `pgvector`, una extensión de PostgreSQL que almacena vectores, proporciona índices de alto rendimiento y un conjunto de operadores para trabajar con ellos.

La mayor parte del trabajo con vectores se centra en encontrar vectores similares. Esto se llama búsqueda de similitud o búsqueda de vecinos más cercanos (*nearest neighbor search*).

---

### ¿Qué es pgvector?

`pgvector` es una extensión de PostgreSQL. Una extensión es un complemento especial o un módulo adicional que le da a la base de datos PostgreSQL nuevos superpoderes.

En este caso, `pgvector` le da a PostgreSQL el poder de hacer lo siguiente:

- Almacenar vectores (el nuevo tipo de datos `vector`)
- Indexar vectores para búsquedas de alta velocidad
- Buscar por similitud (encontrando los vecinos más cercanos)
- Admitir nuevos casos de uso, como búsqueda semántica, recomendaciones impulsadas por IA y chatbots

En resumen, `pgvector` transforma una base de datos PostgreSQL en una base de datos vectorial potente y lista para la empresa para aplicaciones de IA como búsqueda semántica, recomendaciones y procesamiento del lenguaje natural (NLP), integrando eficientemente datos estructurados con capacidades avanzadas de IA/ML. Simplifica las pilas tecnológicas al eliminar la necesidad de bases de datos separadas y habilitar cargas de trabajo de IA avanzadas con SQL familiar.

#### ¿Por qué es esto tan importante?

Esto es un gran avance para muchas empresas. Ya confían en PostgreSQL para sus datos más importantes: listas de clientes, catálogos de productos, transacciones financieras y más. Confían en ella porque es confiable, segura y estable. Como describimos en el [Capítulo 1](https://subscription.packtpub.com/book/data/9781806028474/1), PostgreSQL se ha convertido en la opción de base de datos número 1 para los desarrolladores.

Antes de `pgvector`, para construir una función de IA, había que configurar, administrar y proteger una base de datos vectorial separada. Esto significaba que los datos, el procesamiento y los procesos operativos estaban divididos y tenían que integrarse utilizando código personalizado, lo que a menudo requería copias de datos a gran escala o canalizaciones de sincronización para mantener alineados los *embeddings* y los datos de la fuente de verdad, todo lo cual es complicado, lento y costoso.

`pgvector` agregó capacidades de IA dentro de la misma base de datos que ya era conocida, bien entendida y confiable. El procesamiento de datos semánticos basados en IA se convirtió en una capacidad más de PostgreSQL, utilizando la misma licencia, conjunto de habilidades, procedimientos operativos y modelo de transacciones ACID.

#### Ejemplo real: ¿Qué se almacena como vector?

Hagamos esto real. Imagina que gestionamos un sitio web de comercio electrónico y tenemos una tabla de productos para administrar nombres de productos, marcas, categorías y descripciones.

Queremos agregar una "búsqueda inteligente" que comprenda lo que el usuario quiere decir, no solo lo que escribe:

- Usamos `pgvector` para agregar una nueva columna de vector a nuestra tabla `products`.
- Para cada producto, extraemos su descripción (nombre, categoría, etc.) y la pasamos por un modelo de *embedding* para generar una huella digital.
- Almacenamos esa huella digital en la nueva columna de vector.

Ahora, podemos comenzar a usar la huella digital (es decir, el significado semántico derivado del nombre, la categoría y la descripción) para ejecutar búsquedas inteligentes basadas en el significado en lugar de la sintaxis.

Exploremos esto con dos productos de muestra:

- **Producto 1**:
  - **Nombre**: *Olive Green Smartwatch*
  - **Descripción**: *A rugged smartwatch with GPS fitness tracking for outdoor activities*
  - **Vector**: (El modelo de IA lee este texto): `[0.014, -0.008, 0.129, ... 1536 números ...]`
- **Producto 2**:
  - **Nombre**: *Running Shoes - Waterproof Trail Edition*
  - **Descripción**: *Designed for outdoor runners with water-resistant material and durable grip*
  - **Vector**: (El modelo de IA lee este texto): `[0.016, -0.006, 0.131, ... 1536 números ...]`

Ahora, mira con atención. Estos dos productos no comparten muchas palabras clave. Uno es un reloj inteligente y el otro es un zapato. Una búsqueda clásica por palabra clave de *smartwatch* nunca encontraría las zapatillas para correr.

Pero el modelo de *embeddings* de IA entiende los conceptos. Ve *rugged*, *GPS fitness tracking*, *outdoor activities*, *outdoor runners* y *durable grip*. Reconoce que todos estos artículos pertenecen a la misma categoría general: equipo de fitness para exteriores (*outdoor fitness gear*).

Dado que los conceptos son similares, los dos vectores que genera serán vecinos: matemáticamente muy cercanos entre sí en el espacio de alta dimensionalidad del significado.

Así es como la IA comprende la similitud. Cuando un usuario busca *gear for my hike*, puedes convertir esa consulta de búsqueda en un vector y preguntarle a `pgvector`: *Encuéntrame todos los productos en mi base de datos cuyos vectores estén cerca del vector de esta consulta de búsqueda*.

Y lo hará, en milisegundos, devolviendo tanto el reloj inteligente como las zapatillas para correr. Ese es el poder que `pgvector` aporta a PostgreSQL.

---

### Del concepto al código: Encontrar la fruta más cercana

Volvamos a nuestra lista de frutas para mostrar cómo se ve esto en PostgreSQL.

#### Paso 1: Configurar nuestra pequeña tienda de frutas

Primero, habilitamos la extensión `pgvector`:

```sql
-- Ensure pgvector extension is installed CREATE EXTENSION IF NOT EXISTS vector;
```

Luego, creamos una tabla para almacenar nuestras frutas junto con sus huellas digitales tridimensionales:

```sql
DROP TABLE IF EXISTS fruit_vectors; CREATE TABLE fruit_vectors ( id SERIAL PRIMARY KEY, name TEXT NOT NULL, description TEXT, features VECTOR(3) -- [taste, color, softness] );
```

Las características del vector tridimensional se utilizarán para representar la huella digital, incluida la dulzura, el color, la textura o cualquier otra cosa que desees. El punto es este: PostgreSQL trata la huella digital como un ciudadano de primera clase.

#### Paso 2: Cargar las huellas digitales

Inicializaremos nuestra pequeña tienda de frutas:

```sql
INSERT INTO fruit_vectors (name, description, features) VALUES ('Apple', 'Sweet, medium red, medium firm', '[0.70, 0.60, 0.50]'), ('Banana', 'Very sweet, yellow, soft', '[0.95, 0.90, 0.80]'), ('Mango', 'Very sweet, orange, very soft', '[0.90, 0.80, 0.70]'), ('Lemon', 'Very tart, yellow, firm', '[0.30, 0.20, 0.40]'), ('Orange', 'Sweet-tart, orange, firm', '[0.80, 0.70, 0.40]'), ('Peach', 'Sweet, pinkish, very soft', '[0.85, 0.65, 0.90]'), ('Grape', 'Sweet, purple/green, soft-ish', '[0.75, 0.50, 0.40]'), ('Kiwi', 'Tart-sweet, brown/green, soft', '[0.60, 0.35, 0.60]'), ('Potato', 'Starchy, brown, firm', '[0.20, 0.15, 0.10]'), ('Cherry', 'Sweet, red, firm', '[0.80, 0.55, 0.50]');
```

En aras de este ejemplo simple, hemos inicializado las descripciones de texto y los vectores/huellas digitales. Como veremos más adelante, en aplicaciones del mundo real, se utilizaría un modelo de lenguaje grande (LLM) para extraer el significado de las descripciones e incrustarlas en vectores.

Ahora, Postgres sabe lo que nosotros sabemos: Mango se siente cercano a Banana, Lemon es algo similar, y Potato... bueno, Potato está viviendo su propia vida separada.

#### Paso 3: Encontrar fruta similar

Ahora viene la parte interesante: ejecutemos búsquedas semánticas basadas en los *embeddings*.

Primero, le pediremos a PostgreSQL que compare cada fruta con el plátano e informe la similitud o la distancia semántica.

`pgvector` proporciona varios operadores de similitud:

- `<->`: Distancia L2 (también conocida como distancia euclidiana)
- `<#>`: Producto interno (negativo)
- `<=>`: Distancia del coseno
- `<+>`: Distancia L1
- `<~>`: Distancia de Hamming (vectores binarios)
- `<%>`: Distancia de Jaccard (vectores binarios)

Para nuestro ejemplo simple, usaremos la distancia euclidiana `<->`, que es la distancia estándar en línea recta entre dos puntos en un espacio. Se calcula utilizando una versión generalizada del teorema de Pitágoras y se usa ampliamente en matemáticas, aprendizaje automático y análisis de datos.

Primero, le pediremos a PostgreSQL que compare la huella digital de cada fruta en la tabla con la huella digital del plátano, `[0.95, 0.90, 0.80]`. El operador `<->` es la forma en que `pgvector` dice: "Mide la distancia entre estos dos vectores".

Y luego ordenamos los resultados por esa distancia:

```sql
-- Euclidean distance operator (<->) to find the fruits that have banana-like features SELECT id, name, description, features <-> '[0.95,0.90,0.80]' AS distance FROM fruit_vectors ORDER BY distance ASC;
```

Y de repente, nuestra intuición se vuelve cuantificable:

- Banana se sitúa justo en el centro: distancia cero de sí misma.
- Mango es el vecino más cercano, como esperaríamos dada nuestra comprensión de la similitud.
- Orange está relacionada, pero no tan fuertemente.
- Potato es... una patata: muy lejos en este universo de sabores.

```text
id | name | description | distance ----+--------+-------------------------------+--------------------- 2 | Banana | Very sweet, yellow, soft | 0 3 | Mango | Very sweet, orange, very soft | 0.14999999677141504 6 | Peach | Sweet, pinkish, very soft | 0.2872280991242216 5 | Orange | Sweet-tart, orange, firm | 0.4716990528119823 10 | Cherry | Sweet, red, firm | 0.48476795438810844 ... 9 |Potato | Starchy, brown, firm. | 1.2708265064660649
```

Esta es la búsqueda vectorial en su forma más pura. Sin magia. Sin caja negra. Solo matemáticas y ordenación.

Ya sea que tus vectores tengan 3 números o 3.000, el principio sigue siendo el mismo:

1. Encuentra los elementos con la distancia más pequeña (es decir, los más similares).
2. Devuélvelos en orden de similitud.

Eso es la "búsqueda de similitud". Una idea simple, pero que impulsa las recomendaciones modernas, la búsqueda, la personalización y las experiencias impulsadas por IA que nos rodean.

---

### Indexación para IA: Encontrar la aguja en el pajar

En la sección anterior, aprendimos cómo representar nuestros datos, por ejemplo, productos o perfiles de clientes, en huellas digitales vectoriales, almacenarlos en PostgreSQL y realizar búsquedas de vecinos más cercanos para encontrar elementos de datos semánticamente similares.

Nuestro ejemplo de tienda de frutas tenía 10 artículos en el estante. En la vida real, ¡tendríamos miles o millones de productos en el estante! Necesitamos una forma de encontrar los vectores más cercanos o más similares al instante. Este es el motor detrás de todo tipo de funciones "inteligentes":

- "Encontrar productos similares a este"
- "Mostrarme descripciones de productos que signifiquen lo mismo que mi búsqueda"
- "Encontrar clientes que sean similares a mis mejores clientes"
- "Recomendar productos alternativos si este está agotado"

Los índices nos ayudan a resolver este problema. Tal como comentamos en el [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), en la sección de tipos de datos de PostgreSQL, los tipos de datos y los operadores tienen índices específicos. Por ejemplo, los enteros y las operaciones de igualdad se benefician de los índices B-tree, o los rangos de fechas aprovechan los índices GiST al verificar superposiciones.

Sin un índice, pedirle a Postgres que encuentre el vector más cercano es como intentar encontrar un solo nombre en una guía telefónica de un millón de páginas donde todos los nombres están en un orden aleatorio. Tendrías que leer cada página, una por una. Esto se llama un escaneo completo (*full scan*) y es dolorosamente lento.

De fábrica, `pgvector` proporciona dos índices:

- **IVFFlat** (el índice de supermercado)
- **HNSW** (el índice de mapa de ciudad)

Usaremos un ejemplo simple para explicar la diferencia y proporcionar orientación sobre qué índice usar y cuándo.

#### Cómo funciona la búsqueda vectorial

Antes de profundizar, reflexionemos sobre lo que significa realmente "más cercano". Para medir la cercanía, la base de datos utiliza una métrica de distancia. Puedes pensar en ellas como diferentes tipos de reglas de medir:

- **Distancia L2, `<->` (Euclidiana)**: Esta es la distancia en línea recta "en línea recta" (*as the crow flies*). Mide la distancia estándar en línea recta entre dos puntos en un espacio vectorial. Se calcula utilizando una versión generalizada del teorema de Pitágoras y se utiliza ampliamente en matemáticas, aprendizaje automático y análisis de datos.
- **Distancia del coseno, `<=>`**: Esta es la más popular para comparaciones de texto. No mide la longitud de los vectores; solo mide el ángulo entre ellos. Esto es perfecto para comprender temas.
- **Producto interno, `<#>`**: El producto interno combina la distancia en línea recta y el ángulo entre los vectores. En `pgvector`, `<#>` devuelve el producto interno negativo, por lo que una mayor similitud corresponde a valores más negativos (y ordenar de forma ascendente sigue trayendo primero los resultados más similares).

`pgvector` también proporciona otras métricas de distancia, pero rara vez se utilizan:

- **Distancia L1, `<+>`**: Resulta de sumar todas las distancias de las dimensiones individuales.
- **Distancia de Hamming (solo vectores binarios), `<~>`**: Representa el número de dimensiones en las que difieren dos vectores.
- **Distancia de Jaccard (vectores binarios), `<%>`**: Representa la disimilitud entre dos vectores binarios.

Ahora que comprendemos el concepto de distancia (o similitud, o vecino más cercano), averigüemos cómo podemos usarlo en un caso de uso de gran volumen y cómo los índices de `pgvector` (IVFFlat y HNSW) pueden ayudarnos a lograrlo.

#### Tipo de índice 1: IVFFlat (el índice de supermercado)

Imagina un supermercado gigante con un millón de artículos repartidos aleatoriamente por los estantes. Si quieres encontrar canela, tendrías que recorrer todos los pasillos y revisar cada estante. Eso es un escaneo completo.

Para solucionar esto, el gerente de la tienda agrupa los artículos en secciones lógicas:

- Especias
- Pan
- Bebidas
- Verduras
- Lácteos

Así es precisamente como funciona el índice *Inverted File with Flat Compression* (IVFFlat):

- **Configuración (agrupamiento / clustering)**: Cuando creas el índice, `pgvector` observa todos tus millones de vectores de productos y los agrupa automáticamente en depósitos (o centroides) según su similitud:
  - Depósito A: Artículos de cocina
  - Depósito B: Equipo para actividades al aire libre
  - Depósito C: Electrónica
  - Depósito D: Ropa
- **Búsqueda (*probing*)**: Cuando buscas un nuevo vector de botas de montaña, el índice primero identifica el Depósito B (Equipo para actividades al aire libre) como el centroide más relevante. Solo busca entradas agrupadas alrededor de ese centroide, ignorando por completo los Artículos de cocina y la Electrónica.

Esto hace que la búsqueda sea cientos de veces más rápida. Es una búsqueda aproximada porque la coincidencia perfecta podría estar en otro depósito, pero es increíblemente rápida y suficiente para la mayoría de los casos de uso.

Cuándo usar IVFFlat:

- Ideal para conjuntos de datos masivos (millones o miles de millones de vectores)
- Excelente para cargas de trabajo por lotes (*batch*) (por ejemplo, ejecutar un informe nocturno para encontrar segmentos de clientes)
- Una buena opción cuando se prefiere algo casi perfecto y muy rápido en lugar de perfecto y lento

##### Cómo construir y usar un índice IVFFlat

Profundicemos en la creación de índices IVFFlat. Cuando creas un índice IVFFlat, puedes ajustar dos parámetros de configuración: `lists` y `probes`. Volveremos a la analogía del supermercado para explicarlo:

- **El parámetro `lists`**: El parámetro `lists` corresponde al número de pasillos en el supermercado. Especifica el número de clústeres que el índice IVFFlat utiliza para clasificar los elementos.
  - *El balance*:
    - *Muy pocas listas (por ejemplo, lists = 10)*: Es como tener un supermercado con solo 10 pasillos gigantes (Comida, Limpieza, Electrónica, etc.). Tu pasillo de Comida sigue siendo enorme y lento de buscar.
    - *Demasiadas listas (por ejemplo, lists = 10000)*: Es como tener un pasillo para la sal, otro para la pimienta y otro para el pimentón. Tendrás miles de pasillos pequeños, y solo encontrar el pasillo correcto se vuelve lento.
  - *La regla general*: Quieres un número equilibrado. Un gran punto de partida es la raíz cuadrada de tu número de filas. Si tienes 1 millón de productos, la raíz cuadrada es 1.000. En nuestro ejemplo, `lists = 1000` es un punto de partida perfecto. Estableces este parámetro cuando creas el índice IVFFlat para la columna específica y la operación (en este caso, similitud de coseno) que deseas respaldar con el índice:

```sql
CREATE INDEX ON product -- identify the index type, the column to index, and the operator class USING ivfflat (embedding vector_cosine_ops) WITH (lists = 1000);
```

- **El parámetro `probes`**: Este control define cuántos pasillos se buscarán en el supermercado para encontrar tu producto. Por ejemplo, las botas de montaña están en el límite entre el pasillo de Equipo para actividades al aire libre y el pasillo de Calzado. Si decides dejar de buscar después de un solo pasillo, es posible que no encuentres tus botas, aunque estén en stock.
  - Por defecto, Postgres comprueba solo el mejor pasillo (`probes = 1`), pero puedes indicarle que compruebe más.
  - *El balance*:
    - `probes = 1` (predeterminado): Compruebas solo el pasillo que mejor coincide, como Equipo para actividades al aire libre. Esta búsqueda es de alta velocidad, pero podría fallar si las botas están en el pasillo de Calzado.
    - `probes = 3`: Le dices a Postgres: *Revisa el pasillo de Equipo para actividades al aire libre y los otros dos pasillos que estén más cerca de él (probablemente Calzado y Camping)*.
  - *El resultado*: Aumentar el número de `probes` hace que tu búsqueda sea más precisa pero potencialmente un poco más lenta.

Estableces el valor del parámetro `probes` antes de ejecutar tu búsqueda:

```sql
-- EXAMPLE (pseudocode): FAST, "GOOD ENOUGH" SEARCH (Default) -- Replace [...your_search_vector...] with an actual vector value. -- By default, ivfflat.probes is 1. This is the fastest search. SELECT product_id, product_name FROM product ORDER BY embedding <=> [...your_search_vector...]::vector LIMIT 5; -- EXAMPLE: HIGH-ACCURACY, SLOWER SEARCH -- Replace [...your_search_vector...] with an actual vector value. -- 1. Tell Postgres to be "more accurate" for this one query BEGIN; SET LOCAL ivfflat.probes = 10; -- 2. Run the exact same query -- This time, Postgres will search the 10 best-matching "aisles" -- Replace [...your_search_vector...] with an actual vector value. SELECT product_id, product_name FROM product ORDER BY embedding <=> [...your_search_vector...]::vector LIMIT 5 END;
```

#### Tipo de índice 2: HNSW (el índice de mapa de ciudad)

Imagina que estás en una ciudad enorme y en expansión y quieres encontrar una cafetería específica:

- *La forma lenta (sin índice)*: Camina por cada calle de toda la ciudad hasta encontrarla.
- *La forma IVFFlat*: Ve al distrito de las cafeterías (si existe) y busca en todas las calles de ese distrito. Esto es más rápido, pero podrías perderte una nueva cafetería que acaba de abrir en el distrito financiero.

El índice *Hierarchical Navigable Small Worlds* (HNSW) nos ayuda a abordar este problema. HNSW funciona como un GPS inteligente de múltiples capas y construye múltiples niveles de conexiones (como una red de autopistas, calles y carriles) para encontrar la ruta más rápida posible a cualquier destino:

- **Capa superior (autopistas)**: Conecta solo unos pocos vectores "nodo central" (*hub*) que están muy separados. Para los datos de tus productos, esto podría conectar el nodo de Electrónica con el nodo de Artículos para el hogar.
- **Capa intermedia (calles)**: Un mapa más detallado que te conecta con diferentes vecindarios. Esto podría conectar Portátiles con Teclados.
- **Capa inferior (carriles/calles locales)**: Un mapa muy detallado que conecta cada casa (vector) con sus vecinos directos. Esto conecta un Dell XPS 13 con un MacBook Air.

Veamos cómo funciona la búsqueda HNSW:

1. Comienzas en la autopista (capa superior) y encuentras el nodo central más cercano a tu búsqueda (por ejemplo, Electrónica).
2. Desde ese nodo central, haces zoom y bajas a las calles (por ejemplo, la sección de Portátiles).
3. Sigues las calles, siempre acercándote a tu objetivo, hasta que bajas a los carriles para encontrar la dirección exacta (por ejemplo, el MacBook Air) que estás buscando.

Este método es increíblemente rápido y extremadamente preciso. Es la mejor opción para encontrar los vecinos más cercanos aproximados para aplicaciones en tiempo real.

Cuándo usar HNSW:

- La mejor opción para la mayoría de los casos de uso en producción
- Extremadamente rápido para IA en tiempo real (por ejemplo, resultados de búsqueda instantáneos, recomendaciones en vivo de "También te podría gustar...")
- Proporciona la mayor precisión

##### Cómo construir y usar un índice HNSW

HNSW tiene dos parámetros: `m` (número máximo de conexiones) y `ef_construction` (calidad de búsqueda durante la construcción). Ambos parámetros se configuran durante la creación del índice:

- **`m`**: Este es el número de carreteras (conexiones) que cada casa (vector) puede tener en cada capa de tu mapa.
  - *El balance*:
    - *Valor bajo de m (por ejemplo, m = 4)*: Cada casa solo tiene unos pocos caminos. El mapa es simple, ligero (tamaño de índice pequeño) y rápido de construir, pero es más fácil perderse o perderse un atajo inteligente, lo que lleva a resultados menos precisos.
    - *Valor alto de m (por ejemplo, m = 32)*: Cada casa es un centro neurálgico con muchos caminos que se conectan con todos sus vecinos. El mapa es muy detallado, pesado (gran tamaño de índice) y lento de construir, pero los resultados de búsqueda son extremadamente precisos.
  - *La regla general*: El valor predeterminado es `m = 16`. Este es un punto de partida equilibrado. Puedes aumentarlo a 24 o 32 para una mayor precisión, a costa de memoria.

- **`ef_construction`**: Esta es la diligencia de tu constructor de mapas. Al agregar una nueva casa al mapa, este número controla con qué empeño el constructor busca las mejores carreteras para conectarla.
  - *El balance*:
    - *Valor bajo de ef_construction (por ejemplo, ef_construction = 50)*: El constructor es perezoso. Conecta la nueva casa a las primeras carreteras buenas que encuentra. Esto es muy rápido de construir, pero la calidad del mapa puede ser deficiente (por ejemplo, los caminos pueden llevar a callejones sin salida).
    - *Valor alto de ef_construction (por ejemplo, ef_construction = 400)*: El constructor es perfeccionista. Busca diligentemente las mejores carreteras absolutas para conectar la nueva casa. Esto hace que tu índice sea mucho más lento de construir, pero el mapa final es de alta calidad, lógico y proporciona resultados de búsqueda muy precisos.
  - *La regla general*: El valor predeterminado es `ef_construction = 64`. Esto está bien para pruebas pequeñas. Para un índice de producción real, querrás aumentar esto (por ejemplo, de 100 a 400) y dejarlo ejecutándose durante la noche. Un mejor mapa en tiempo de construcción significa búsquedas más rápidas y precisas más adelante.

##### Cómo crear un índice HNSW (con parámetros)

Establecemos los valores de `m` y `ef_construction` durante la creación del índice:

```sql
-- Create our "City Map" index with 16 "roads" per house -- and a "perfectionist" builder CREATE INDEX ON embeddings.product_embedding -- identify the index type, the column to index, and the operator class USING hnsw (embedding vector_cosine_ops) -- m=16 means each node connects to 16 neighbors (default) -- ef_construction=200 means higher accuracy during index building WITH (m = 16, ef_construction = 200);
```

##### Búsqueda con HNSW

Así como IVFFlat tiene un parámetro en tiempo de búsqueda llamado `probes`, HNSW tiene un control llamado `ef_search`. Define con cuánta exhaustividad deseas buscar en el mapa. Es la diligencia de búsqueda de tu GPS:

- *El balance*:
  - *Valor bajo de ef_search (por ejemplo, ef_search = 20)*: Tu GPS simplemente encuentra una buena ruta. Encontrar la ruta es súper rápido, pero la ruta podría no ser la mejor absoluta.
  - *Valor alto de ef_search (por ejemplo, ef_search = 100)*: Tu GPS piensa más profundamente, comprobando múltiples rutas para asegurarse de encontrar el atajo perfecto. El GPS es más preciso, es probable que la ruta sea mejor, pero la búsqueda puede ser un poco más lenta.
- *La regla general*: Comienza con la configuración predeterminada de `ef_search = 40`. Puedes aumentarlo en el momento de la búsqueda si necesitas una mayor precisión. Estableces este control antes de ejecutar tu búsqueda, al igual que con IVFFlat:

```sql
-- EXAMPLE: HIGH-ACCURACY SEARCH -- 1. Tell PostgreSQL to be "more diligent" for this search BEGIN; SET LOCAL hnsw.ef_search = 100; -- 2. Run the query -- PostgreSQL will search the "map" more thoroughly -- Replace [...your_search_vector...] with an actual vector value. SELECT product_id, product_name FROM product ORDER BY embedding <=> [...your_search_vector...]::vector LIMIT 5; END;
```

#### Elegir el índice adecuado para pgvector

Como se describió anteriormente, `pgvector` nos proporciona dos índices: IVFFlat y HNSW. Funcionan de manera diferente (ver Figura 16.3) y tienen diferentes fortalezas y debilidades (ver Tabla 16.3).

*Figura 16.3: Comparación de IVFFlat y HNSW*

La Tabla 16.3 resume sus fortalezas y debilidades.

| Característica | IVFFlat | HNSW |
| :--- | :---: | :---: |
| Conjunto de datos grande (eficiente en almacenamiento) | ✅ | |
| Conjunto de datos estático | ✅ | |
| Conjunto de datos dinámico/frecuentemente actualizado | | ✅ |
| Carga de trabajo por lotes/fuera de línea (*batch/offline*) | ✅ | |
| Preferencia por velocidad (baja latencia) sobre precisión y tamaño | | ✅ |
| Preferencia por precisión (*recall*) sobre tamaño y velocidad | | ✅ |
| Menor consumo de memoria | ✅ | |

*Tabla 16.3: Comparación de IVFFlat con HNSW*

La Tabla 16.4 proyecta esas capacidades sobre casos de uso para ayudar al desarrollador a tomar una decisión informada.

| Tu caso de uso | Índice recomendado | ¿Por qué? |
| :--- | :--- | :--- |
| Necesito resultados instantáneos en tiempo real (por ejemplo, búsqueda semántica, chatbots y motor de recomendaciones) | **HNSW** | Optimizado para consultas de baja latencia y alto rendimiento con mayor recuperación (*recall*) a una latencia determinada, ideal cuando los usuarios esperan respuestas. |
| Tengo un conjunto de datos muy grande, en su mayoría estático, y ejecuto análisis periódicos o puntuación por lotes | **IVFFlat** | La creación de índices es mucho más rápida y los índices son más pequeños, lo que lo convierte en una buena opción para colecciones enormes donde se puede reconstruir ocasionalmente y las consultas son menos sensibles a la latencia. |
| Mi conjunto de datos es grande y se actualiza con frecuencia (documentos, eventos de usuario, IoT, etc.) | **HNSW** | Más robusto ante inserciones/actualizaciones continuas; la recuperación se degrada menos con la rotación de datos y no requiere reconstrucciones completas con tanta frecuencia como IVFFlat. |
| Estoy comenzando y quiero un valor predeterminado seguro | **HNSW** | Ofrece una recuperación sólida con un ajuste mínimo y funciona bien en muchas cargas de trabajo de búsqueda semántica y RAG; principalmente ajustas `ef_search` para velocidad versus precisión. |
| Tengo recursos limitados y me importa principalmente el tamaño del índice y el tiempo de construcción | **IVFFlat** | Utiliza menos memoria y se construye mucho más rápido que HNSW, a costa de cierta recuperación y más esfuerzo de ajuste. |

*Tabla 16.4: Mapeo de índices de pgvector a casos de uso*

Como regla general simple, recomendamos comenzar con HNSW. Cubre bien muchos casos de uso.

---

### Búsqueda semántica con pgvector

Hasta ahora, hemos visto cómo se usan los vectores para identificar a los vecinos más cercanos, por ejemplo, qué fruta es como un plátano. Para responder a esa consulta, comparamos los vectores de todas las frutas y devolvimos una lista ordenada por similitud. ¡Esos son los vecinos más cercanos del plátano!

Ahora, exploraremos la búsqueda semántica, que compara el significado de un término de consulta con los *embeddings* (es decir, la semántica) de nuestros elementos de datos. Por ejemplo, le pedimos al sistema que nos muestre *men's premium clothes*, y compara la intención de esta consulta con los significados de todas las descripciones de productos, devolviendo productos que coinciden semánticamente con la intención.

La búsqueda de texto completo que revisamos en el [Capítulo 14](https://subscription.packtpub.com/book/data/9781806028474/14) selecciona registros basados en coincidencias sintácticas. Los lexemas, la derivación de palabras, las expresiones regulares y los trigramas permiten una coincidencia potente, pero sigue siendo una coincidencia basada en la sintaxis y no en la semántica.

La búsqueda semántica es diferente: busca por significado en lugar de por palabras exactas.

#### Enfoque paso a paso para la búsqueda semántica

En la sección anterior, vimos cómo almacenar la semántica de un elemento de datos como vectores en PostgreSQL, indexarlos y usar los operadores de `pgvector` para identificar a los vecinos más cercanos.

En esta sección, recorreremos el proceso de creación de *embeddings* para el conjunto de datos y la consulta para ejecutar una búsqueda semántica, y repasaremos paso a paso los aspectos técnicos en PostgreSQL.

##### El proceso de búsqueda semántica

La Figura 16.4 describe los siguientes pasos para preparar el conjunto de datos para la búsqueda semántica, ejecutar una consulta y devolver los resultados:

1. Los elementos de datos que queremos usar para nuestra búsqueda semántica (nombres de productos, descripciones, marca e información de categoría) se reúnen en un solo documento de texto descriptivo.
2. El texto descriptivo se envía a un LLM, como OpenAI, para generar *embeddings*. Esta es una operación por lotes o se realiza cada vez que cambian las etiquetas o las descripciones.
3. El LLM devuelve los *embeddings* como un vector.
4. El vector se almacena en PostgreSQL y se asocia con el producto original.
5. Cuando un usuario emite una consulta, por ejemplo, *Show me men's premium attire*, el texto de esa solicitud también se envía al LLM para su análisis semántico (2) y la creación de los *embeddings* para esa consulta (3).
6. Luego, el *embedding* se usa en la consulta, no el texto original.
7. PostgreSQL ejecuta la consulta contra el vector de consulta y los vectores del conjunto de datos para identificar el conjunto de resultados (8).

*Figura 16.4: El proceso de búsqueda semántica*

##### Implementación de la búsqueda semántica en PostgreSQL

En PostgreSQL, ejecutamos los siguientes pasos técnicos:

1. Crear la extensión `pgvector`. Esto proporcionará el tipo de datos `vector`, los índices HNSW e IVFFlat, y los operadores de similitud (`<->`, `<=>`, etc.):

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

2. Agregar una tabla para almacenar la información semántica sobre los productos. Esta tabla almacena la "huella digital" semántica del producto, basada en la etiqueta, la marca, la categoría y la descripción corta. Un vector con un tamaño de 1536 admite *embeddings* de texto de la mayoría de los modelos de IA populares:

```sql
CREATE TABLE embeddings.product_embeddings ( product_id INTEGER PRIMARY KEY REFERENCES product.product(id) ON DELETE CASCADE, embedding VECTOR(1536) NOT NULL);
```

3. Generar los *embeddings* del producto mediante un procedimiento almacenado en PL/Python3u que combine los elementos descriptivos de la definición del producto (etiqueta, marca, categoría y descripciones) y llame a OpenAI para generar el *embedding* correspondiente. El vector de *embeddings* se almacena en la tabla `product_embedding`:

```sql
SELECT api.embed_products();
```

4. Indexar los *embeddings* utilizando HNSW, optimizando para la distancia L2:

```sql
CREATE INDEX idx_product_embedding_hnsw ON embeddings.product_embedding USING hnsw (embedding vector_l2_ops );
```

5. Realizar una búsqueda semántica:
   - Llamar a la función PL/Python `openai_embed`, que envía el texto a OpenAI y devuelve el *embedding* de la consulta como un vector.
   - Alinear los productos y sus *embeddings*.
   - Consultar qué productos coinciden con el *embedding* de la consulta utilizando la distancia L2 (`<->`):

```sql
WITH -- CTE to get the query embedding query_embedding AS ( SELECT api.openai_embed('Men''s attire for a formal occasion')::vector AS vec ), -- CTE to get products with their embeddings prod_w_embedding AS ( SELECT p.id, p.label, pe.embedding FROM product AS p JOIN embeddings.product_embedding pe ON p.id = pe.product_id ) -- Final selection with similarity calculation SELECT pwe.id, pwe.label, 1 - (pwe.embedding <=> qe.vec) AS similarity FROM prod_w_embedding AS pwe, query_embedding AS qe ORDER BY pwe.embedding <-> qe.vec LIMIT 3;
```

Esto genera la lista priorizada de productos cuyas descripciones son más similares al texto de la consulta *Men's attire for a formal occasion*.

```text
id | label | similarity ----+-----------------------------------------+--------------------- 14 | Suit coat - business perfect, by Brioni | 0.47748804226589037 2 | Dress shirt by Boss | 0.466213031596334 3 | Dress shirt by Eaton | 0.4610442656589828
```

Los *embeddings* (Pasos 2 y 3) también podrían almacenarse en una columna adicional en lugar de en una tabla separada.

Las funciones PL/Python3u que generan los *embeddings* del producto (Paso 3) y el *embedding* de la consulta están en el repositorio de GitHub y se explicarán en detalle.

##### Qué sucede internamente

Esto es lo que hace PostgreSQL entre bastidores para encontrar los resultados:

1. Toma la huella digital para *Men's attire for a formal occasion*.
2. Ingresa al índice HNSW en el nivel superior de autopista.
3. Viaja hacia abajo a través de calles y carriles más pequeños.
4. Se detiene cuando encuentra las casas (productos) más cercanas.

PostgreSQL no escanea toda la lista de productos: devuelve resultados en milisegundos.

#### Búsqueda sintáctica vs. Búsqueda semántica

Ahora que sabemos cómo funciona la búsqueda semántica, exploremos la diferencia entre las capacidades de búsqueda de texto completo sintáctico que exploramos en el [Capítulo 14](https://subscription.packtpub.com/book/data/9781806028474/14) y la búsqueda semántica. Imagina que buscamos en el conjunto de datos de productos de muestra *Men's attire for a formal occasion*. Incluso utilizando el potente FTS de PostgreSQL, la búsqueda no encuentra ninguna coincidencia en nuestro conjunto de datos de muestra, ya que ninguna de las etiquetas o descripciones coincide sintácticamente con el texto de la consulta:

```sql
SELECT DISTINCT p.id, p.label product, b.label brand FROM product p JOIN brand b ON p.brand_id = b.id WHERE p. longdescription @@ websearch_to_tsquery('english', 'Men''s attire for a formal occasion');
```

Como vimos anteriormente, la búsqueda semántica proporciona resultados significativos incluso cuando ninguna de las palabras coincide. Solo la intención (también conocida como el significado) coincidió, y eso es suficiente.

---

### Uso de pgvector para construir un motor de recomendaciones

Los motores de recomendación se han convertido en parte de las experiencias cotidianas al comprar, escuchar música o mirar televisión:

- Amazon: "Los clientes que compraron esto también compraron..."
- Netflix: "Porque viste The Crown..."
- Spotify: "Tu Daily Mix 1"

Te mostraremos cómo usar PostgreSQL 18 con `pgvector` para crear recomendaciones directamente dentro de tu base de datos.

#### Tipos de recomendaciones

Primero, veamos los diferentes tipos de motores de recomendación:

- **Recomendaciones de productos similares**: Muestra artículos similares al que un usuario está viendo ("También te podría gustar...")
- **Recomendaciones basadas en el usuario**: Sugiere productos que les gustaron a usuarios similares
- **Recomendaciones basadas en gustos**: Sugiere productos que coincidan con el historial de navegación de un usuario

Ahora, te mostraremos cómo usar `pgvector` para implementar motores de recomendación.

#### Recomendaciones de producto a producto

Las recomendaciones de producto a producto son como nuestro ejemplo anterior: "Encontrar frutas que sean como los plátanos".

Supongamos que un usuario está considerando actualmente la chaqueta de cuero para hombre de Boss (`product_id = 9`) y nos gustaría sugerir tres productos similares.

La siguiente consulta obtiene primero el *embedding* (es decir, la codificación semántica) del producto considerado actualmente. Luego, compara el *embedding* con los de todos los demás productos, excluye la chaqueta de cuero y presenta tres opciones adicionales en orden de similitud semántica:

```sql
-- Example query that finds similar products to a given product ID WITH -- CTE to get the embedding of the target product target_product_embedding AS ( SELECT pe.embedding FROM embeddings.product_embedding pe WHERE pe.product_id = 9 -- Replace with the target product ID ), -- CTE to get products with their embeddings prod_w_embedding AS ( SELECT p.id, p.label, pe.embedding FROM product AS p JOIN embeddings.product_embedding pe ON p.id = pe.product_id ) -- Final selection with similarity calculation SELECT pwe.id, pwe.label, 1 - (pwe.embedding <=> tpe.embedding) AS similarity FROM prod_w_embedding AS pwe, target_product_embedding AS tpe WHERE pwe.id <> 9 -- Exclude the target product itself ORDER BY pwe.embedding <=> tpe.embedding LIMIT 3;
```

Esta consulta devuelve las coincidencias más cercanas al instante: otra chaqueta de cuero, un abrigo deportivo y una gabardina:

```text
id | label | similarity ----+----------------------------------------+-------------------- 10 | Casual Leather Jacket by Aeropostale | 0.6645131511076556 13 | Sports coat - business casual, by Boss | 0.6453439804706597 15 | Trench coat - always perfect, by Boss | 0.6345191976141462
```

#### Listas de preferencias de usuario al estilo Spotify

En este segundo ejemplo, crearemos un conjunto de recomendaciones al estilo Spotify basadas en transacciones de compras anteriores. Supongamos que un cliente compró recientemente una chaqueta de cuero (`product_id = 9`), jeans Levi's (`8`) y una camiseta de The Gap (`6`), y nos gustaría crear una lista personalizada de cinco recomendaciones que se alineen con su patrón de compra. Para simplificar, este ejemplo recomienda basándose en patrones de compra conjunta; los sistemas reales también tienen en cuenta la actualidad y las señales de "ya comprado" para evitar sugerir duplicados.

Primero, promediamos los vectores que representan el historial de compras anterior. Luego, comparamos ese promedio con todos los productos, excluyendo los comprados anteriormente, y proporcionamos una lista priorizada de recomendaciones:

```sql
WITH -- CTE to get the average embedding of the target products avg_product_embedding AS ( SELECT AVG(pe.embedding) AS avg_embedding FROM embeddings.product_embedding pe WHERE pe.product_id IN (9, 8, 6) ), -- CTE to get products with their embeddings prod_w_embedding AS ( SELECT p.id, p.label, pe.embedding FROM product AS p JOIN embeddings.product_embedding pe ON p.id = pe.product_id ) -- Final selection with similarity calculation SELECT pwe.id, pwe.label, 1 - (pwe.embedding <=> ape.avg_embedding) AS similarity FROM prod_w_embedding AS pwe, avg_product_embedding AS ape WHERE pwe.id NOT IN (9, 8, 6) -- Exclude the target products themselves ORDER BY pwe.embedding <=> ape.avg_embedding LIMIT 3;
```

```text
id | label | similarity ----+------------------------+-------------------- 7 | T-Shirt by Diesel | 0.7825087967377364 1 | Dress shirt by The Gap | 0.6423455730813185 2 | Dress shirt by Boss | 0.638560386165918
```

Esta consulta produce recomendaciones personalizadas que equilibran los intereses, tal como Spotify adapta las listas de reproducción a tu combinación única de gustos.

El cálculo del interés promedio podría ajustarse ponderándolo según la fecha y la cantidad de compras anteriores, o eliminando valores atípicos para evitar un punto de partida "a mitad de camino" y sin sentido.

---

### Resumen

Este capítulo proporcionó una guía completa para convertir PostgreSQL en una base de datos nativa de IA sólida. Comenzamos con el concepto fundamental de un vector como una "huella digital matemática" que permite a la IA comprender el significado y la similitud, utilizando una analogía simple de frutas (Plátano vs. Patata) y una consulta SQL básica con el operador de distancia `<->`. Luego explicamos por qué los índices son críticos para el rendimiento, contrastando un escaneo completo lento con dos tipos de índices importantes de `pgvector`: IVFFlat (el índice de supermercado) para búsquedas aproximadas a gran escala, y HNSW (el índice de mapa de ciudad), la opción rápida y precisa para aplicaciones en tiempo real. Aprendiste habilidades prácticas para crear y ajustar ambos índices ajustando sus "controles" o parámetros clave: `lists` y `probes` para IVFFlat, y `m` y `ef_construction` para HNSW. Finalmente, aplicamos estas habilidades para crear dos funciones del mundo real en un conjunto de datos de comercio electrónico: un motor de búsqueda semántica que encuentra productos por intención y dos tipos de motores de recomendación, incluido un modelo de preferencias de usuario al estilo Spotify utilizando `AVG()`.

`pgvector`, con su tipo de datos vectorial, índices y operaciones de búsqueda, proporciona una herramienta potente para extender el uso de nuestra base de datos relacional más allá de las transacciones y la analítica hacia aplicaciones de IA del mundo real. Los proyectos que anteriormente habrían requerido múltiples sistemas para manejar transacciones e IA ahora se pueden realizar usando solo PostgreSQL, beneficiándose de una operación unificada, una velocidad tremenda y una arquitectura sencilla que evita la codificación personalizada.

No necesitas una base de datos separada. No necesitas crear código personalizado para integrar los resultados de IA con los resultados de la base de datos transaccional: puedes hacerlo todo en PostgreSQL.

Habiendo dominado los fundamentos de cómo almacenar y consultar vectores, te mostraremos cómo desbloquear su verdadero potencial en el siguiente capítulo. Nos sumergiremos en el poder del razonamiento multimodal mediante el diseño de consultas de alto rendimiento que combinan la búsqueda vectorial de IA con filtros SQL tradicionales, demostrando cómo aprovechar las capacidades centrales de SQL para crear consultas de IA híbridas y eficientes en conjuntos de datos muy grandes, todo dentro de la misma plataforma de datos.
