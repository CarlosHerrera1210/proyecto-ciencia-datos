# Informe técnico: diagnóstico inicial de calidad de Instacart

**Aprendiz:** Carlos Eduardo Herrera Sibaja  
**Programa:** Análisis y Desarrollo de Software  
**Fecha:** 04/11/2026

## 1. Nombre del proyecto y pregunta analítica

**Proyecto:** Diagnóstico inicial de calidad de datos de Instacart.

Este diagnóstico revisa la estructura y la calidad inicial de cinco tablas de Instacart antes de usarlas en análisis posteriores. La pregunta analítica del proyecto es: **¿Qué patrones descriptivos se observan en los pedidos y artículos de Instacart, en particular en el horario de compra y la recompra de productos?**

## 2. Descripción de los datasets

| Archivo | Unidad de observación | Contenido | Relación con la pregunta |
|---|---|---|---|
| `instacart_orders.csv` | Un pedido | Usuario, número de pedido, día, hora y días desde el pedido anterior | Fuente del análisis de horario de compra y frecuencia |
| `order_products.csv` | Un artículo dentro de un pedido | Producto, posición en el carrito y si fue recomprado | Fuente del análisis de recompra |
| `products.csv` | Un producto | Nombre, pasillo y departamento | Permite nombrar y agrupar los productos |
| `aisles.csv` | Un pasillo | Nombre del pasillo | Catálogo para agrupar productos |
| `departments.csv` | Un departamento | Nombre del departamento | Catálogo para agrupar productos |

Las claves `order_id`, `product_id`, `aisle_id` y `department_id` relacionan las tablas. Todos los archivos usan `;` como separador.

**Aspectos de calidad revisados:** dimensiones y tipos, valores faltantes, filas duplicadas y unicidad de identificadores, rangos válidos (hora 0–23, día 0–6, intervalo 0–30, `reordered` 0/1), integridad de claves entre archivos y categorías de relleno en los catálogos.

## 3. Resultados de la inspección estructural

Resultados obtenidos con `shape`, `dtypes`, `info()` y `describe()` en `notebooks/data_quality.ipynb`.

| Dataset | Filas | Columnas | Tipos de datos | Observaciones |
|---|---:|---:|---|---|
| `instacart_orders.csv` | 478.967 | 6 | `order_id`, `user_id`, `order_number`, `order_dow`, `order_hour_of_day`: int64; `days_since_prior_order`: float64 | 28.819 nulos en `days_since_prior_order`; 15 filas duplicadas exactas |
| `order_products.csv` | 4.545.007 | 4 | `order_id`, `product_id`, `reordered`: int64; `add_to_cart_order`: float64 | 836 nulos en `add_to_cart_order`; tabla más grande (≈90 MB) |
| `products.csv` | 49.694 | 4 | `product_id`, `aisle_id`, `department_id`: int64; `product_name`: str (texto) | 1.258 nulos en `product_name` |
| `aisles.csv` | 134 | 2 | `aisle_id`: int64; `aisle`: str | Sin nulos ni duplicados; incluye la categoría `missing` (id 100) |
| `departments.csv` | 21 | 2 | `department_id`: int64; `department`: str | Sin nulos ni duplicados; incluye la categoría `missing` (id 21) |

## 4. Valores faltantes, tipos de datos e inconsistencias

**Valores faltantes**
- `days_since_prior_order`: 28.819 nulos. Coinciden exactamente con los 28.819 pedidos con `order_number == 1`, y no hay ningún nulo fuera de los primeros pedidos. El nulo significa "no hay pedido anterior".
- `add_to_cart_order`: 836 nulos en 70 pedidos. Todos esos pedidos tienen 65 artículos o más, y el máximo valor registrado es 64. Indica un límite en el registro de la posición en el carrito.
- `product_name`: 1.258 nulos. Todos los productos afectados están en el pasillo 100 y el departamento 21, ambos llamados `missing` en sus catálogos.

**Tipos de datos**
- Los identificadores, contadores, día y hora son enteros (int64), coherentes con su significado.
- `add_to_cart_order` es float64 solo porque contiene nulos; todos sus valores son enteros. Debería ser un entero con nulos (`Int64`).
- `days_since_prior_order` es float64 por la misma razón; sus valores están entre 0 y 30.
- Las columnas de texto se cargan como `str` (pandas 3.x), lo cual es adecuado.

**Inconsistencias**
- `instacart_orders.csv` tiene 15 filas duplicadas exactas: 15 `order_id` repetidos con valores idénticos. Todas son del día 3 a las 2 h, lo que sugiere un error de carga y no pedidos reales.
- `products.csv` tiene 104 nombres repetidos al ignorar mayúsculas y espacios (0 repetidos de forma exacta). Pueden ser el mismo producto escrito de forma distinta.
- No se encontraron valores fuera de rango en hora (0–23), día (0–6) ni `reordered` (0/1).
- No hay claves huérfanas: todos los `order_id` y `product_id` de `order_products` existen en sus tablas, y todos los `aisle_id` y `department_id` de `products` existen en los catálogos.

## 5. Principales hallazgos de calidad por dataset

| Dataset | Problema identificado | Evidencia encontrada | Posible impacto | Acción propuesta |
|---|---|---|---|---|
| `instacart_orders.csv` | Filas duplicadas | `orders.duplicated().sum()` = 15; `order_id.duplicated().sum()` = 15; todas con `order_dow` = 3 y `order_hour_of_day` = 2 | Infla en 15 los pedidos del miércoles a las 2 h y altera las frecuencias por horario | Eliminar duplicados con `drop_duplicates()` y verificar que `order_id` quede único |
| `instacart_orders.csv` | Nulos en `days_since_prior_order` | 28.819 nulos = 28.819 pedidos con `order_number == 1`; 0 nulos en otros pedidos | Si se imputan con 0 se falsea la frecuencia de compra | Conservar como nulo y excluirlo solo en cálculos de intervalo; no imputar |
| `instacart_orders.csv` | Tipo float en intervalo | `dtypes`: `days_since_prior_order` float64 | Bajo; solo de presentación | Convertir a `Int64` si se necesita un entero |
| `order_products.csv` | Nulos en `add_to_cart_order` | 836 nulos en 70 pedidos; máximo = 64; pedidos afectados de ≥ 65 artículos | Pierde la posición del artículo 65 en adelante; no afecta a `reordered` | Conservar las filas (sí sirven para la recompra); documentar el límite de 64; opcionalmente marcar con un valor centinela |
| `order_products.csv` | Tipo float en `add_to_cart_order` | `dtypes`: float64; 0 valores no enteros | Bajo; puede confundir al interpretar | Convertir a `Int64` |
| `order_products.csv` | Gran volumen | `shape` = (4.545.007, 4) | Mayor consumo de memoria y tiempo | Leer solo las columnas necesarias (`usecols`) y evitar copias |
| `products.csv` | Nombres ausentes | 1.258 nulos en `product_name`, todos con `aisle_id` = 100 y `department_id` = 21 (`missing`) | No se pueden nombrar estos productos en los resultados | Etiquetar como `Unknown` y conservar sus identificadores |
| `products.csv` | Nombres repetidos por formato | 104 duplicados tras `str.lower().str.strip()`; 0 exactos | Un mismo producto podría contarse por separado al agrupar por nombre | Normalizar el texto antes de agrupar y revisar los casos |
| `aisles.csv` / `departments.csv` | Categoría de relleno | `aisle` = `missing` (id 100); `department` = `missing` (id 21) | Una categoría sin significado aparecería en los rankings | Tratarla como "sin clasificar" o excluirla de los análisis por categoría |
| Los cinco datasets | Integridad de claves | 0 claves huérfanas; 0 valores fuera de rango | Ninguno | No requiere acción |

## 6. Acciones de limpieza y preparación propuestas

1. Eliminar las 15 filas duplicadas de `instacart_orders` y confirmar que `order_id` es único.
2. Mantener los nulos de `days_since_prior_order` como "primer pedido" y documentarlo.
3. Convertir `add_to_cart_order` y `days_since_prior_order` a `Int64`.
4. Documentar el límite de 64 posiciones en el carrito y conservar esas filas para el análisis de recompra.
5. Reemplazar los `product_name` nulos por `Unknown` y tratar las categorías `missing` como "sin clasificar".
6. Normalizar `product_name` (minúsculas, sin espacios extra) antes de agrupar por nombre.
7. Cargar `order_products` solo con las columnas necesarias para cada análisis.

En esta actividad las acciones solo se proponen; los datos originales no se modifican.

## 7. Conclusiones

Los cinco archivos están en buenas condiciones para continuar con el proyecto. Las claves son íntegras, los rangos son válidos y casi todos los nulos tienen una explicación: primer pedido, límite del carrito o productos sin clasificar. El único error real son las 15 filas duplicadas de pedidos, que deben eliminarse antes de analizar el horario de compra. Para la recompra, `reordered` está completa y sin valores inválidos. Tras una limpieza ligera, los datos permiten responder la pregunta analítica, siempre que se respete la granularidad de cada tabla: una fila de `order_products` es un artículo, no un pedido ni un cliente.

Para reproducir los resultados, ejecute `notebooks/data_quality.ipynb` siguiendo las instrucciones del `README.md`.
