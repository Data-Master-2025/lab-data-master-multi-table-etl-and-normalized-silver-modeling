![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab Complejo | ETL Multi-tabla y Modelado Silver Normalizado

## 🎯 Objetivo

Ejecutar un flujo ETL avanzado desde Bronze a Silver usando un dataset con múltiples columnas, referencias y categorías. Se implementará un modelo Silver normalizado dividido en tablas referenciales y de hechos, siguiendo las normas SEAT.

## Requisitos

* Haz un ***fork*** de este repositorio.
* Clona este repositorio.

## Entrega

- Haz Commit y Push
- Crea un Pull Request (PR)
- Copia el enlace a tu PR (con tu solución) y pégalo en el campo de entrega del portal del estudiante – solo así se considerará entregado el lab

## 📁 Dataset: B_ORDERS_RAW_ADVANCED.csv

**Ubicación Bronze:** `DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_ADVANCED`

## 🛢️ 1. Limpieza y transformación desde Bronze

### Tabla de hechos Silver: `S_ORDERS_FACT`

```sql
CREATE OR REPLACE TABLE DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_FACT AS
SELECT
  ID_ORDER,
  TRY_TO_DATE(DTE_ORDER, 'YYYY-MM-DD') AS DTE_ORDER,
  ID_CUSTOMER,
  ID_PART,
  TRY_TO_NUMBER(QTY_ORDERED) AS QTY_ORDERED,
  TRY_TO_NUMBER(AMT_TOTAL) AS AMT_TOTAL,
  REF_PAYMENT_METHOD,
  TRY_TO_DATE(DTE_DELIVERY_EST, 'YYYY-MM-DD') AS DTE_DELIVERY_EST,
  REF_COUNTRY_CODE,
  REF_DISCOUNT_CODE,
  TRY_TO_NUMBER(QTY_BACKORDER) AS QTY_BACKORDER,
  TRY_TO_NUMBER(RAT_DISCOUNT_APPLIED) AS RAT_DISCOUNT_APPLIED,
  TRY_TO_TIMESTAMP(TST_INSERTION) AS TST_INSERTION,
  AUD_USR_INSERT
FROM DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_ADVANCED
WHERE 
  IS_NUMBER(QTY_ORDERED) AND TRY_TO_NUMBER(QTY_ORDERED) > 0
  AND IS_NUMBER(AMT_TOTAL) AND TRY_TO_NUMBER(AMT_TOTAL) > 0
  AND ID_CUSTOMER IS NOT NULL AND ID_PART != 'ERROR';
```

## 📊 2. Tablas de dimensiones

### `S_DIM_ORDER_CHANNEL`

```sql
CREATE OR REPLACE TABLE DEV_SILVER_DB.S_SUPPLY_CHAIN.S_DIM_ORDER_CHANNEL AS
SELECT DISTINCT NULLIF(CAT_ORDER_CHANNEL, '') AS CAT_ORDER_CHANNEL
FROM DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_ADVANCED
WHERE CAT_ORDER_CHANNEL IS NOT NULL;
```

### `S_DIM_CUSTOMER_SEGMENT`

```sql
CREATE OR REPLACE TABLE DEV_SILVER_DB.S_SUPPLY_CHAIN.S_DIM_CUSTOMER_SEGMENT AS
SELECT DISTINCT NULLIF(TYP_CUSTOMER_SEGMENT, '') AS TYP_CUSTOMER_SEGMENT
FROM DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_ADVANCED
WHERE TYP_CUSTOMER_SEGMENT IS NOT NULL;
```

## 🧩 3. Enriquecimiento y Join en una vista

```sql
CREATE OR REPLACE VIEW DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_ENRICHED AS
SELECT
  F.*,
  O.CAT_ORDER_CHANNEL,
  C.TYP_CUSTOMER_SEGMENT,
  B.DES_ORDER_NOTE
FROM DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_FACT F
LEFT JOIN DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_ADVANCED B ON F.ID_ORDER = B.ID_ORDER
LEFT JOIN DEV_SILVER_DB.S_SUPPLY_CHAIN.S_DIM_ORDER_CHANNEL O ON B.CAT_ORDER_CHANNEL = O.CAT_ORDER_CHANNEL
LEFT JOIN DEV_SILVER_DB.S_SUPPLY_CHAIN.S_DIM_CUSTOMER_SEGMENT C ON B.TYP_CUSTOMER_SEGMENT = C.TYP_CUSTOMER_SEGMENT;
```

## 🚀 4. Carga incremental

```sql
INSERT INTO DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_FACT
SELECT * FROM (
  -- mismo SELECT que arriba
) WHERE TRY_TO_TIMESTAMP(TST_INSERTION) > (
  SELECT MAX(TST_INSERTION) FROM DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_FACT
);
```

## 🧱 5. Modelado en SqlDBM

### Tablas Silver:

- `S_ORDERS_FACT`
- `S_DIM_ORDER_CHANNEL`
- `S_DIM_CUSTOMER_SEGMENT`

### Recomendaciones:

- Prefijo `S_` obligatorio
- Clave primaria `ID_ORDER` en fact table
- Claves foráneas en dimensiones
- Nombres físicos sin espacios, snake_case
- Añadir metadatos: origen, confidencialidad, GDPR

## 🧠 Ejercicio

1. Importa las tres tablas en SqlDBM
2. Establece relaciones PK-FK
3. Agrega descripciones a cada campo
4. Exporta `CREATE TABLE` documentado
5. Compara diseño visual vs implementación SQL

## Entregables

Dentro de tu repositorio forkeado, asegúrate de incluir los siguientes archivos:

* `fact_table.sql` – Script con la creación y carga de la tabla de hechos `S_ORDERS_FACT`
* `dim_tables.sql` – Script con la creación de las tablas de dimensiones `S_DIM_ORDER_CHANNEL` y `S_DIM_CUSTOMER_SEGMENT`
* `view_enriched.sql` – Script con la creación de la vista `S_ORDERS_ENRICHED` (join de enriquecimiento)
* `incremental_load.sql` – Script con la lógica de carga incremental hacia la tabla de hechos
* `sqlDBM_export.sql` – Export del modelo visual documentado en SqlDBM (con relaciones PK-FK y metadatos)
* `lab-notes.md` – Documento explicativo que incluya:
  * Descripción de los pasos del ETL y limpieza aplicada
  * Relación entre tablas y estructura del modelo en SqlDBM
  * Descripción de campos, claves primarias y foráneas
  * Respuestas al ejercicio de modelado
* *(Opcional)* Capturas de pantalla del modelo visual en SqlDBM o validaciones en Snowflake

## ✅ Conclusión

Este lab simula un escenario real en SEAT: múltiples entidades, dependencias, categorizaciones, auditoría y carga incremental. Te permite entender cómo diseñar una arquitectura limpia, eficiente y lista para analítica.

- 📁 Dataset: `B_ORDERS_RAW_ADVANCED.csv`  
- 📂 Silver destino: `DEV_SILVER_DB.S_SUPPLY_CHAIN`