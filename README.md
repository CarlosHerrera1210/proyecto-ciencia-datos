# Proyecto de ciencia de datos: horario de compra y recompra en Instacart

**Aprendiz:** Carlos Eduardo Herrera Sibaja  
**Programa:** Análisis y Desarrollo de Software  
**Última actualización:** 24/11/2026

## Propósito

Analizar los pedidos de Instacart para describir cuándo compran los clientes y qué tanto recompran productos. El proyecto avanza por etapas:

| Etapa | Actividad | Producto |
|---|---|---|
| 1. Diagnóstico de calidad | GA1-220501094-01-AA1-EV04 | `notebooks/data_quality.ipynb` y `reports/data_quality.md` |
| 2. Preparación, limpieza y estructuración | GA1-220501094-01-AA1-EV07 | `notebooks/data_preparation.ipynb` y `data/processed/` |

## Pregunta analítica del proyecto

¿Qué patrones descriptivos se observan en los pedidos y artículos de Instacart, en particular en el horario de compra y la recompra de productos?

## Fuentes de datos

Son cinco archivos CSV de Instacart, separados por `;`:

| Archivo | Contenido | Relaciones principales |
|---|---|---|
| `instacart_orders.csv` | Pedidos: usuario, hora, día y número de pedido | `order_id`, `user_id` |
| `order_products.csv` | Artículos incluidos en cada pedido (muestra) | `order_id`, `product_id` |
| `products.csv` | Catálogo de productos | `product_id`, `aisle_id`, `department_id` |
| `aisles.csv` | Catálogo de pasillos | `aisle_id` |
| `departments.csv` | Catálogo de departamentos | `department_id` |

Los CSV originales se guardan localmente en `data/raw/` y no se versionan por su tamaño (`order_products.csv` pesa 86 MB). Para obtenerlos, descárguelos de la [carpeta de datasets de Instacart](https://drive.google.com/drive/folders/19vYGYdE0hI1DzvpBcCZskzJ0u2TZjwas) y colóquelos en `data/raw/` sin modificarlos.

## Estructura

```text
proyecto-ciencia-datos/
├── README.md
├── .gitignore
├── requirements.txt
├── data/
│   ├── raw/                          # CSV originales (locales, no versionados)
│   └── processed/                    # Datasets limpios y estructurados (versionados)
├── notebooks/
│   ├── data_quality.ipynb            # Etapa 1: diagnóstico de calidad
│   └── data_preparation.ipynb        # Etapa 2: limpieza, análisis y estructuración
└── reports/
    └── data_quality.md               # Informe del diagnóstico de calidad
```

## Preparar y ejecutar

Desde la carpeta raíz del proyecto, cree el entorno virtual e instale las dependencias:

```bash
python -m venv .venv
```

En Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
jupyter lab
```

Si PowerShell bloquea la activación del entorno, ejecute antes `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`. En Linux o macOS, use `source .venv/bin/activate`.

Con los cinco CSV en `data/raw/`, abra los notebooks en este orden, seleccione el kernel del entorno `.venv` y ejecute **Run All**:

1. `notebooks/data_quality.ipynb`: menos de 1 minuto.
2. `notebooks/data_preparation.ipynb`: unos 3 minutos. Vuelve a generar todos los archivos de `data/processed/`.

Para ejecutarlos sin abrir Jupyter:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/data_quality.ipynb
jupyter nbconvert --to notebook --execute --inplace notebooks/data_preparation.ipynb
```

## Dependencias

| Paquete | Uso |
|---|---|
| `pandas` | Carga, limpieza, análisis y combinación de datos (probado con 3.1.1) |
| `pyarrow` | Lectura y escritura de archivos Parquet (probado con 26.0.1) |
| `jupyterlab` | Entorno para abrir y ejecutar los notebooks |
| `ipykernel` | Kernel de Python del entorno virtual |

Probado con Python 3.13.

## Etapa 1: diagnóstico de calidad

| Dataset | Filas | Columnas | Hallazgo principal |
|---|---:|---:|---|
| `instacart_orders.csv` | 478.967 | 6 | 15 filas duplicadas; 28.819 nulos en `days_since_prior_order` (todos son primeros pedidos) |
| `order_products.csv` | 4.545.007 | 4 | 836 nulos en `add_to_cart_order` (pedidos de más de 64 artículos) |
| `products.csv` | 49.694 | 4 | 1.258 nombres nulos (pasillo y departamento `missing`); nombres repetidos por formato |
| `aisles.csv` | 134 | 2 | Sin nulos; incluye la categoría `missing` |
| `departments.csv` | 21 | 2 | Sin nulos; incluye la categoría `missing` |

No hay claves huérfanas ni valores fuera de rango. El detalle está en [`reports/data_quality.md`](reports/data_quality.md).

## Etapa 2: preparación, limpieza y estructuración

### Transformaciones realizadas

**Limpieza y estandarización.** Se usan funciones reutilizables encadenadas con `pipe()`, que registran cada paso en una bitácora. Los resultados se validan con `assert`.

| Dataset | Transformación | Resultado |
|---|---|---|
| `orders` | Eliminar duplicados exactos | 478.967 → 478.952 filas |
| `orders` | Conservar los nulos de `days_since_prior_order` (`Int8`) y crear `is_first_order` | Los 28.817 nulos son exactamente los primeros pedidos; no se imputan |
| `orders` | Crear `day_name` a partir de `order_dow` | Día estandarizado (supuesto: 0 = domingo) |
| `order_products` | Conservar los nulos de `add_to_cart_order` (`Int8`); `reordered` como `bool` | 836 posiciones no registradas a partir del artículo 65 |
| `products` | Normalizar `product_name` (minúsculas, sin espacios sobrantes) | 419 nombres con espacios corregidos |
| `products` | Rellenar los nombres nulos con `unknown product` | 1.258 nulos → 0 |
| `products` | Crear `canonical_product_id` | 141 productos con nombre repetido apuntan a un id canónico |
| `aisles`, `departments` | Renombrar `missing` a `unclassified` | Categoría sin clasificar estandarizada |
| Todos | Ajustar los tipos (`int8`–`int32`, `Int8`, `bool`, `category`) | Memoria de `order_products`: 138,7 → 47,7 MB |

**Filtrado y agrupación.** Los pedidos se categorizan con `pd.cut()` e `isin()` según:
- franja horaria;
- fin de semana;
- etapa del cliente;
- intervalo desde el pedido anterior.

Se aplican filtros con condiciones booleanas, `isin()`, `between()` y `query()`, y los indicadores se calculan con `groupby()`, `agg()` y `pivot_table()`.

**Combinación y estructuración.** Las tablas se unen con `merge()` validado (`validate='m:1'`/`'1:1'` e `indicator`). Se comprueba que no se pierdan registros y que la tabla combinada reproduzca los indicadores. Con los pedidos originales la relación `m:1` falla por los duplicados, lo que confirma que la limpieza era necesaria.

### Indicadores principales

| Indicador | Resultado |
|---|---|
| Pedidos en la franja de mayor demanda | 47,3 % por la tarde (12-17 h); hora pico: 10 h |
| Tamaño promedio de la canasta | 10,1 artículos; alrededor de uno más en fin de semana |
| Tasa de recompra en pedidos recurrentes | 63,1 % |
| Recompra según la etapa del cliente | 36,9 % (pedidos 2-4) frente a 77,1 % (pedidos 15+) |
| Departamento con mayor recompra | `dairy eggs`, 71,5 % |
| Recompra según el intervalo entre pedidos | 67,5 % si vuelve en 0-7 días frente a 45,6 % si tarda 30+ días |

La demanda se concentra entre las 6 y las 17 h. La recompra depende sobre todo de la etapa del cliente, de la frecuencia de compra y del tipo de producto; la hora del pedido casi no la cambia.

### Datasets procesados (`data/processed/`)

| Archivo | Filas | Contenido |
|---|---:|---|
| `orders_consolidated.parquet` | 478.952 | Una fila por pedido, con categorías (`time_slot`, `is_weekend`, `customer_stage`, `prior_interval`) y resumen de la canasta (`n_items`, `n_reordered`, `n_departments`, `reorder_rate`, `has_items`) |
| `order_items.parquet` | 4.545.007 | Una fila por artículo de pedido, con `canonical_product_id`, `aisle` y `department` |
| `products.parquet` | 49.694 | Catálogo limpio con nombres de pasillo y departamento |
| `indicators_summary.csv` | 7 | Resumen de indicadores |
| `orders_by_time_slot.csv` | 4 | Pedidos y clientes por franja horaria |
| `reorder_by_department.csv` | 21 | Tasa de recompra y participación por departamento |
| `reorder_by_stage_and_time_slot.csv` | 3 | Tasa de recompra por etapa del cliente y franja horaria |

Los nulos que quedan son intencionales y están documentados en el notebook:
- **Primeros pedidos:** no tienen intervalo desde el pedido anterior.
- **28.906 pedidos sin artículos en la muestra:** sus métricas de canasta quedan nulas.
- **Artículos a partir del puesto 65 del carrito:** su posición no se registró.

Para leer los archivos:

```python
import pandas as pd

orders = pd.read_parquet('data/processed/orders_consolidated.parquet')
```

### Limitaciones

- `order_products` es una muestra: 28.906 pedidos no tienen artículos registrados.
- El nombre del día supone que `order_dow = 0` es domingo.
- `days_since_prior_order` tiene un tope de 30 días, así que 30 significa "30 días o más".

### Enfoque propio

Este proyecto no repite los laboratorios ni los retos de Instacart del curso: no usa histogramas, listas de los 20 productos más pedidos ni relleno con valores centinela. Analiza la pregunta del proyecto mediante franjas horarias, etapas del cliente, intervalos de compra y recompra por departamento.

## Flujo de trabajo en Git

- `main`: versión entregada en la etapa 1.
- `develop`: rama de integración de las actividades.
- `feature/project-data-preparation`: desarrollo de la etapa 2, fusionado en `develop`.
