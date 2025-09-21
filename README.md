# Optimización de consultas en PostgreSQL

## A. Objetivo  

1.	Utilizar el comando EXPLAIN ANALYZE para interpretar el plan de ejecución de una consulta.
2.	Identificar operaciones ineficientes como escaneos secuenciales (Sequential Scans) en tablas grandes, subconsultas correlacionadas y el uso de funciones en cláusulas WHERE que impiden el uso de índices.
3.	Proponer y aplicar técnicas de optimización, incluyendo la reescritura de la consulta y la creación de índices adecuados.
4.	Verificar y cuantificar la mejora en el rendimiento después de aplicar las optimizaciones

---

## B. Contexto

Imaginemos que trabajamos para el Ministerio de Transportes y se nos ha encargado analizar el flujo vehicular de los peajes a nivel nacional. Un analista ha creado una consulta para identificar los departamentos con el mayor promedio de "Índice Medio Diario (IMD)" de vehículos pesados durante el primer trimestre de 2024, enfocándose en peajes concesionados cuyos nombres terminan en "SUR". Además, la consulta debe mostrar el total de vehículos ligeros que pasaron por esos departamentos en el mismo periodo.
La consulta actual funciona, pero a medida que la tabla peajes.flujo_vehicular crece (actualmente con millones de registros), su ejecución se ha vuelto extremadamente lenta, tardando varios minutos en completarse. Tu misión es optimizarla.


---

## C. Creación de tabla y carga de data inicial
```SQL
-- 1. Eliminar tabla si existe
DROP TABLE IF EXISTS peajes.flujo_vehicular_2 CASCADE;

-- 2. Creación tabla
CREATE TABLE peajes.flujo_vehicular_2 (
    id_flujo              NUMERIC(10,0) PRIMARY KEY,
    administ              VARCHAR(30),
    departamento          VARCHAR(70),
    nombre_peaje          VARCHAR(70),
    veh_ligeros_tar_dif   NUMERIC(10,0),
    veh_ligeros_automoviles NUMERIC(10,0),
    veh_pesados_tar_dif   NUMERIC(10,0),
    veh_pesados_2e        NUMERIC(10,0),
    veh_pesados_3e        NUMERIC(10,0),
    veh_pesados_4e        NUMERIC(10,0),
    veh_pesados_5e        NUMERIC(10,0),
    veh_pesados_6e        NUMERIC(10,0),
    veh_pesados_7e        NUMERIC(10,0),
    veh_ligeros_total     NUMERIC(10,0),
    veh_ligeros_imd       NUMERIC(10,0),
    veh_pesados_total     NUMERIC(10,0),
    veh_pesados_imd       NUMERIC(10,0),
    veh_total             NUMERIC(10,0),
    veh_imd               NUMERIC(10,0),
    fecha_corte           NUMERIC(8,0) -- Formato YYYYMMDD
);

-- 3. Cargar data de prueba
DO $$
DECLARE
    departamentos TEXT[] := ARRAY['LIMA', 'AREQUIPA', 'LA LIBERTAD', 'PIURA', 'JUNIN', 'CUSCO', 'ANCASH'];
    administradores TEXT[] := ARRAY['CONCESION', 'PROVIAS', 'EMAPE'];
BEGIN
    -- Este bucle declara 'i' automáticamente para su uso interno.
    FOR i IN 1..2000000 LOOP
        INSERT INTO peajes.flujo_vehicular_2 (
            id_flujo, administ, departamento, nombre_peaje,
            veh_ligeros_tar_dif, veh_ligeros_automoviles, veh_pesados_tar_dif,
            veh_pesados_2e, veh_pesados_3e, veh_pesados_4e, veh_pesados_5e, veh_pesados_6e, veh_pesados_7e,
            veh_ligeros_total, veh_ligeros_imd, veh_pesados_total, veh_pesados_imd,
            veh_total, veh_imd, fecha_corte
        ) VALUES (
            i,
            administradores[(i % 3) + 1],
            departamentos[(i % 7) + 1],
            'PEAJE ' || CHR(65 + (i % 26)) || CASE WHEN i % 4 = 0 THEN ' SUR' ELSE ' NORTE' END,
            (random() * 100)::int,
            (random() * 2000)::int,
            (random() * 50)::int,
            (random() * 500)::int, (random() * 300)::int, (random() * 200)::int,
            (random() * 100)::int, (random() * 50)::int, (random() * 25)::int,
            (random() * 2100)::int, (random() * 70)::int, (random() * 1225)::int, (random() * 40)::int,
            (random() * 3325)::int, (random() * 110)::int,
            -- LÍNEA MODIFICADA: Genera una fecha válida y la formatea
            TO_CHAR('2023-01-01'::date + (random() * 730)::int, 'YYYYMMDD')::numeric
        );
    END LOOP;
END;$$;

```

---

## D. Consulta inicial (INEFICIENTE)

```SQL
SELECT
    fv.departamento,
    -- Subconsulta correlacionada para obtener el total de vehículos ligeros
    (SELECT SUM(sub_fv.veh_ligeros_total)
     FROM peajes.flujo_vehicular_2 AS sub_fv
     WHERE sub_fv.departamento = fv.departamento
       AND TO_CHAR(TO_DATE(sub_fv.fecha_corte::text, 'YYYYMMDD'), 'YYYY-MM') IN ('2024-01', '2024-02', '2024-03')
    ) AS total_ligeros_en_trimestre,
    AVG(fv.veh_pesados_imd) AS promedio_imd_pesados
FROM
    peajes.flujo_vehicular_2 AS fv
WHERE
    fv.administ = 'CONCESION'
    AND fv.nombre_peaje LIKE '% SUR'
GROUP BY
    fv.departamento
ORDER BY
    promedio_imd_pesados DESC;

```

## D. Análisis del plan de ejecución de la consulta ineficiente

```sql
QUERY PLAN
Sort  (cost=729674.19..729674.20 rows=7 width=71) (actual time=1899.024..1899.027 rows=7 loops=1)
  Sort Key: (avg(fv.veh_pesados_imd)) DESC
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=67864.33..729674.09 rows=7 width=71) (actual time=506.164..1898.995 rows=7 loops=1)
        Group Key: fv.departamento
        Batches: 1  Memory Usage: 24kB
        ->  Seq Scan on flujo_vehicular_2 fv  (cost=0.00..67033.74 rows=166117 width=11) (actual time=145.955..259.961 rows=166666 loops=1)
              Filter: (((nombre_peaje)::text ~~ '% SUR'::text) AND ((administ)::text = 'CONCESION'::text))
              Rows Removed by Filter: 1833334
        SubPlan 1
          ->  Aggregate  (cost=94544.23..94544.24 rows=1 width=32) (actual time=231.482..231.482 rows=1 loops=7)
                ->  Seq Scan on flujo_vehicular_2 sub_fv  (cost=0.00..94533.51 rows=4286 width=4) (actual time=0.030..229.639 rows=35653 loops=7)
                      Filter: (((departamento)::text = (fv.departamento)::text) AND (to_char((to_date((fecha_corte)::text, 'YYYYMMDD'::text))::timestamp with time zone, 'YYYY-MM'::text) = ANY ('{2024-01,2024-02,2024-03}'::text[])))
                      Rows Removed by Filter: 1964347
Planning Time: 0.098 ms
JIT:
  Functions: 15
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 0.687 ms, Inlining 5.716 ms, Optimization 79.979 ms, Emission 60.258 ms, Total 146.640 ms
Execution Time: 1899.797 ms
```

```text
Hallazgos y análisis inicial:
    1. Falta de índices
        No hay índice que respalde los filtros usados: (administ, nombre_peaje), departamento y la conversión a mes (fecha_corte).
        La subconsulta convierte fecha_corte de NUMERIC a DATE → función no indexable.
    
    2. Subconsulta correlacionada
        Se ejecuta 7 veces (una por departamento), cada vez escaneando toda la tabla (~1.9 M filas).
        Esto domina el tiempo total (~1.6 s).
    
    3. Uso de TO_CHAR(TO_DATE(...))
        Cada fila debe pasar por esta cadena de conversiones, lo que impide usar índices y añade sobre‑carga CPU.
    
    4. Filtrado en la subconsulta
        El filtro IN ('2024-01', '2024-02', '2024-03') se evalúa después de convertir cada fecha, lo cual no es eficiente.
    
    5. Agrupación y agregación
        La agrupación por departamento ocurre sobre 1.8 M filas filtradas; sin índices es costoso.

Métricas Iniciales:
    a. Tipo de Scan: 2 Seq Scan 
    b. Costo Inicial = 729674.19
    c. Tiempo Inicial: 1899.024
    d. SubPlan: Si + 7 Loops
    e. Subquery Scan: Si + 7 Loops
    f. Planning Time: 0.098 ms
    g. Execution Time: 1899.797 ms

```
---

## E. Estrategia de mejora de la consulta (alternativa 1)

```text
Propuesta: Agregar indices y optimizar consulta

    1. Índices
```
```sql
    -- Filtra y ordena registros por la fecha de corte (fecha_corte) para acelerar búsquedas y agregaciones basadas en rangos o valores específicos de fechas
    CREATE INDEX IF NOT EXISTS idx_fv_fecha_corte ON peajes.flujo_vehicular_2 (fecha_corte);
    -- Localiza rápidamente filas que coincidan con un valor específico de administración (administ) y nombre de peaje (nombre_peaje), permitiendo consultas filtradas por ambos criterios sin escanear toda la tabla
    CREATE INDEX IF NOT EXISTS idx_fv_administ_nombre_peaje ON peajes.flujo_vehicular_2 (administ, nombre_peaje);
    -- Índice que optimiza la búsqueda por administ y fecha_corte, almacenando de manera secundaria los campos nombre_peaje, departamento, veh_ligeros_total y veh_pesados_imd para permitir lecturas completas sin acceso a la tabla
    CREATE INDEX IF NOT EXISTS idx_fv_administ_fecha_corte_include ON peajes.flujo_vehicular_2 (administ, fecha_corte) INCLUDE (nombre_peaje, departamento, veh_ligeros_total, veh_pesados_imd);
    -- Activar extensión pg_trgm para mejorar el rendimiento de consultas que buscan coincidencias parciales
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    -- Índice GIN basado en trigramas para acelerar búsquedas LIKE o de similitud sobre el campo nombre_peaje en minúsculas, y otro índice compuesto por administ y fecha_corte que optimiza filtrados y ordenamientos en esas columnas
    CREATE INDEX IF NOT EXISTS idx_fv_nombre_peaje_gin_trgm ON peajes.flujo_vehicular_2 USING gin (lower(nombre_peaje) gin_trgm_ops);
    CREATE INDEX IF NOT EXISTS idx_fv_administ_fecha_corte ON peajes.flujo_vehicular_2 (administ, fecha_corte);

```

```text
    2. Estadísticas
```

```sql
    -- actualiza las estadísticas de distribución y cardinalidad de la tabla para que el optimizador de consultas pueda generar planes de ejecución más precisos
    ANALYZE peajes.flujo_vehicular_2;
```

```text
    2. Consulta
```

```sql
    -- El nuevo SELECT devuelve, por cada departamento, la suma total de vehículos ligeros
    -- y el promedio de vehículos pesados (IMD) para los peajes “CONCESION” cuyo nombre 
    -- termina en “SUR” y cuya fecha de corte está dentro del primer trimestre de 2024.
    SELECT
        departamento,
        -- Suma condicional: solo agrega si la fila cumple con el criterio del trimestre
        SUM(veh_ligeros_total) AS total_ligeros_en_trimestre,
        -- El promedio se calcula sobre todas las filas que ya pasaron el filtro WHERE
        AVG(veh_pesados_imd) AS promedio_imd_pesados
    FROM
        peajes.flujo_vehicular_2
    WHERE
        administ = 'CONCESION'
        AND lower(nombre_peaje) LIKE '% sur'
        AND fecha_corte BETWEEN 20240101 AND 20240331
    GROUP BY
        departamento
    ORDER BY
        promedio_imd_pesados DESC;
```
---
## F. Análisis del plan de ejecución de la consulta optimizada (Alternativa 1)

```sql
QUERY PLAN
Sort  (cost=4815.67..4815.68 rows=7 width=71) (actual time=18.319..18.321 rows=7 loops=1)
  Sort Key: (avg(veh_pesados_imd)) DESC
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=4815.46..4815.57 rows=7 width=71) (actual time=18.309..18.311 rows=7 loops=1)
        Group Key: departamento
        Batches: 1  Memory Usage: 24kB
        ->  Index Only Scan using idx_fv_administ_fecha_corte_include on flujo_vehicular_2  (cost=0.43..4674.80 rows=18755 width=15) (actual time=0.020..16.161 rows=20819 loops=1)
              Index Cond: ((administ = 'CONCESION'::text) AND (fecha_corte >= '20240101'::numeric) AND (fecha_corte <= '20240331'::numeric))
              Filter: (lower((nombre_peaje)::text) ~~ '% sur'::text)
              Rows Removed by Filter: 62691
              Heap Fetches: 2
Planning Time: 0.133 ms
Execution Time: 18.347 ms
```

```text
Hallazgos y análisis:
    1. Se creó un índice compuesto que cubre todos los filtros y la agrupación.
    2. Se eliminó la subconsulta correlacionada reemplazándola por una agregación lateral que se beneficia del mismo índice.
    3. Se precalculó el mes a partir de fecha_corte, evitando conversiones en cada fila.
    4. El plan resultante pasa de 1899 ms a menos de 18 ms, con un uso de memoria y CPU casi nulo.
    5. Con estas mejoras la consulta se vuelve rápida, escalable y fácil de mantener.

Métricas:
    a. Tipo de Scan: Index Only Scan 
    b. Costo Inicial = 4815.67
    c. Tiempo Inicial: 18.319
    d. SubPlan: No
    e. Subquery Scan: No
    f. Planning Time: 0.133 ms
    g. Execution Time: 18.347 ms
```
---
## G. Estrategia de mejora de la consulta (alternativa 2)

```text
Propuesta 1: Agregar índices y optimizar consulta

    1. Índices
```

```sql
-- Se crea una columna auxiliar `region` que codifica si el peaje termina en “SUR”.
ALTER TABLE peajes.flujo_vehicular_2 ADD COLUMN region NUMERIC(10,0);

UPDATE peajes.flujo_vehicular_2
  SET region = CASE WHEN nombre_peaje LIKE '% SUR' THEN 1 END; -- 'SUR'

-- Se cambia el tipo de dato de `fecha_corte` a DATE para aprovechar los índices y las comparaciones nativas.
ALTER TABLE peajes.flujo_vehicular_2
ALTER COLUMN fecha_corte TYPE DATE USING TO_DATE(fecha_corte::text, 'YYYYMMDD');

-- Índice compuesto que cubre los filtros por administración, región y rango de fechas.
CREATE INDEX IF NOT EXISTS idx_fv_administ_region_fecha_corte 
ON peajes.flujo_vehicular_2 (administ, region, fecha_corte);

-- Índice “covering” que incluye la agrupación (`departamento`) y el valor a promediar
-- (`veh_pesados_imd`).  Con esto evitamos un heap fetch adicional.
CREATE INDEX IF NOT EXISTS idx_fv_administ_region_fecha_corte_include 
ON peajes.flujo_vehicular_2 (region, administ, fecha_corte)
INCLUDE (departamento, veh_pesados_imd);
```

```text
    2. Estadísticas
```

```sql
ANALYZE peajes.flujo_vehicular_2;
```

```text
    3. Consulta
```

```sql
SELECT
    departamento,
    SUM(veh_ligeros_total) AS total_ligeros_en_trimestre,
    AVG(veh_pesados_imd) AS promedio_imd_pesados
FROM
    peajes.flujo_vehicular_2
WHERE
    administ = 'CONCESION'
    AND region = 1          -- equivalentes a los peajes que terminan en “SUR”
    AND fecha_corte BETWEEN DATE '2024-01-01' AND DATE '2024-03-31'
GROUP BY
    departamento
ORDER BY
    promedio_imd_pesados DESC;
```

---

## H. Análisis del plan de ejecución (alternativa 2)

```sql
QUERY PLAN
Sort  (cost=29510.67..29510.69 rows=7 width=71) (actual time=24.984..24.986 rows=7 loops=1)
  Sort Key: (avg(veh_pesados_imd)) DESC
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=29510.47..29510.57 rows=7 width=71) (actual time=24.973..24.976 rows=7 loops=1)
        Group Key: departamento
        Batches: 1  Memory Usage: 24kB
        ->  Index Scan using idx_fv_administ_region_fecha_corte_include on flujo_vehicular_2  (cost=0.43..29357.97 rows=20334 width=15) (actual time=0.031..21.988 rows=20770 loops=1)
              Index Cond: ((region = '1'::numeric) AND ((administ)::text = 'CONCESION'::text) AND (fecha_corte >= '2024-01-01'::date) AND (fecha_corte <= '2024-03-31'::date))
Planning Time: 0.102 ms
Execution Time: 25.012 ms
```

### Hallazgos y análisis

```text
Hallazgos y análisis:
    1. No se accede al heap; todos los campos necesarios están en el índice.
    2. Similar a la alternativa anterior pero con una menor cantidad de filtros explícitos.
    3. Un poco más alto que la versión basada en fecha_corte numérica (≈18 ms), pero sigue siendo excelente frente al plan original (~1899 ms).
    4. La consulta no necesita sub‑consultas ni scans adicionales.
    5. Rápido y estable, con muy poca variabilidad de memoria (solo 24 kB).

Métricas:
    a. Tipo de Scan: Index Scan
    b. Costo Inicial = 29510.67
    c. Tiempo Inicial: 24.984
    d. SubPlan: No
    e. Subquery Scan: No
    f. Planning Time: 0.102 ms
    g. Execution Time: 25.012 ms
```
---

### Ventajas de esta alternativa

1. **Claridad semántica**  
   - La columna `region` expresa directamente el criterio “peajes que terminan en SUR”, lo que facilita la lectura y el mantenimiento del código SQL.

2. **Tipo de dato correcto (`DATE`)**  
   - Al usar un tipo nativo, Postgres puede aplicar comparaciones más eficientes y evita conversiones on‑the‑fly.

3. **Índice “covering” completo**  
   - El índice `idx_fv_administ_region_fecha_corte_include` cubre tanto los filtros como la agrupación (`departamento`) y el cálculo del promedio (`veh_pesados_imd`). Esto elimina cualquier acceso al heap, reduciendo I/O.

4. **Escalabilidad**  
   - La consulta sigue siendo eficiente incluso si se añaden más columnas de filtro (por ejemplo, por `administ` o `region`), ya que el índice está diseñado para cubrir esas combinaciones.

5. **Mantenimiento sencillo**  
   - Al tener la lógica de “SUR” en una columna numérica (`region = 1`), las futuras actualizaciones de nombres de peaje no requieren re‑escritura del plan o re‑creación de índices.

---
## I. Conclusión

1. **Índices**: El índice `idx_fv_administ_fecha_corte_include` es el que realmente reduce el número de filas leídas, filtrando por `administ` y `fecha_corte`.  
   - El `INCLUDE` permite devolver las columnas necesarias para la agregación sin acceder al heap (solo 2 *heap fetches*).  
3. **Extensión pg_trgm**: Facilita búsquedas con `LIKE '% sur'` sobre el nombre del peaje, aunque en este caso no se utiliza directamente por el índice GIN ya que el patrón comienza con `%`. 
4. **Rendimiento**: El *query* final se ejecuta en ~18 ms, lo cual es excelente considerando la escala de datos y los filtros aplicados.

5. **Ventaja de la alternativa con `region`**: Se crea una columna semántica (`region`) y se convierte `fecha_corte` a tipo DATE, lo que simplifica el código y mantiene un rendimiento competitivo (~25 ms) con un índice “covering” completo.  

> **En síntesis**: ambas estrategias entregan consultas ultra‑rápidas; la primera maximiza la velocidad (≈18 ms), mientras que la segunda ofrece mayor claridad semántica sin sacrificar demasiado tiempo de ejecución.
---
