# Base de Datos de Películas



## Integrantes

- Espinosa Nicolas
- Vallejos Emanuel
- Zanel Enzo

---

## Dataset utilizado

Archivo utilizado para el proyecto:

`movies_metadata.csv`

El dataset contiene información de películas como:

- título original
- fecha de estreno
- idioma original
- géneros
- colecciones
- países de producción
- productoras
- presupuesto
- ingresos
- duración
- popularidad
- calificaciones

Algunas columnas del CSV contienen estructuras similares a JSON, por ejemplo:

- `genres`
- `production_companies`
- `production_countries`
- `belongs_to_collection`
- `spoken_languages`

Dataset utilizado:

https://www.kaggle.com/datasets/sibamsamanta07/movies-dataset-45k-films-with-budget-and-revenue

---

## Tecnologías utilizadas

- MariaDB
- HeidiSQL
- SQL
- CSV

---

# Proceso de creación y normalización

## 1. Creación de la base de datos

La base se creó indicando explícitamente el juego de caracteres y la collation:

```sql
CREATE DATABASE peliculasdb
CHARACTER SET utf8mb4
COLLATE utf8mb4_general_ci;

```

### ¿Qué significa?

- `CREATE DATABASE peliculasdb`: crea la base de datos.
- `CHARACTER SET utf8mb4`: permite almacenar correctamente caracteres Unicode, por ejemplo tildes, ñ y caracteres de otros idiomas.
- `COLLATE utf8mb4_general_ci`: define cómo MariaDB compara y ordena los textos.

La collation se definió desde el inicio para evitar incompatibilidades al comparar textos extraídos con `JSON_TABLE`.

---

## 2. Tabla `crudo`

Primero se creó una tabla para recibir el CSV prácticamente sin transformación.

```sql
CREATE TABLE crudo (
    adult TEXT,
    belongs_to_collection TEXT,
    budget TEXT,
    genres TEXT,
    homepage TEXT,
    id TEXT,
    imdb_id TEXT,
    original_language TEXT,
    original_title TEXT,
    overview TEXT,
    popularity TEXT,
    poster_path TEXT,
    production_companies TEXT,
    production_countries TEXT,
    release_date TEXT,
    revenue TEXT,
    runtime TEXT,
    spoken_languages TEXT,
    estado TEXT,
    tagline TEXT,
    title TEXT,
    video TEXT,
    vote_average TEXT,
    vote_count TEXT
);
```

### ¿Por qué primero se usó `TEXT`?

En esta etapa no queríamos imponer todavía tipos de datos estrictos.

El CSV podía contener:

- valores vacíos,
- números almacenados como texto,
- fechas,
- estructuras similares a JSON,
- registros corruptos.

Por eso se utilizó `TEXT` como formato inicial. La idea era importar primero los datos y normalizarlos después.

Esta tabla funciona como una copia de trabajo del archivo original.

Cantidad importada:

**45.466 registros**

---

## 3. Importación del CSV

La carga se realizó con `LOAD DATA LOCAL INFILE`:

```sql
LOAD DATA LOCAL INFILE 'C:/Users/Lenovo v15/Downloads/movies_metadata.csv'
INTO TABLE crudo
CHARACTER SET utf8mb4
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
ESCAPED BY '^'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

### Sentencias importantes

- `FIELDS TERMINATED BY ','`: indica que las columnas están separadas por comas.
- `OPTIONALLY ENCLOSED BY '"'`: permite que una celda esté encerrada entre comillas dobles, algo necesario cuando el texto contiene comas.
- `ESCAPED BY '^'`: indica que `^` se utiliza como carácter de escape durante la importación. Se eligió para conservar correctamente las barras `\` presentes en algunos datos JSON.
- `IGNORE 1 ROWS`: ignora la primera fila del CSV porque contiene los encabezados.

---

## 4. Detección de registros corruptos

Se detectaron seis filas físicas correspondientes a tres registros corruptos del archivo original.

La consulta utilizada fue:

```sql
SELECT *
FROM crudo
WHERE vote_count IS NULL
   OR id NOT REGEXP '^[0-9]+$';
```

### ¿Qué hace `REGEXP`?

```sql
id REGEXP '^[0-9]+$'
```

comprueba que el contenido del campo `id` esté formado únicamente por números.

- `^`: comienzo del texto.
- `[0-9]`: cualquier número del 0 al 9.
- `+`: uno o más números.
- `$`: final del texto.

---

## 5. Creación de `crudo_valido`

En lugar de modificar la tabla `crudo`, se creó otra tabla para trabajar únicamente con los registros válidos.

Primero se copió la estructura:

```sql
CREATE TABLE crudo_valido LIKE crudo;
```

Después se eliminó el `id` original de esa tabla de trabajo:

```sql
ALTER TABLE crudo_valido
DROP COLUMN id;
```

Y se agregó un identificador propio:

```sql
ALTER TABLE crudo_valido
ADD id_crudo INT AUTO_INCREMENT PRIMARY KEY FIRST;
```

### ¿Qué significa?

- `CREATE TABLE ... LIKE`: copia la estructura de otra tabla.
- `ALTER TABLE`: modifica una tabla que ya existe.
- `AUTO_INCREMENT`: genera automáticamente un número correlativo para cada fila.
- `PRIMARY KEY`: identifica de forma única cada registro.
- `FIRST`: coloca visualmente esa columna al principio de la tabla.

Después se copiaron solamente las filas válidas:

```sql
INSERT INTO crudo_valido (
    adult,
    belongs_to_collection,
    budget,
    genres,
    homepage,
    imdb_id,
    original_language,
    original_title,
    overview,
    popularity,
    poster_path,
    production_companies,
    production_countries,
    release_date,
    revenue,
    runtime,
    spoken_languages,
    estado,
    tagline,
    title,
    video,
    vote_average,
    vote_count
)
SELECT
    adult,
    belongs_to_collection,
    budget,
    genres,
    homepage,
    imdb_id,
    original_language,
    original_title,
    overview,
    popularity,
    poster_path,
    production_companies,
    production_countries,
    release_date,
    revenue,
    runtime,
    spoken_languages,
    estado,
    tagline,
    title,
    video,
    vote_average,
    vote_count
FROM crudo
WHERE id REGEXP '^[0-9]+$'
  AND vote_count IS NOT NULL;
```

Cantidad resultante:

**45.460 registros válidos**

---

# Tablas normalizadas

## 6. Tabla `idiomas`

```sql
CREATE TABLE idiomas (
    id_idioma INT AUTO_INCREMENT PRIMARY KEY,
    cod_idioma CHAR(2) NOT NULL UNIQUE
);
```

Los idiomas se cargaron sin repetir valores:

```sql
INSERT INTO idiomas (cod_idioma)
SELECT DISTINCT original_language
FROM crudo_valido
WHERE original_language IS NOT NULL
ORDER BY original_language ASC;
```

### Sentencias importantes

- `DISTINCT`: elimina valores repetidos del resultado.
- `UNIQUE`: impide que se guarde dos veces el mismo código de idioma.
- `CHAR(2)`: se utiliza porque los códigos de idioma tienen dos caracteres.

Cantidad:

**89 idiomas**

---

## 7. Tabla `generos`

```sql
CREATE TABLE generos (
    id_genero INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL UNIQUE
);
```

Esta fue una de las primeras tablas en las que fue necesario leer información almacenada dentro de una estructura JSON.

### Ejemplo con `JSON_TABLE`

```sql
INSERT INTO generos (nombre)
SELECT DISTINCT datos.nombre
FROM crudo_valido,

JSON_TABLE(
    REPLACE(genres, '''', '"'),
    '$[*]' COLUMNS (
        nombre VARCHAR(50) PATH '$.name'
    )
) AS datos

ORDER BY datos.nombre ASC;
```

### ¿Cómo funciona?

El CSV puede contener una celda como:

```text
[{'id': 16, 'name': 'Animation'}, {'id': 35, 'name': 'Comedy'}]
```

Primero:

```sql
REPLACE(genres, '''', '"')
```

reemplaza las comillas simples por comillas dobles para obtener un JSON interpretable por MariaDB.

Después:

```sql
JSON_TABLE(...)
```

transforma los elementos del JSON en filas y columnas temporales.

En:

```sql
'$[*]'
```

- `$` representa la raíz del JSON.
- `[*]` significa todos los elementos del arreglo.

Y:

```sql
nombre VARCHAR(50) PATH '$.name'
```

crea una columna temporal llamada `nombre` tomando la propiedad `name` de cada objeto.

Finalmente:

```sql
AS datos
```

asigna el alias `datos` a esa tabla temporal, por eso podemos usar:

```sql
datos.nombre
```

Cantidad:

**20 géneros**

---

## 8. Tabla `paises_produccion`

```sql
CREATE TABLE paises_produccion (
    id_pais INT AUTO_INCREMENT PRIMARY KEY,
    cod_pais CHAR(2) NOT NULL UNIQUE,
    nombre VARCHAR(100) NOT NULL
);
```

El código del país se definió como `UNIQUE`, pero el nombre no, porque pueden existir códigos diferentes asociados a nombres iguales.

La carga también se realizó con `JSON_TABLE`, siguiendo la misma lógica explicada en la sección de géneros.

Cantidad:

**161 países**

---

## 9. Tabla `productoras`

```sql
CREATE TABLE productoras (
    id_productora INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    INDEX idx_productoras_nombre (nombre)
);
```

### ¿Por qué se creó un índice?

```sql
INDEX idx_productoras_nombre (nombre)
```

crea una estructura auxiliar de búsqueda sobre la columna `nombre`.

Esto permite que MariaDB encuentre una productora por nombre sin tener que recorrer toda la tabla repetidamente.

El índice fue especialmente importante cuando se creó la relación entre películas y productoras.

Cantidad:

**23.427 productoras**

---

## 10. Tabla `colecciones`

```sql
CREATE TABLE colecciones (
    id_coleccion INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL
);
```

La información de colección también se obtuvo desde JSON.

En este caso se trataba de un solo objeto y no de un arreglo, por lo que se utilizó:

```sql
'$'
```

en lugar de:

```sql
'$[*]'
```

Cantidad:

**1.695 colecciones**

---

## 11. Tabla principal `peliculas`

```sql
CREATE TABLE peliculas (
    id INT PRIMARY KEY,
    titulo_original VARCHAR(255),
    fecha_estreno DATE,
    id_idioma INT,
    id_coleccion INT,
    FOREIGN KEY (id_idioma) REFERENCES idiomas(id_idioma),
    FOREIGN KEY (id_coleccion) REFERENCES colecciones(id_coleccion)
);
```


Para cargar películas se utilizaron `LEFT JOIN` porque una película podía no tener colección o idioma relacionado.

Ejemplo:

```sql
LEFT JOIN idiomas AS i
    ON c.original_language = i.cod_idioma
```

`LEFT JOIN` mantiene la película aunque no encuentre una coincidencia; en ese caso la clave foránea puede quedar en `NULL`.

---

# Relaciones muchos-a-muchos

Las películas pueden tener varios géneros, varios países y varias productoras.

Al mismo tiempo, un género, un país o una productora pueden estar asociados a muchas películas.

Por eso estas relaciones son N:M y se resolvieron con tablas intermedias.

---

## 12. `peliculas_generos`

```sql
CREATE TABLE peliculas_generos (
    id_pelicula INT,
    id_genero INT,
    PRIMARY KEY (id_pelicula, id_genero),
    FOREIGN KEY (id_pelicula) REFERENCES peliculas(id),
    FOREIGN KEY (id_genero) REFERENCES generos(id_genero)
);
```

### Clave primaria compuesta

```sql
PRIMARY KEY (id_pelicula, id_genero)
```

significa que la combinación de ambos valores debe ser única.

Esto evita relacionar dos veces la misma película con el mismo género.

La carga se realizó nuevamente con `JSON_TABLE`.

Ejemplo del JOIN utilizado:

```sql
JOIN JSON_TABLE(
    REPLACE(c.genres, '''', '"'),
    '$[*]' COLUMNS (
        nombre VARCHAR(50) PATH '$.name'
    )
) AS datos
    ON TRUE
```

`ON TRUE` permite combinar cada película con todas las filas generadas por su JSON.

Cantidad:

**91.085 relaciones**

---

## 13. `peliculas_paises_produccion`

```sql
CREATE TABLE peliculas_paises_produccion (
    id_pelicula INT,
    id_pais INT,
    PRIMARY KEY (id_pelicula, id_pais),
    FOREIGN KEY (id_pelicula) REFERENCES peliculas(id),
    FOREIGN KEY (id_pais) REFERENCES paises_produccion(id_pais)
);
```

La relación se realizó mediante el código del país extraído del JSON.

Cantidad:

**49.423 relaciones**

---

## 14. `peliculas_productoras`

```sql
CREATE TABLE peliculas_productoras (
    id_pelicula INT,
    id_productora INT,
    PRIMARY KEY (id_pelicula, id_productora),
    FOREIGN KEY (id_pelicula) REFERENCES peliculas(id),
    FOREIGN KEY (id_productora) REFERENCES productoras(id_productora)
);
```

La lógica de carga es similar a las otras tablas intermedias, por lo que no se repite nuevamente toda la sentencia.

En este caso el índice:

```sql
idx_productoras_nombre
```

permitió acelerar el `JOIN` contra `productoras.nombre`.

Cantidad:

**70.535 relaciones**

---

# Resumen de tablas y registros

| Tabla | Descripción | Registros |
|---|---|---:|
| `crudo` | Datos importados directamente del CSV | 45.466 |
| `crudo_valido` | Registros válidos para normalización | 45.460 |
| `idiomas` | Idiomas originales | 89 |
| `generos` | Géneros cinematográficos | 20 |
| `paises_produccion` | Países de producción | 161 |
| `productoras` | Empresas productoras | 23.427 |
| `colecciones` | Colecciones | 1.695 |
| `peliculas` | Información principal de películas | 45.460 |
| `peliculas_generos` | Relación Película ↔ Género | 91.085 |
| `peliculas_paises_produccion` | Relación Película ↔ País | 49.423 |
| `peliculas_productoras` | Relación Película ↔ Productora | 70.535 |

---

# Vistas

Se crearon cuatro vistas para simplificar consultas que requieren unir varias tablas:

- `vista_peliculas_detalle`
- `vista_peliculas_generos`
- `vista_peliculas_paises`
- `vista_peliculas_productoras`

Ejemplo de creación de una vista:

```sql
CREATE VIEW vista_peliculas_detalle AS
SELECT
    p.id,
    p.titulo_original,
    p.fecha_estreno,
    i.cod_idioma AS idioma,
    c.nombre AS coleccion
FROM peliculas AS p

LEFT JOIN idiomas AS i
    ON p.id_idioma = i.id_idioma

LEFT JOIN colecciones AS c
    ON p.id_coleccion = c.id_coleccion;
```

Una vista guarda una consulta con un nombre y permite consultar información relacionada sin volver a escribir todos los `JOIN`.

Las demás vistas siguen la misma lógica, relacionando películas con géneros, países y productoras.

El orden se aplica al consultar la vista mediante `ORDER BY`, no dentro de su definición.

---

# Índices

Además de los índices generados automáticamente por `PRIMARY KEY` y `UNIQUE`, se agregaron índices alternativos.

## Productoras

```sql
INDEX idx_productoras_nombre (nombre)
```

Acelera búsquedas y relaciones por nombre de productora.

## Fecha de estreno

```sql
CREATE INDEX idx_peliculas_fecha_estreno
ON peliculas (fecha_estreno);
```

Ayuda en consultas por fecha o rango de fechas:

```sql
SELECT *
FROM peliculas
WHERE fecha_estreno >= '2000-01-01';
```

## Título original

```sql
CREATE INDEX idx_peliculas_titulo
ON peliculas (titulo_original);
```

Ayuda en búsquedas por título:

```sql
SELECT *
FROM peliculas
WHERE titulo_original LIKE 'Toy%';
```

MariaDB decide automáticamente cuándo utilizar un índice.

Se puede comprobar el plan de una consulta utilizando:

```sql
EXPLAIN
SELECT *
FROM peliculas
WHERE fecha_estreno >= '2000-01-01';
```

---

# Estructura de relaciones

```text
peliculas
   |
   |---- idiomas
   |
   |---- colecciones
   |
   |---- peliculas_generos -------- generos
   |
   |---- peliculas_paises_produccion ---- paises_produccion
   |
   |---- peliculas_productoras ---- productoras
```

---

# Backup de la base de datos

El backup completo se encuentra comprimido en:

`backup_peliculasdb.zip`

Dentro del ZIP se encuentra:

`backup_peliculasdb.sql`

El backup contiene la estructura y los datos de la base.

## Restauración con HeidiSQL

1. Descargar `backup_peliculasdb.zip`.
2. Extraer el archivo.
3. Abrir HeidiSQL.
4. Conectarse al servidor MariaDB.
5. Abrir `backup_peliculasdb.sql`.
6. Ejecutar el script completo.
7. Actualizar el panel de bases de datos.

El backup contiene la creación y selección de `peliculasdb`, por lo que no es necesario crearla manualmente antes de ejecutar el script.

---

# Archivos del repositorio

```text
peliculas/
│
├── movies_metadata.csv
├── backup_peliculasdb.zip
├── diagrama.jpeg
└── README.md
```

---

# Resultado final

La base quedó normalizada en tablas relacionadas, evitando mantener listas o estructuras complejas dentro de las tablas finales.

La tabla `crudo` conserva los datos importados desde el CSV y `crudo_valido` funciona como fuente limpia para el proceso de normalización.

La estructura final permite consultar películas junto con sus idiomas, colecciones, géneros, países de producción y productoras mediante relaciones SQL.
