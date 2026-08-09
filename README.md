# La presencia latinoamericana en el cine español (2021–2025)

Proyecto de data engineering y análisis que estudia la participación de países latinoamericanos en la producción del "cine español" (según la metodología ICAA/Rentrak) entre 2021 y 2025.

> **Versión mejorada de un proyecto anterior.** Este repositorio es una reconstrucción completa del proyecto original (entregado como trabajo final del bootcamp de IT Academy). Se ha rehecho el pipeline de extracción y enriquecimiento, se ha sustituido el análisis en pandas/CSV por un esquema relacional real en MySQL con `FOREIGN KEY`, y se han corregido varias decisiones metodológicas de la primera versión (ver la sección [Diferencias con la versión anterior](#diferencias-con-la-versión-anterior)).

## Aviso metodológico importante

Los datos de origen (ICAA/Rentrak) **no representan la cartelera global de todas las películas exhibidas en España**. Cubren únicamente películas con participación española en la producción — lo que la propia fuente define como "cine español". Todas las tablas derivadas de esta fuente (`estrenos_esp`, `icaa_raw`, `icaa_peliculas` y el resto del esquema final) deben interpretarse en esos términos: el foco analítico es la participación de países latinoamericanos en la producción dentro de ese corpus, no la exhibición comercial en salas españolas en general.

Un subconjunto de películas latinoamericanas sin coproducción española (identificadas vía IMDb) se trata como caso aparte, fuera del alcance del corpus principal de coproducciones.

## Pipeline

El proyecto está organizado en seis notebooks numerados, cada uno con una responsabilidad única:

| Notebook | Descripción |
|---|---|
| [`0_extraccion_pdfs.ipynb`](0_extraccion_pdfs.ipynb) | Lee los PDFs de taquilla y espectadores del ICAA y construye `icaa_raw` (una fila por película y año en cartelera). No resuelve `icaa_id` ni toca el catálogo ICAA. |
| [`1_icaa_id_y_enriquecimiento.ipynb`](1_icaa_id_y_enriquecimiento.ipynb) | Resuelve `icaa_id` por título contra el catálogo ICAA y scrapea la ficha de cada película: distribuidora real, género, nacionalidad, subvenciones, dirección y guion. Construye `icaa_peliculas`. |
| [`2_enriquecimiento_tmdb.ipynb`](2_enriquecimiento_tmdb.ipynb) | Enriquece a las personas (dirección y guion) con sexo y país. Fuentes en cascada: TMDB (principal) → Wikidata (secundaria, vía `imdb_id`) → IMDb directo (terciaria, descartada por bloqueo anti-bot). |
| [`3_distribuidoras.ipynb`](3_distribuidoras.ipynb) | Normaliza distribuidoras como dimensión propia, deduplicando por identidad real de empresa (no por cadena exacta) a partir de las dos fuentes crudas disponibles. |
| [`4_tablas_finales.ipynb`](4_tablas_finales.ipynb) | Construye el esquema final: 16 tablas en MySQL (`icaa_cine`) con `FOREIGN KEY` reales, sin prefijos `dim_`/`fact_`. |
| [`5_analisis_latam.ipynb`](5_analisis_latam.ipynb) | Análisis exploratorio mediante consultas SQL directas contra MySQL (no pandas/CSV): alcance del corpus, evolución temporal, distribuidoras, géneros, subvenciones, dirección/guion y países coproductores. |

### Jerarquía de fuentes

Los datos se priorizan de forma jerárquica y explícita en todo el pipeline:

1. **ICAA** — fuente primaria y autoritativa (ficha oficial de cada película)
2. **TMDB** — fuente secundaria, usada para enriquecimiento de personas y para las películas sin `icaa_id`
3. **Wikidata** — fuente terciaria, solo como respaldo para huecos de TMDB

## Esquema final

Base de datos MySQL `latam_cine_esp`, 16 tablas con claves foráneas reales:

`peliculas` · `generos` · `paises` · `participacion_pais` · `direccion` · `guion` · `direccion_peliculas` · `guion_peliculas` · `taquilla_cine_esp` · `subvenciones` · `distribuidoras` · `peliculas_distribuidoras` · `peliculas_generos` · `peliculas_generos_padre` · `empresas_productoras` · `peliculas_empresas_productoras`

Puntos clave del diseño:
- `participacion_pais.porcentaje` viene siempre de ICAA — no requiere imputación.
- `peliculas_generos_padre` ofrece una versión de género de etiqueta única por película (agrupación temática), como alternativa a `peliculas_generos`, que es multivalor.
- `distribuidoras` incluye una entrada `"Independiente"` que representa películas sin distribuidor identificado (no una empresa real) — se excluye sistemáticamente de cualquier ranking de distribuidoras.

## Análisis exploratorio

`5_analisis_latam.ipynb` consulta directamente MySQL vía SQL y cubre:

1. Configuración y conexión
2. Alcance del corpus frente al total de "cine español"
3. Evolución temporal (2021–2025)
4. Películas destacadas (años en cartelera, espectadores, recaudación)
5. Distribuidoras
6. Géneros
7. Subvenciones
8. Guion y dirección
9. Países coproductores
10. Empresas productoras

## Diferencias con la versión anterior

Este proyecto documenta abiertamente en qué se aparta de la primera versión (bootcamp) y por qué:

- **Idiomas**: sección eliminada — el cruce por título con TMDB no era suficientemente fiable.
- **Rating TMDB**: sustituido por dos proxies construidos desde datos propios del corpus — subvención media por género (proxy de reconocimiento institucional) y años medios en cartelera (proxy de interés sostenido en el tiempo) — ya que un rating de audiencia global no estaba disponible ni era consistente con el resto del pipeline.
- **Género**: pasa a ser de etiqueta única por película (`peliculas_generos_padre`) para el análisis agregado, en lugar de tratarse como multivalor sin orden fiable.
- **Almacenamiento y consulta**: el análisis se ejecuta con SQL sobre un esquema relacional real en MySQL, no sobre CSV cargados en pandas.
- **Distribuidoras**: se separan en una dimensión propia con normalización de identidad de empresa, en vez de quedar embebidas como texto en la tabla de películas.

Otras limitaciones metodológicas quedan documentadas explícitamente donde aplican (deduplicación de `icaa_id`, cobertura de subvenciones, fiabilidad del matching por nombre en Wikidata, bloqueo anti-bot de IMDb, etc.), siguiendo el principio del proyecto de no ocultar los huecos de los datos.

## Stack técnico

- **Lenguaje**: Python 3.14, notebooks Jupyter (VSCode)
- **Base de datos**: MySQL (`latam_cine_esp`), gestionada con MySQL Workbench
- **Extracción y scraping**: `pdfplumber`, `requests`, `beautifulsoup4`, `selenium` (Firefox, vía `webdriver_manager`)
- **Datos y persistencia**: `pandas`, `sqlalchemy` + `pymysql`
- **Fuentes externas**: catálogo web ICAA (`infoicaa.mcu.es`, `sede.mcu.gob.es`), API de TMDB, SPARQL de Wikidata
- **Visualización**: `matplotlib`, `seaborn`

## Estructura del repositorio

```
.
├── 0_extraccion_pdfs.ipynb
├── 1_icaa_id_y_enriquecimiento.ipynb
├── 2_enriquecimiento_tmdb.ipynb
├── 3_distribuidoras.ipynb
├── 4_tablas_finales.ipynb
├── 5_analisis_latam.ipynb
└── README.md
```

## Cómo ejecutar

1. Clona el repositorio e instala las dependencias (ver `requirements.txt`).
2. Crea un archivo `.env` con las credenciales de MySQL y las claves de API necesarias (TMDB, etc.).
3. Ejecuta los notebooks en orden, del `0` al `5`. Cada uno consume los CSV generados por el anterior; el notebook `4` es el único que escribe en MySQL.
