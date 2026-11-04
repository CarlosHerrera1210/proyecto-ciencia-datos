# Diagnóstico inicial de calidad de datos de Instacart

**Aprendiz:** Carlos Eduardo Herrera Sibaja  
**Programa:** Análisis y Desarrollo de Software  
**Fecha:** 04/11/2026

## Propósito

Inspeccionar la estructura y calidad inicial de los cinco archivos de Instacart antes de continuar con su análisis. Se revisan dimensiones, nombres y tipos de columnas, valores ausentes, estadísticas descriptivas, duplicados y consistencia básica de las claves entre archivos.

## Pregunta analítica del proyecto

¿Qué patrones descriptivos se observan en los pedidos y artículos de Instacart, en particular en el horario de compra y la recompra de productos?

## Aspectos de calidad a revisar

1. Dimensiones, nombres de columnas y tipos de datos de cada archivo.
2. Valores faltantes por columna y su significado.
3. Filas duplicadas y unicidad de los identificadores.
4. Rangos válidos: hora (0–23), día (0–6), días desde el pedido anterior (0–30) y `reordered` (0/1).
5. Integridad de las claves entre archivos y categorías de relleno (`missing`) en los catálogos.

## Datasets

Los archivos CSV se leen con `;` como separador y cada archivo se carga en un DataFrame independiente:

| Archivo | Contenido | Relaciones principales |
|---|---|---|
| `instacart_orders.csv` | Pedidos, usuario, hora, día y número de pedido | `order_id`, `user_id` |
| `order_products.csv` | Artículos incluidos en cada pedido | `order_id`, `product_id` |
| `products.csv` | Catálogo de productos y sus categorías | `product_id`, `aisle_id`, `department_id` |
| `aisles.csv` | Catálogo de pasillos | `aisle_id` |
| `departments.csv` | Catálogo de departamentos | `department_id` |

Los CSV se mantienen localmente en `data/raw/` y están excluidos del control de versiones por su tamaño. Para obtenerlos, consulte la [carpeta de datasets de Instacart](https://drive.google.com/drive/folders/19vYGYdE0hI1DzvpBcCZskzJ0u2TZjwas). No publique datos con restricciones ni archivos grandes; descargue los archivos y colóquelos en `data/raw/`.

## Estructura

```text
proyecto-ciencia-datos/
├── README.md
├── .gitignore
├── requirements.txt
├── data/raw/                 # CSV locales, no versionados
├── notebooks/
│   └── diagnostico_calidad.ipynb
└── reports/
    └── informe_diagnostico.md
```

## Preparar y ejecutar

Desde la carpeta raíz del proyecto, cree y active un entorno virtual e instale las dependencias:

```bash
python -m venv .venv
```

En Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
jupyter lab
```

Si PowerShell bloquea la activación del entorno, ejecute antes `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`. En Linux o macOS use `source .venv/bin/activate`.

Abra `notebooks/diagnostico_calidad.ipynb`, seleccione el kernel del entorno `.venv` y ejecute **Run All**. Asegúrese de que los cinco CSV estén en `data/raw/`. La ejecución completa tarda menos de un minuto.

Alternativa sin abrir Jupyter:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/diagnostico_calidad.ipynb
```

## Dependencias

| Paquete | Uso |
|---|---|
| `pandas` | Carga e inspección de los datos (probado con 3.1.1) |
| `jupyterlab` | Entorno para abrir y ejecutar el Notebook |
| `ipykernel` | Kernel de Python del entorno virtual |

Probado con Python 3.13.

## Resumen de resultados

| Dataset | Filas | Columnas | Hallazgo principal |
|---|---:|---:|---|
| `instacart_orders.csv` | 478.967 | 6 | 15 filas duplicadas; 28.819 nulos en `days_since_prior_order` (todos son primeros pedidos) |
| `order_products.csv` | 4.545.007 | 4 | 836 nulos en `add_to_cart_order` (pedidos de más de 64 artículos) |
| `products.csv` | 49.694 | 4 | 1.258 nombres nulos (pasillo/departamento `missing`); 104 nombres repetidos por formato |
| `aisles.csv` | 134 | 2 | Sin nulos; incluye la categoría `missing` |
| `departments.csv` | 21 | 2 | Sin nulos; incluye la categoría `missing` |

No hay claves huérfanas ni valores fuera de rango. El detalle está en [`reports/informe_diagnostico.md`](reports/informe_diagnostico.md).

## Productos

- Notebook ejecutable con inspección estructural y controles iniciales de calidad.
- Informe técnico con resultados, interpretación, limitaciones y acciones propuestas: [`reports/informe_diagnostico.md`](reports/informe_diagnostico.md).
