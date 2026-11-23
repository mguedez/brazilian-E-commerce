# Plan de cambios: Caso de estudio 1 (Olist), de la entrega de avance a la final

> **Entrega final:** lunes 23/11/2026. Informe PDF (≤ 6 págs. sin referencias) + notebook ejecutable de principio a fin + datos curados + controles.
>
> Fuentes de este plan: consigna (`Caso de estudio 1_ datos que merezcan confianza.pdf`), informe de avance (`caso-estudio-1.pdf`), notebook (`brazilian_e_commerce.ipynb`) y las dos devoluciones del profe (`feedback-profe.jpeg`, `feedback-profe-2.jpeg`).
>
> **Fuera de alcance por ahora:** feature engineering y geolocalización fina (lat/long, mapas por ciudad), porque van en el entregable 2. Solo se dejan anotados al final.
>
> ⚠️ Las cifras marcadas con **(≈ verificar)** son valores conocidos del dataset público de Olist, pero **no las corrí acá** porque el dataset no está descargado en esta máquina. Hay que confirmarlas al ejecutar el notebook y usar siempre el número que salga del código.

---

## 0. Resumen ejecutivo (qué tenemos y qué falta)

| Criterio (devolución) | Nivel actual | Qué hace falta para llegar a "Sólido/Destacado" |
|---|---|---|
| 1. Preguntas y predicción | En camino | Reescribir las preguntas en clave de **confiabilidad + decisión + población**. Hipótesis de calidad **concretas por tabla**, con criterio de refutación y contraste una por una. |
| 2. Entender los datos | Sólido | Agregar inventario cuantitativo (filas, tamaño, período, procedencia y licencia) y diagrama de relaciones. |
| 3. Explorar | En camino | Más gráficos de distribución (histogramas, boxplots, scatter, correlaciones, categóricas), cobertura temporal, y **lectura de cada gráfico** (evidencia → explicación → decisión → límite). Contexto externo para los picos. |
| 4. Calidad | En camino | **Validez** (precios, fechas fuera de rango, secuencias imposibles), **consistencia** (pagos vs. ítems), **cobertura temporal**, procedencia. Perfil reproducible en una tabla. |
| 5. Integrar y curar | En camino | `validate=` en todos los merge, filas antes/después (99.441 como control), población declarada, versión curada con **grano + contrato**, flags en vez de borrar. |
| 6. Cierre y uso de IA | En camino | Controles con `assert`/Pandera, **desafío con ≥ 3 reglas violadas**, usos habilitados / con revisión / no recomendados, **declaración de IA obligatoria**. |

### Problemas concretos detectados en el notebook actual (además de lo que dijo el profe)

1. **La facturación promedio por estado está sesgada hacia abajo.** En la celda de facturación promedio se divide `facturacion.sum()` (que ignora NaN) por `order_id.nunique()` (que **sí** cuenta los pedidos sin ítems). Los ≈ 775 pedidos sin ítems **(≈ verificar)** suman al denominador y no al numerador. Además se mezclan pedidos `canceled` y `unavailable`.
2. **Costo de envío y facturación usan poblaciones distintas.** `costo_envio.mean()` ignora NaN, pero la facturación promedio no. Los dos gráficos comparan poblaciones diferentes sin decirlo.
3. **`drop_duplicates(inplace=True)` sobre `dfs`** pisa la versión cruda en memoria. Después no se puede comparar antes y después (la consigna lo exige explícitamente).
4. **`review_score` se promedia por pedido** sin revisar que `order_reviews` tiene `review_id` duplicados y pedidos con más de una reseña **(≈ verificar: ~814 review_id repetidos, ~550 pedidos con >1 reseña)**. Promediar es una decisión de curación y hay que documentarla.
5. **`states` se mergea 3 veces** (copy-paste) sin `validate` ni control de que no queden NaN en `state_name`.
6. **Los gráficos no tienen población ni interpretación.** La consigna pide título, unidades, población e interpretación en cada visualización.
7. **`errors="coerce"` en fechas** puede convertir en silencio valores inválidos en NaT. Hay que contar cuántos NaT nuevos aparecen respecto de los NaN originales (deberían ser 0).
8. **No existe ningún control ejecutable** (`assert`, Pandera). Hoy nada puede "fallar".
9. **No hay versión curada exportada** ni contrato.
10. **Reproducibilidad:** falta una celda de instrucciones (`pip install`, credenciales de Kaggle, versión del dataset) y un `requirements.txt`. Falta también `.gitignore` para no subir los CSV crudos.
11. Detalles: la columna `product_name_lenght` está mal escrita en la fuente (conviene renombrarla en la curada y documentarlo); celdas con `sales_by_state.head(15)` en el medio que no muestran nada (solo se muestra la última expresión).

---

## 1. Cambios en el INFORME (PDF)

### 1.1 Estructura propuesta (≤ 6 páginas sin referencias)

| Sección | Págs. aprox. | Contenido |
|---|---|---|
| 1. Introducción y pregunta central | 0,3 | Dataset, procedencia y las 3 preguntas reformuladas, con la **decisión** que apoya cada una. |
| 2. Comprensión de los datos | 0,7 | Tabla de inventario con **filas, columnas, tamaño, período y grano/clave**, diagrama de relaciones y `customer_id` vs. `customer_unique_id`. |
| 3. Predicción de calidad y contraste | 0,7 | Tabla de hipótesis: tabla, por qué, qué la refutaría, resultado (✔ confirmada / ✘ refutada / parcial). |
| 4. Exploración | 1,5 | 4–6 gráficos clave, cada uno con evidencia, explicación, decisión y límite. |
| 5. Diagnóstico de calidad | 1 | Perfil por dimensión: completitud, unicidad, validez, consistencia, integridad, cobertura temporal y procedencia. Clasificar cada hallazgo como *inválido / evento poco frecuente / ausencia con significado / problema de integración / limitación de la fuente*. |
| 6. Curación, contrato y controles | 1 | Grano, contrato, decisiones, antes/después (99.441 → N) y lista de controles. |
| 7. Desafío y conclusiones | 0,6 | 3+ casos inyectados con qué detecta y qué no cada control, y usos habilitados / con revisión / no recomendados. |
| 8. Declaración de uso de IA | 0,2 | Obligatoria (ver §1.7). |
| Referencias + Anexo (link al notebook/repo) | — | No cuentan para las 6 págs. |

### 1.2 Reescribir las preguntas de análisis (crítica principal del profe)

**Problema:** las preguntas actuales ("¿en qué estados se concentran las ventas?", etc.) son descriptivas. No dicen **qué decisión se toma**, **sobre qué población**, ni **qué problema de calidad podría falsear la respuesta**.

**Propuesta: tres preguntas en formato decisión + población + amenaza.**

> **P1 (Volumen y facturación por estado).** ¿Puede el dataset sostener la decisión de **priorizar estados para inversión comercial** (marketing, acuerdos logísticos) según su volumen de pedidos y su facturación?
> - **Población:** pedidos con `order_status == "delivered"` y al menos un ítem, comprados entre **2017-01-01 y 2018-08-31** (período con cobertura completa; ver §3.6).
> - **Métrica:** cantidad de pedidos, clientes únicos (`customer_unique_id`) y facturación (`sum(price)`) por estado.
> - **Problemas de calidad que podrían falsearla:** incluir pedidos cancelados o no disponibles (inflan la facturación); pedidos sin ítems (sesgan el promedio); contar `customer_id` como si fueran personas; meses incompletos al inicio y al final de la serie; estados con pocos pedidos (varianza alta).

> **P2 (Costo de envío entre regiones).** ¿Es `freight_value` lo bastante confiable para **comparar el costo logístico entre regiones** y decidir, por ejemplo, un subsidio de envío por estado?
> - **Población:** ítems de pedidos entregados, con agregación a nivel pedido y también como **proporción flete / precio**.
> - **Problemas de calidad que podrían falsearla:** `freight_value = 0` (¿envío gratis o dato faltante?); el flete se reparte entre ítems del mismo pedido; inconsistencias entre `payments.payment_value` y `price + freight_value`; el costo depende del **vendedor** (origen) y no solo del destino.

> **P3 (Tiempos de entrega y satisfacción).** ¿Se puede usar el dataset para **evaluar el desempeño logístico por región** (atraso respecto de la fecha estimada) y su relación con la satisfacción (`review_score`)?
> - **Población:** pedidos `delivered` con `order_delivered_customer_date` no nula y secuencia de fechas válida.
> - **Problemas de calidad que podrían falsearla:** fechas en orden imposible (entrega antes de la compra, despacho antes de la aprobación); reseñas duplicadas o múltiples por pedido; reseñas creadas antes de la entrega; pedidos `delivered` sin fecha de entrega.

**Texto para poner antes de las preguntas en el informe (borrador):**

> La pregunta central del caso es qué decisiones puede sostener el dataset y qué problemas reducen su confiabilidad. La traducimos en tres decisiones concretas de una empresa de e-commerce: priorizar estados comercialmente (P1), definir políticas de envío por región (P2) y evaluar el desempeño logístico (P3). Para cada una declaramos la población sobre la que se calcula y los problemas de calidad que podrían invalidarla. La exploración y el diagnóstico de calidad se organizan para responder si esas amenazas se materializan.

### 1.3 Reescribir las hipótesis de calidad (crítica: "muy generales")

Formato pedido por el profe: **en qué tabla, por qué, qué la refutaría**, y después contrastarlas una por una. Propuesta (en el informe va como tabla y el resultado sale del notebook):

| # | Dimensión | Tabla.columna | Hipótesis (predicción) | Por qué lo esperamos | Qué la refutaría | Resultado esperado *(completar con notebook)* |
|---|---|---|---|---|---|---|
| H1 | Completitud | `orders.order_delivered_customer_date` | Los nulos se concentran en estados ≠ `delivered`; en `delivered` hay < 0,1 % de nulos. | Un pedido no entregado no puede tener fecha de entrega. | Que > 0,1 % de los `delivered` tengan la fecha nula. | Parcial: 8 `delivered` sin fecha → anomalía menor y se marca con flag. |
| H2 | Unicidad | `geolocation` | Hay filas completamente duplicadas. | La tabla se arma por prefijo de CP con muchas coordenadas por prefijo. | 0 filas duplicadas. | Confirmada: 261.831 duplicadas. |
| H3 | Unicidad | `order_reviews.review_id` | `review_id` **no** es único, y hay pedidos con > 1 reseña. | El sistema de encuestas puede reenviar o asociar una reseña a varios pedidos. | `review_id` único y 1 reseña por pedido. | ≈ verificar. |
| H4 | Integridad | `orders` → `order_items` | Existen pedidos sin ítems. | Pedidos cancelados o no disponibles antes de cargar productos. | Todos los `order_id` de `orders` aparecen en `order_items`. | ≈ 775 pedidos sin ítems (verificar), concentrados en `unavailable`/`canceled`. |
| H5 | Validez | `orders` (fechas) | Hay secuencias de fechas imposibles (aprobación < compra, entrega < despacho, etc.). | Carga manual o asincrónica de eventos logísticos. | 0 violaciones de orden temporal. | ≈ verificar (se esperan cientos en `carrier_date < approved_at`). |
| H6 | Validez | `order_items.freight_value`, `price` | `price > 0` siempre; existen `freight_value == 0`. | Envío gratis promocional. | `price <= 0` (sería inválido) / ningún flete 0. | ≈ verificar (≈ 380 ítems con flete 0). |
| H7 | Consistencia | `order_payments` vs. `order_items` | `sum(payment_value)` ≈ `sum(price + freight_value)` por pedido, con diferencias por cuotas/vouchers. | Intereses de cuotas, vouchers y redondeos. | Diferencias grandes (> R$ 1) en > 5 % de los pedidos. | ≈ verificar. |
| H8 | Cobertura temporal | `orders.order_purchase_timestamp` | Los extremos de la serie (fines de 2016, sep–oct 2018) están incompletos. | El dataset es una muestra con fecha de corte. | Volumen mensual estable en todo el período. | ≈ verificar (2016 casi vacío; sep/oct 2018 con muy pocos pedidos). |
| H9 | Completitud | `products` | Los 610 productos sin categoría son **los mismos** que no tienen nombre, descripción ni fotos. | Un mismo proceso de carga fallido (feedback-2: "productos con misma cantidad de faltantes"). | Que los nulos estén en filas distintas. | ≈ verificar (se espera que coincidan). |

> **Nota para el informe:** la consigna pide "predecir los **dos principales** problemas". Elegir explícitamente **dos** como predicción principal (sugerencia: **H4 integridad pedidos–ítems** y **H8 cobertura temporal**, porque son las que más afectan P1) y presentar el resto como hipótesis secundarias.

### 1.4 Sección de exploración en el informe

El informe de avance **no muestra ningún resultado** (crítica explícita). Para cada gráfico importante, un párrafo corto con esta plantilla:

> **Figura N. [Título].** *Población:* … *Unidades:* …
> **Evidencia:** qué se ve (con números).
> **Explicación posible:** por qué pasa.
> **Decisión:** qué hacemos en la curación o el análisis por esto.
> **Límite:** qué todavía no podemos concluir.

Ejemplo, ya escrito para el gráfico de pedidos por estado:

> **Figura 2. Pedidos entregados por estado del cliente (2017-01 a 2018-08).** *Población:* pedidos `delivered` con ítems (n = …). *Unidades:* cantidad de pedidos.
> **Evidencia:** SP concentra ≈ 42 % de los pedidos, seguido de RJ y MG; los 3 suman ≈ 66 %.
> **Explicación:** coincide con la distribución de población y de PIB de Brasil, y con que la mayoría de los vendedores está en SP (menor costo y plazo de envío).
> **Decisión:** reportar también **pedidos por cada 100 mil habitantes** para no confundir tamaño de mercado con penetración; los estados con < 100 pedidos se marcan como "muestra insuficiente" en lugar de excluirlos sin aviso.
> **Límite:** el dataset es una muestra de Olist, no del e-commerce brasileño total; no podemos afirmar participación de mercado.

### 1.5 Sección de calidad: perfil por dimensión

Agregar una tabla resumen (sale de la celda §3.9 del notebook):

| Dimensión | Regla | Umbral | Resultado | Clasificación | Decisión |
|---|---|---|---|---|---|
| Completitud | `delivered` ⇒ `order_delivered_customer_date` no nula | 0 violaciones esperadas; tolerancia < 0,1 % | … | Inválido (menor) | Flag `flag_fecha_entrega_faltante` |
| Unicidad | `order_id` único en `orders` | 0 | 0 | — | Control `assert` |
| Validez | `price > 0`, `freight_value >= 0` | 0 | … | — / Evento poco frecuente | Flag `flag_flete_cero` |
| Validez | `purchase <= approved <= carrier <= customer` | … | … | Inválido o problema de carga | Flag; se excluyen de P3 |
| Validez | fechas dentro de [2016-09-01, 2018-10-31] | 0 | … | — | `assert` |
| Consistencia | \|pagos − (precio + flete)\| ≤ R$ 1 | ≥ 95 % de pedidos | … | Limitación de la fuente (cuotas/vouchers) | Flag `flag_pago_inconsistente` |
| Integridad | FKs sin huérfanos | 0 | 0 | — | `assert` |
| Integridad | pedidos sin ítems | — | ≈ 775 | Ausencia con significado (canceled/unavailable) | Fuera de la población de P1/P2 |
| Cobertura temporal | pedidos/mes ≥ 1.000 | — | 2016 y sep–oct 2018 no cumplen | Limitación de la fuente | Ventana 2017-01 → 2018-08 |
| Procedencia | Fuente Kaggle, versión, licencia CC BY-NC-SA 4.0, anonimizado | — | — | — | Documentado |

La tabla tiene que mostrar explícitamente la distinción que pide la consigna: **inválido / poco frecuente / ausencia con significado / problema de integración / limitación de la fuente**.

### 1.6 Sección de cierre

> **Usos habilitados:** ranking de estados por volumen de pedidos entregados (P1) en la ventana 2017-01 → 2018-08; distribución de costos de flete por estado de destino (P2) como valor relativo; tasa de atraso por estado (P3) excluyendo pedidos con fechas inválidas.
>
> **Usos que exigen revisión adicional:** facturación absoluta (diferencias pagos vs. ítems, vouchers); análisis por ciudad (nombres sin normalizar, muchas ciudades con n chico); relación satisfacción–atraso (reseñas duplicadas o múltiples); análisis de recurrencia de clientes (pocos `customer_unique_id` con más de un pedido).
>
> **Usos no recomendados:** estimar participación de mercado o crecimiento real del e-commerce brasileño (es una muestra de un solo marketplace); comparar 2016 o sep–oct 2018 con el resto (cobertura incompleta); conclusiones sobre estados con < 100 pedidos; cualquier análisis a nivel persona (datos anonimizados, sin demografía).

### 1.7 Declaración de uso de IA (obligatoria, hoy falta)

Plantilla, siguiendo la secuencia **predecir → intentar → consultar → verificar → declarar**:

| Herramienta | Para qué | Qué modificamos | Cómo validamos |
|---|---|---|---|
| Claude (Anthropic) | Revisión del notebook y del informe contra la consigna y la devolución; propuesta de checklist de cambios y de controles. | Reescribimos las preguntas e hipótesis con nuestros propios datos; ajustamos umbrales. | Ejecutamos cada control en el notebook y verificamos los números contra la salida; cada integrante revisó y puede explicar las celdas. |
| (otras: ChatGPT, Copilot, etc.) | … | … | … |

Aclarar también que **no se compartieron datos sensibles ni credenciales** (el dataset es público y anonimizado).

### 1.8 Otros arreglos menores del informe

- Tabla 1: agregar columnas **filas**, **columnas**, **tamaño (MB)** y **clave/grano**. Corregir "Identificacion" y "ubicacion" (faltan tildes).
- Agregar **período** exacto (min/max de `order_purchase_timestamp`) y **procedencia**: Kaggle `olistbr/brazilian-ecommerce`, versión descargada, fecha de descarga y licencia CC BY-NC-SA 4.0.
- Diagrama de relaciones (esquema estrella: `orders` al centro). Puede ser una imagen hecha con draw.io o dbdiagram.
- Referencias: agregar Pandera, la consigna y las fuentes externas usadas para los picos (§3.7). En la referencia de Olist, poner la URL de Kaggle.
- Anexo: link al **repositorio** además de Colab, porque la corrección no puede depender de servicios externos.

---

## 2. Cambios estructurales en el NOTEBOOK

### 2.1 Nueva estructura de secciones

```
0. Título, integrantes, cómo reproducir (NUEVA)
1. Setup: imports, constantes, semilla (MODIFICADA)
2. Carga de la fuente original (raw, inmutable) (MODIFICADA)
3. Comprensión: inventario de archivos (NUEVA)
4. Predicción de calidad: hipótesis (NUEVA, markdown)
5. Exploración
   5.1 Tipos y resumen numérico (min/max/percentiles) (NUEVA)
   5.2 Variables categóricas: distribuciones (NUEVA)
   5.3 Variables numéricas: histogramas, boxplots y outliers (NUEVA)
   5.4 Relaciones: scatter y correlaciones (NUEVA)
   5.5 Cobertura temporal y contexto externo (NUEVA)
   5.6 Texto: reseñas y nombres de ciudades (NUEVA)
6. Diagnóstico de calidad
   6.1 Completitud (EXISTE, ampliar)
   6.2 Unicidad y claves (EXISTE, ampliar con reviews)
   6.3 Validez (NUEVA)
   6.4 Consistencia (NUEVA)
   6.5 Integridad (EXISTE, ampliar en ambos sentidos)
   6.6 Cobertura temporal (NUEVA)
   6.7 Contraste de hipótesis (NUEVA)
   6.8 Perfil de calidad consolidado (NUEVA)
7. Curación
   7.1 Población declarada (NUEVA)
   7.2 Tabla curada con validate= y antes/después (MODIFICADA)
   7.3 Flags de calidad (NUEVA)
   7.4 Contrato (Pandera) (NUEVA)
   7.5 Exportación (NUEVA)
8. Análisis de las preguntas P1, P2, P3 (MODIFICADA: gráficos existentes + lectura)
9. Controles ejecutables (NUEVA)
10. Desafío: casos que violan reglas (NUEVA)
11. Conclusiones y declaración de IA (NUEVA, markdown)
```

### 2.2 Archivos nuevos en el repo

```
.gitignore          # data/raw/, data/curated/*.parquet, *.csv
requirements.txt    # pandas, numpy, matplotlib, seaborn, kagglehub, pandera, pyarrow
README.md           # cómo obtener datos y correr el notebook
data/curated/       # (ignorado) salida del notebook
```

`.gitignore` sugerido:
```gitignore
data/raw/
data/curated/
*.csv
*.parquet
.ipynb_checkpoints/
```

`requirements.txt` sugerido:
```
pandas>=2.2
numpy>=1.26
matplotlib>=3.8
seaborn>=0.13
kagglehub>=0.3
pandera>=0.20
pyarrow>=15
```

---

## 3. Celdas nuevas y modificadas (código listo para pegar)

> Convención: `raw` = fuente original (nunca se modifica); `cur` = versiones curadas. Todos los gráficos llevan **título, unidades, población** y, debajo, una celda markdown de **lectura**.

### 3.0 [NUEVA, markdown] Cómo reproducir

```markdown
# Caso de estudio 1: Olist, datos que merezcan confianza
**Integrantes:** Gustavo Hunter, María Clara Gómez, Manuela Guédez, Guillermo Tula

## Cómo reproducir
1. `pip install -r requirements.txt`
2. Configurar credenciales de Kaggle (`~/.kaggle/kaggle.json` o variables `KAGGLE_USERNAME`/`KAGGLE_KEY`).
   En Colab, `kagglehub` pide login interactivo.
3. Ejecutar *Run All*. El notebook descarga el dataset `olistbr/brazilian-ecommerce` con `kagglehub`,
   **no modifica los CSV originales** y escribe la versión curada en `data/curated/`.
4. Los datos crudos **no** se suben al repositorio (ver `.gitignore`).

**Fuente:** https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce · Licencia CC BY-NC-SA 4.0
```

### 3.1 [MODIFICADA] Setup

```python
import os, hashlib
from pathlib import Path

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import kagglehub
import pandera.pandas as pa  # en pandera < 0.24 usar: import pandera as pa

pd.set_option("display.float_format", "{:,.2f}".format)
sns.set_theme(style="whitegrid")

CURATED_DIR = Path("data/curated")
CURATED_DIR.mkdir(parents=True, exist_ok=True)

# Período con cobertura completa (se justifica en 6.6)
VENTANA_INI = pd.Timestamp("2017-01-01")
VENTANA_FIN = pd.Timestamp("2018-08-31 23:59:59")
MIN_PEDIDOS_ESTADO = 100
```

### 3.2 [MODIFICADA] Carga raw inmutable + inventario

Reemplaza la celda gigante de carga. Separa **carga**, **inventario** y **exploración**, y guarda un hash para la procedencia.

```python
path = kagglehub.dataset_download("olistbr/brazilian-ecommerce")
print("Ruta del dataset:", path)

files = {
    "customers": "olist_customers_dataset.csv",
    "geolocation": "olist_geolocation_dataset.csv",
    "order_items": "olist_order_items_dataset.csv",
    "order_payments": "olist_order_payments_dataset.csv",
    "order_reviews": "olist_order_reviews_dataset.csv",
    "orders": "olist_orders_dataset.csv",
    "products": "olist_products_dataset.csv",
    "sellers": "olist_sellers_dataset.csv",
    "category_translation": "product_category_name_translation.csv",
}

raw = {name: pd.read_csv(os.path.join(path, f)) for name, f in files.items()}

def md5(fp, chunk=1 << 20):
    h = hashlib.md5()
    with open(fp, "rb") as fh:
        for b in iter(lambda: fh.read(chunk), b""):
            h.update(b)
    return h.hexdigest()

inventario = pd.DataFrame([
    {
        "tabla": name,
        "archivo": f,
        "filas": raw[name].shape[0],
        "columnas": raw[name].shape[1],
        "tamaño_MB": os.path.getsize(os.path.join(path, f)) / 1e6,
        "filas_duplicadas": raw[name].duplicated().sum(),
        "md5": md5(os.path.join(path, f)),
    }
    for name, f in files.items()
])
inventario
```

```python
# Período cubierto por cada tabla con fechas
fechas = {
    "orders": "order_purchase_timestamp",
    "order_reviews": "review_creation_date",
    "order_items": "shipping_limit_date",
}
pd.DataFrame({
    t: {"min": pd.to_datetime(raw[t][c]).min(), "max": pd.to_datetime(raw[t][c]).max()}
    for t, c in fechas.items()
}).T
```

> **Markdown debajo:** grano y clave de cada tabla (orders: 1 fila por pedido, PK `order_id`; order_items: PK `(order_id, order_item_id)`; payments: PK `(order_id, payment_sequential)`; reviews: **sin PK confiable**, ver 6.2; geolocation: sin PK, varias coordenadas por prefijo de CP).

### 3.3 [NUEVA, markdown] Predicción de calidad

Copiar la tabla H1–H9 de §1.3 (sin la columna de resultado). **Tiene que ir antes de la exploración** porque la consigna pide predecir antes de mirar.

### 3.4 [NUEVA] Exploración: resumen numérico (min/max) y tipos

```python
num_tables = ["order_items", "order_payments", "products", "order_reviews"]
for t in num_tables:
    print("=" * 60, t)
    display(raw[t].describe(percentiles=[.01, .05, .25, .5, .75, .95, .99]).T)
```

> **Lectura (markdown):** comentar mín/máx sospechosos. Por ejemplo: `price` mín ≈ 0,85 y máx ≈ 6.735 (verificar); `payment_installments` con mín 0 (inválido: no puede haber 0 cuotas); `payment_value` = 0; productos con `product_weight_g` = 0.

### 3.5 [NUEVA] Distribuciones de variables categóricas

Lo pide el feedback 2: "si bien el ranking es numérico, semánticamente es categórico". `review_score` y `payment_installments` se tratan como **categóricas ordinales**.

```python
cat_vars = [
    ("orders", "order_status"),
    ("order_payments", "payment_type"),
    ("order_payments", "payment_installments"),
    ("order_reviews", "review_score"),
    ("customers", "customer_state"),
    ("sellers", "seller_state"),
]

fig, axes = plt.subplots(3, 2, figsize=(14, 14))
for ax, (t, c) in zip(axes.flat, cat_vars):
    vc = raw[t][c].value_counts().sort_index() if c in ("review_score", "payment_installments") \
         else raw[t][c].value_counts()
    vc.plot.bar(ax=ax)
    ax.set_title(f"{t}.{c}  (n = {len(raw[t]):,} filas de {t})")
    ax.set_ylabel("Cantidad de registros")
    for i, v in enumerate(vc.values):
        ax.text(i, v, f"{v/vc.sum():.1%}", ha="center", va="bottom", fontsize=7)
plt.tight_layout(); plt.show()
```

```python
# Top 20 categorías de producto (por ítems vendidos), traducidas
items_cat = (raw["order_items"]
    .merge(raw["products"][["product_id", "product_category_name"]],
           on="product_id", how="left", validate="many_to_one")
    .merge(raw["category_translation"], on="product_category_name",
           how="left", validate="many_to_one"))

top_cat = items_cat["product_category_name_english"].fillna("(sin categoría)").value_counts().head(20)
plt.figure(figsize=(10, 7))
sns.barplot(x=top_cat.values, y=top_cat.index)
plt.title(f"Top 20 categorías por ítems vendidos (población: todos los ítems, n = {len(items_cat):,})")
plt.xlabel("Cantidad de ítems"); plt.ylabel("Categoría")
plt.tight_layout(); plt.show()

# Control: ¿hay categorías sin traducción?
sin_trad = items_cat.loc[items_cat["product_category_name"].notna()
                         & items_cat["product_category_name_english"].isna(),
                         "product_category_name"].unique()
print("Categorías sin traducción:", sin_trad)
```

> **Lectura:** el desbalance de `order_status` (≈ 97 % delivered) justifica que la población de P1 sea `delivered`. `review_score` es **bimodal** (mucho 5 y bastante 1), así que conviene reportar la mediana o el % de 1–2 en vez de la media. `payment_type` tiene `not_defined` (≈ 3 filas, inválido). Hay categorías sin traducción (≈ 2: `pc_gamer`, `portateis_cozinha_e_preparadores_de_alimentos`; verificar), que es un **problema de integración**.

### 3.6 [NUEVA] Distribuciones numéricas y outliers

```python
items = raw["order_items"]

fig, axes = plt.subplots(2, 2, figsize=(14, 9))
sns.histplot(items["price"], bins=100, ax=axes[0, 0])
axes[0, 0].set(title="Distribución de precio por ítem", xlabel="Precio (R$)", ylabel="Ítems")

sns.histplot(np.log10(items["price"]), bins=80, ax=axes[0, 1])
axes[0, 1].set(title="Precio por ítem (escala log10)", xlabel="log10(Precio R$)", ylabel="Ítems")

sns.histplot(items["freight_value"], bins=100, ax=axes[1, 0])
axes[1, 0].set(title="Distribución de flete por ítem", xlabel="Flete (R$)", ylabel="Ítems")

sns.boxplot(x=items["freight_value"], ax=axes[1, 1])
axes[1, 1].set(title="Boxplot de flete por ítem (outliers IQR)", xlabel="Flete (R$)")
fig.suptitle(f"Población: todos los ítems de order_items (n = {len(items):,})")
plt.tight_layout(); plt.show()
```

```python
# Outliers por regla IQR: cuántos y qué son
def outliers_iqr(s, k=1.5):
    q1, q3 = s.quantile([.25, .75])
    iqr = q3 - q1
    return (s < q1 - k * iqr) | (s > q3 + k * iqr)

for col in ["price", "freight_value"]:
    m = outliers_iqr(items[col])
    print(f"{col}: {m.sum():,} outliers IQR ({m.mean():.1%}) · máx = {items[col].max():,.2f}")

# Los 10 ítems más caros con su categoría: ¿son plausibles?
items_cat.nlargest(10, "price")[["order_id", "price", "freight_value", "product_category_name_english"]]
```

```python
# Boxplot de flete por estado del cliente (top 10 por volumen)
it_state = (items
    .merge(raw["orders"][["order_id", "customer_id"]], on="order_id", validate="many_to_one")
    .merge(raw["customers"][["customer_id", "customer_state"]], on="customer_id", validate="many_to_one"))
top10 = it_state["customer_state"].value_counts().head(10).index
plt.figure(figsize=(12, 6))
sns.boxplot(data=it_state[it_state["customer_state"].isin(top10)],
            x="customer_state", y="freight_value", showfliers=True)
plt.yscale("log")
plt.title("Flete por ítem según estado del cliente (top 10 por volumen)")
plt.ylabel("Flete (R$, escala log)"); plt.xlabel("Estado del cliente")
plt.tight_layout(); plt.show()
```

> **Lectura:** el precio tiene una distribución fuertemente asimétrica a la derecha (por eso la escala log). Los outliers de precio son **eventos poco frecuentes** (electrónica, muebles), **no errores**: se mantienen y se reportan con la mediana. Los fletes 0 son un caso a analizar aparte (§3.11).

### 3.7 [NUEVA] Relaciones: scatter y correlaciones

```python
prod = raw["products"].rename(columns={
    "product_name_lenght": "product_name_length",
    "product_description_lenght": "product_description_length",
})
items_prod = items.merge(prod, on="product_id", how="left", validate="many_to_one")
items_prod["volumen_cm3"] = (items_prod["product_length_cm"]
                             * items_prod["product_height_cm"]
                             * items_prod["product_width_cm"])

fig, axes = plt.subplots(1, 2, figsize=(14, 5))
sample = items_prod.sample(20_000, random_state=42)
sns.scatterplot(data=sample, x="price", y="freight_value", alpha=.2, s=8, ax=axes[0])
axes[0].set(xscale="log", yscale="log", title="Precio vs. flete por ítem (muestra 20k)",
            xlabel="Precio (R$, log)", ylabel="Flete (R$, log)")
sns.scatterplot(data=sample, x="product_weight_g", y="freight_value", alpha=.2, s=8, ax=axes[1])
axes[1].set(xscale="symlog", yscale="log", title="Peso vs. flete por ítem (muestra 20k)",
            xlabel="Peso (g, symlog)", ylabel="Flete (R$, log)")
plt.tight_layout(); plt.show()
```

```python
num_cols = ["price", "freight_value", "product_weight_g", "volumen_cm3",
            "product_photos_qty", "product_name_length", "product_description_length"]
corr = items_prod[num_cols].corr(method="spearman")
plt.figure(figsize=(8, 6))
sns.heatmap(corr, annot=True, fmt=".2f", cmap="RdBu_r", vmin=-1, vmax=1)
plt.title("Correlación de Spearman entre atributos numéricos (población: ítems)")
plt.tight_layout(); plt.show()
```

> **Lectura:** se usa Spearman porque las distribuciones son asimétricas. Se espera que el flete correlacione más con peso y volumen que con precio. Si es así, para **P2** el costo de envío debe compararse controlando por peso, porque si no un estado "caro" puede ser solo un estado que compra productos más pesados. **Límite:** el flete también depende de la distancia vendedor–cliente, que se analiza en el entregable 2 (geolocalización).

### 3.8 [NUEVA] Cobertura temporal + contexto externo (picos de venta)

Lo pide el feedback 2: "inyectar datos macro/micro para justificar picos". Es exploración, no feature engineering.

```python
o = raw["orders"].copy()
o["order_purchase_timestamp"] = pd.to_datetime(o["order_purchase_timestamp"])
o["mes"] = o["order_purchase_timestamp"].dt.to_period("M")

mensual = o.groupby(["mes", "order_status"]).size().unstack(fill_value=0)
mensual["total"] = mensual.sum(axis=1)
display(mensual[["total", "delivered", "canceled"]])

diario = o.set_index("order_purchase_timestamp").resample("D").size()

# Eventos externos documentados (citar fuentes en el informe)
eventos = {
    "2017-11-24": "Black Friday 2017",
    "2018-05-21": "Paro de camioneros (greve dos caminhoneiros)",
    "2018-06-14": "Inicio Mundial 2018",
    "2017-12-25": "Navidad",
}

fig, ax = plt.subplots(figsize=(15, 5))
diario.plot(ax=ax, lw=.8)
for fecha, label in eventos.items():
    ax.axvline(pd.Timestamp(fecha), color="red", ls="--", lw=.8)
    ax.text(pd.Timestamp(fecha), diario.max() * .95, label, rotation=90, fontsize=8, va="top")
ax.axvspan(VENTANA_INI, VENTANA_FIN, color="green", alpha=.06, label="Ventana de análisis")
ax.set(title="Pedidos por día (todos los estados del pedido)", xlabel="Fecha de compra", ylabel="Pedidos/día")
ax.legend(); plt.tight_layout(); plt.show()
```

```python
# Contexto macro (opcional pero suma): IPCA / tipo de cambio BRL-USD del Banco Central de Brasil (SGS)
# Serie 433 = IPCA mensual; serie 3698 = USD/BRL venta mensual. API pública, sin credenciales.
def bcb_serie(codigo, ini="01/01/2016", fin="31/12/2018"):
    url = (f"https://api.bcb.gov.br/dados/serie/bcdata.sgs.{codigo}/dados"
           f"?formato=json&dataInicial={ini}&dataFinal={fin}")
    s = pd.read_json(url)
    s["data"] = pd.to_datetime(s["data"], dayfirst=True)
    return s.set_index("data")["valor"].rename(codigo)

try:
    usd = bcb_serie(3698).resample("M").mean()
    pedidos_mes = o.set_index("order_purchase_timestamp").resample("M").size()
    fig, ax1 = plt.subplots(figsize=(13, 4))
    pedidos_mes.plot(ax=ax1, label="Pedidos/mes")
    ax2 = ax1.twinx(); usd.plot(ax=ax2, color="orange", label="USD/BRL")
    ax1.set(title="Pedidos mensuales vs. tipo de cambio USD/BRL (BCB, serie 3698)",
            ylabel="Pedidos/mes"); ax2.set_ylabel("BRL por USD")
    plt.tight_layout(); plt.show()
except Exception as e:
    print("No se pudo descargar la serie del BCB (el notebook sigue igual):", e)
```

> **Lectura:**
> - **Evidencia:** pico claro el 24/11/2017 (Black Friday), con ≈ 1.100+ pedidos en un día (verificar). Hay una caída o desaceleración alrededor del paro de camioneros (mayo 2018). La serie empieza casi vacía en sep–dic 2016 (con un hueco) y se corta abruptamente en sep 2018, donde casi todos los pedidos son `canceled`/no entregados.
> - **Explicación:** Black Friday es un evento comercial conocido; el corte de 2018 corresponde a la fecha de extracción del dataset, no a una caída real de ventas.
> - **Decisión:** la ventana de análisis es **2017-01 → 2018-08**. El pico de Black Friday se mantiene como **evento poco frecuente válido**, no es un outlier a eliminar.
> - **Límite:** la correlación con variables macro es ilustrativa; con ≈ 20 meses no se puede inferir causalidad.

### 3.9 [NUEVA] Texto: reseñas y nombres de ciudad

Lo pide el feedback 2: "cómo manejar texto".

```python
rv = raw["order_reviews"].copy()
rv["tiene_titulo"] = rv["review_comment_title"].notna()
rv["tiene_mensaje"] = rv["review_comment_message"].notna()
rv["largo_mensaje"] = rv["review_comment_message"].fillna("").str.len()

# ¿La ausencia de comentario tiene significado? → % con mensaje según score
display(rv.groupby("review_score")[["tiene_titulo", "tiene_mensaje"]].mean()
          .style.format("{:.1%}"))

plt.figure(figsize=(9, 5))
sns.boxplot(data=rv[rv["tiene_mensaje"]], x="review_score", y="largo_mensaje")
plt.title("Largo del comentario según puntaje (población: reseñas con comentario)")
plt.xlabel("Puntaje (categoría ordinal 1–5)"); plt.ylabel("Caracteres")
plt.tight_layout(); plt.show()

# Limpieza básica de texto (para uso futuro, no se sobrescribe la columna original)
import unicodedata, re
def normalizar_texto(s):
    if pd.isna(s): return s
    s = unicodedata.normalize("NFKD", str(s)).encode("ascii", "ignore").decode()
    s = re.sub(r"\s+", " ", s.lower()).strip()
    return s

rv["mensaje_norm"] = rv["review_comment_message"].map(normalizar_texto)
rv.loc[rv["tiene_mensaje"], ["review_comment_message", "mensaje_norm"]].head()
```

```python
# Nombres de ciudad: ¿hay variantes del mismo nombre?
cities = raw["customers"][["customer_city", "customer_state"]].drop_duplicates()
cities["city_norm"] = cities["customer_city"].map(normalizar_texto)
variantes = (cities.groupby(["customer_state", "city_norm"])["customer_city"]
             .nunique().loc[lambda s: s > 1])
print(f"Ciudades con > 1 grafía en customers: {len(variantes)}")

# Mismo chequeo cruzado: ciudades de sellers que no matchean exacto con customers
sellers_c = set(raw["sellers"]["seller_city"].map(normalizar_texto))
raw_s = set(raw["sellers"]["seller_city"])
print("Ciudades de sellers que cambian al normalizar:", len(raw_s) - len(sellers_c))
raw["sellers"].loc[raw["sellers"]["seller_city"].str.contains(r"[\d/@\\-]", regex=True), "seller_city"].unique()[:20]
```

> **Lectura:** los comentarios faltantes son **ausencia con significado** (el comentario es opcional), no un error. **No se imputan.** Se espera que los puntajes bajos tengan más comentarios y más largos (las quejas se explican). En `seller_city` aparecen valores sucios (por ejemplo, con "/", "\\", números o emails; verificar), que son un **problema de validez** y se documentan. El análisis por ciudad queda como "requiere revisión".

### 3.10 [MODIFICADA] Completitud (ampliar la celda de faltantes)

Agregar a la celda existente:

```python
# Faltantes en fechas cruzado con estado → tabla (no solo prints)
date_cols = ["order_approved_at", "order_delivered_carrier_date", "order_delivered_customer_date"]
tabla_nulos = pd.DataFrame({c: raw["orders"].loc[raw["orders"][c].isna(), "order_status"].value_counts()
                            for c in date_cols}).fillna(0).astype(int)
tabla_nulos["total_por_estado"] = raw["orders"]["order_status"].value_counts()
tabla_nulos
```

```python
# Feedback 2: "revisar productos que tienen misma cantidad de faltantes" → ¿son las mismas filas?
p = raw["products"]
cols_610 = ["product_category_name", "product_name_lenght",
            "product_description_lenght", "product_photos_qty"]
mismas_filas = p[cols_610].isna().all(axis=1).sum()
alguna = p[cols_610].isna().any(axis=1).sum()
print(f"Productos con las 4 columnas nulas: {mismas_filas} · con al menos una: {alguna}")

cols_dim = ["product_weight_g", "product_length_cm", "product_height_cm", "product_width_cm"]
print("Productos sin dimensiones (las 4):", p[cols_dim].isna().all(axis=1).sum())
print(p[p[cols_dim].isna().any(axis=1)])

# ¿Esos productos se venden? ¿cuánto pesan en la facturación?
ids_sin_cat = p.loc[p["product_category_name"].isna(), "product_id"]
it_sin_cat = items[items["product_id"].isin(ids_sin_cat)]
print(f"Ítems de productos sin categoría: {len(it_sin_cat):,} ({len(it_sin_cat)/len(items):.2%}) "
      f"· {it_sin_cat['price'].sum()/items['price'].sum():.2%} de la facturación")

# Mapa de nulos (patrón visual)
plt.figure(figsize=(10, 4))
sns.heatmap(p.isna().sort_values(cols_610[0]).T, cbar=False)
plt.title("Patrón de nulos en products (filas ordenadas; negro/claro = nulo)")
plt.xlabel("Productos"); plt.tight_layout(); plt.show()
```

> **Lectura:** si las 610 filas coinciden, se trata de **un solo problema de carga** (faltó el catálogo para esos productos), no de cuatro problemas separados. **Decisión:** `product_category_name` se rellena con `"sin_categoria"` en la curada (categoría explícita, no NaN) y se agrega el flag `flag_producto_sin_catalogo`.

### 3.11 [NUEVA] Unicidad: reseñas

```python
rv = raw["order_reviews"]
print("review_id duplicados:", rv["review_id"].duplicated().sum())
print("(review_id, order_id) duplicados:", rv.duplicated(["review_id", "order_id"]).sum())
print("pedidos con > 1 reseña:", (rv["order_id"].value_counts() > 1).sum())
print("pedidos sin reseña:", (~raw["orders"]["order_id"].isin(rv["order_id"])).sum())

# Ejemplo de un review_id compartido por varios pedidos
dup_id = rv.loc[rv["review_id"].duplicated(keep=False)].sort_values("review_id")
dup_id.head(6)
```

> **Decisión:** para cada pedido se toma la **reseña más reciente** (`review_answer_timestamp` máx.) en vez del promedio, y se marca `flag_multi_review`. Se documenta que `review_id` **no es PK** (problema de la fuente).

### 3.12 [NUEVA] Validez

```python
o = raw["orders"].copy()
for c in ["order_purchase_timestamp", "order_approved_at", "order_delivered_carrier_date",
          "order_delivered_customer_date", "order_estimated_delivery_date"]:
    nulos_antes = o[c].isna().sum()
    o[c] = pd.to_datetime(o[c], errors="coerce")
    nuevos_nat = o[c].isna().sum() - nulos_antes
    assert nuevos_nat == 0, f"{c}: {nuevos_nat} fechas no parseables convertidas a NaT"

reglas_validez = {
    # Orden temporal
    "approved < purchase":          (o["order_approved_at"] < o["order_purchase_timestamp"]),
    "carrier < approved":           (o["order_delivered_carrier_date"] < o["order_approved_at"]),
    "carrier < purchase":           (o["order_delivered_carrier_date"] < o["order_purchase_timestamp"]),
    "customer < carrier":           (o["order_delivered_customer_date"] < o["order_delivered_carrier_date"]),
    "customer < purchase":          (o["order_delivered_customer_date"] < o["order_purchase_timestamp"]),
    "estimated < purchase":         (o["order_estimated_delivery_date"] < o["order_purchase_timestamp"]),
    # Rango
    "purchase fuera de 2016-09..2018-10":
        ~o["order_purchase_timestamp"].between("2016-09-01", "2018-10-31 23:59:59"),
    "entrega > 180 días":
        (o["order_delivered_customer_date"] - o["order_purchase_timestamp"]).dt.days > 180,
    # Estado vs. fechas
    "delivered sin fecha entrega":
        (o["order_status"] == "delivered") & o["order_delivered_customer_date"].isna(),
    "canceled con fecha entrega":
        (o["order_status"] == "canceled") & o["order_delivered_customer_date"].notna(),
}
it, pay, pr = raw["order_items"], raw["order_payments"], raw["products"]
reglas_validez_otras = {
    "items.price <= 0":                  (it["price"] <= 0),
    "items.freight_value < 0":           (it["freight_value"] < 0),
    "items.freight_value == 0":          (it["freight_value"] == 0),
    "payments.payment_value <= 0":       (pay["payment_value"] <= 0),
    "payments.installments < 1":         (pay["payment_installments"] < 1),
    "payments.type == not_defined":      (pay["payment_type"] == "not_defined"),
    "products.weight_g == 0":            (pr["product_weight_g"] == 0),
    "reviews.score fuera de 1..5":       ~raw["order_reviews"]["review_score"].between(1, 5),
}

validez = pd.DataFrame(
    [{"regla": k, "tabla": "orders", "violaciones": int(v.sum()), "pct": v.mean() * 100}
     for k, v in reglas_validez.items()] +
    [{"regla": k, "tabla": k.split(".")[0], "violaciones": int(v.sum()), "pct": v.mean() * 100}
     for k, v in reglas_validez_otras.items()]
)
validez
```

```python
# ¿Las violaciones temporales se concentran en algún período o estado del pedido?
m = reglas_validez["carrier < approved"]
o.loc[m].groupby(o.loc[m, "order_purchase_timestamp"].dt.to_period("M")).size().plot.bar(figsize=(12, 3))
plt.title("Pedidos con despacho anterior a la aprobación, por mes de compra")
plt.ylabel("Pedidos"); plt.tight_layout(); plt.show()
```

> **Lectura:** clasificar cada regla en la tabla del informe. `carrier < approved` en pocas horas probablemente es un **problema de registro** (zona horaria o carga asincrónica): se marca con flag y se excluye de P3. `freight_value == 0` es un **evento poco frecuente válido** (envío gratis): se mantiene con flag. `installments < 1` y `not_defined` son **inválidos**: flag y exclusión del análisis de pagos.

### 3.13 [NUEVA] Consistencia: pagos vs. ítems

```python
pagos_pedido = raw["order_payments"].groupby("order_id")["payment_value"].sum().rename("pagado")
items_pedido = (raw["order_items"].assign(total=lambda d: d["price"] + d["freight_value"])
                .groupby("order_id")["total"].sum().rename("items_total"))
cons = pd.concat([pagos_pedido, items_pedido], axis=1, join="inner")
cons["dif"] = cons["pagado"] - cons["items_total"]

tol = 1.0  # R$; justificar: redondeos y centavos
print(f"Pedidos comparables: {len(cons):,}")
print(f"|dif| <= R$ {tol}: {(cons['dif'].abs() <= tol).mean():.2%}")
print(cons["dif"].describe(percentiles=[.01, .05, .5, .95, .99]))

plt.figure(figsize=(10, 4))
sns.histplot(cons.loc[cons["dif"].abs() > tol, "dif"].clip(-200, 200), bins=100)
plt.title("Diferencia pagado − (precio + flete) en pedidos con |dif| > R$ 1 (recortado a ±200)")
plt.xlabel("Diferencia (R$)"); plt.ylabel("Pedidos"); plt.tight_layout(); plt.show()

# Pedidos con pagos pero sin ítems, y viceversa
print("Con pagos y sin ítems:", len(set(pagos_pedido.index) - set(items_pedido.index)))
print("Con ítems y sin pagos:", len(set(items_pedido.index) - set(pagos_pedido.index)))
```

> **Lectura:** las diferencias positivas son compatibles con **intereses de cuotas** (cruzar con `payment_installments`) y las negativas con **vouchers**. **Decisión:** la facturación de P1 se mide con `price` (valor de la mercadería), no con `payment_value`, y eso se declara. Se agrega el flag `flag_pago_inconsistente`. **Límite:** sin datos de promociones no se puede confirmar la causa de cada diferencia.

### 3.14 [MODIFICADA] Integridad (en ambos sentidos)

La celda actual solo revisa hijo → padre. Hay que agregar padre → hijo, que es donde aparece el problema:

```python
ids_orders = set(raw["orders"]["order_id"])
padre_sin_hijo = {
    "orders sin items":    len(ids_orders - set(raw["order_items"]["order_id"])),
    "orders sin payments": len(ids_orders - set(raw["order_payments"]["order_id"])),
    "orders sin reviews":  len(ids_orders - set(raw["order_reviews"]["order_id"])),
    "products nunca vendidos": len(set(raw["products"]["product_id"]) - set(raw["order_items"]["product_id"])),
    "customers sin orders": len(set(raw["customers"]["customer_id"]) - set(raw["orders"]["customer_id"])),
    "zip clientes sin geolocation": (~raw["customers"]["customer_zip_code_prefix"]
                                     .isin(raw["geolocation"]["geolocation_zip_code_prefix"])).sum(),
}
display(pd.Series(padre_sin_hijo, name="registros"))

# ¿Qué estados tienen los pedidos sin ítems?
sin_items = raw["orders"][~raw["orders"]["order_id"].isin(raw["order_items"]["order_id"])]
sin_items["order_status"].value_counts()
```

> **Lectura:** los pedidos sin ítems son casi todos `unavailable`/`canceled`, es decir, **ausencia con significado**. Esto explica el sesgo del promedio de facturación del notebook de avance. Los CP sin geolocalización se documentan para el entregable 2.

### 3.15 [NUEVA] Contraste de hipótesis (una por una)

```python
resultado_hip = pd.DataFrame([
    {"H": "H1", "evidencia": f"{validez.set_index('regla').loc['delivered sin fecha entrega','violaciones']} delivered sin fecha",
     "umbral": "< 0,1 % de delivered", "veredicto": None},
    {"H": "H2", "evidencia": f"{raw['geolocation'].duplicated().sum():,} filas duplicadas", "umbral": "> 0", "veredicto": None},
    {"H": "H3", "evidencia": f"{raw['order_reviews']['review_id'].duplicated().sum()} review_id dup.", "umbral": "> 0", "veredicto": None},
    {"H": "H4", "evidencia": f"{padre_sin_hijo['orders sin items']} pedidos sin ítems", "umbral": "> 0", "veredicto": None},
    # ... H5–H9
])
# Completar 'veredicto' con lógica (confirmada/refutada/parcial) y comentar cada una en markdown
resultado_hip
```

> En markdown, una línea por hipótesis: **predicción → evidencia → confirmada/refutada → qué aprendimos**. Esto responde directamente al "contrástenlas una por una".

### 3.16 [NUEVA] Perfil de calidad consolidado

Una celda que arme la tabla del §1.5 a partir de `validez`, `padre_sin_hijo`, `key_checks`, `checks_fk`, `missing_df` y la cobertura mensual. Esa tabla va al informe tal cual (perfil **reproducible**).

---

## 4. Curación (celdas que reemplazan "Limpieza de duplicados" y "Tabla analítica")

### 4.1 [NUEVA, markdown] Población y grano declarados

```markdown
## Población y grano de la tabla curada `orders_curated`

- **Grano:** 1 fila = 1 pedido (`order_id`, PK).
- **Incluye:** todos los pedidos de `orders` (99.441). **No se borran filas**: la población de
  cada pregunta se define con columnas booleanas (`en_poblacion_p1`, etc.) y flags de calidad.
  Así se puede comparar antes y después y saber qué filas quedaron afectadas.
- **Población P1/P2:** `order_status == "delivered"` ∧ tiene ítems ∧ compra en [2017-01-01, 2018-08-31].
- **Población P3:** P1 ∧ secuencia de fechas válida ∧ `order_delivered_customer_date` no nula.
- **Facturación:** `sum(price)` de los ítems (valor de mercadería, sin flete ni intereses).
- **Reseña:** la más reciente del pedido.
```

### 4.2 [MODIFICADA] Geolocalización sin `inplace`

```python
cur = {}  # versiones curadas; raw queda intacto
cur["geolocation"] = raw["geolocation"].drop_duplicates()
print(f"geolocation: {len(raw['geolocation']):,} → {len(cur['geolocation']):,} "
      f"(−{len(raw['geolocation']) - len(cur['geolocation']):,} duplicados exactos)")
assert cur["geolocation"].duplicated().sum() == 0
```

### 4.3 [MODIFICADA] Tabla analítica con `validate=` y antes/después

```python
N_ORDERS = len(raw["orders"])  # 99.441: control de grano
log = []  # bitácora antes/después

def paso(df, desc):
    log.append({"paso": desc, "filas": len(df), "cols": df.shape[1]})
    assert df["order_id"].is_unique, f"Se rompió el grano en: {desc}"
    assert len(df) == N_ORDERS, f"Cambió la cantidad de pedidos en: {desc} ({len(df)})"
    return df

# Orders con fechas tipadas (ya validadas en 3.12)
orders_c = o.copy()
orders_c = paso(orders_c, "orders tipado")

# Ítems agregados a nivel pedido
order_values = (raw["order_items"].groupby("order_id")
    .agg(facturacion=("price", "sum"), costo_envio=("freight_value", "sum"),
         cantidad_items=("order_item_id", "count"), n_sellers=("seller_id", "nunique"),
         flete_cero_items=("freight_value", lambda s: (s == 0).sum()))
    .reset_index())

# Reseña más reciente por pedido
rv_last = (raw["order_reviews"]
    .assign(review_answer_timestamp=lambda d: pd.to_datetime(d["review_answer_timestamp"]),
            review_creation_date=lambda d: pd.to_datetime(d["review_creation_date"]))
    .sort_values("review_answer_timestamp")
    .groupby("order_id")
    .agg(review_score=("review_score", "last"),
         review_creation_date=("review_creation_date", "last"),
         n_reviews=("review_id", "count"))
    .reset_index())

pagos = (raw["order_payments"].groupby("order_id")
         .agg(pagado=("payment_value", "sum"), cuotas_max=("payment_installments", "max"),
              medio_pago_principal=("payment_type", lambda s: s.mode().iat[0]))
         .reset_index())

df = orders_c.merge(raw["customers"][["customer_id", "customer_unique_id", "customer_city", "customer_state"]],
                    on="customer_id", how="left", validate="one_to_one")
df = paso(df, "+ customers (1:1)")

df = df.merge(order_values, on="order_id", how="left", validate="one_to_one")
df = paso(df, "+ items agregados (1:1)")

df = df.merge(pagos, on="order_id", how="left", validate="one_to_one")
df = paso(df, "+ pagos agregados (1:1)")

df = df.merge(rv_last, on="order_id", how="left", validate="one_to_one")
df = paso(df, "+ reseña más reciente (1:1)")

df = df.merge(states, left_on="customer_state", right_on="state_code", how="left", validate="many_to_one")
df = paso(df, "+ nombre de estado (m:1)")
assert df["state_name"].notna().all(), "Hay códigos de estado sin nombre"

pd.DataFrame(log)
```

> Esta tabla de bitácora es el **"99.441 en la tabla analítica: ese es el control"** que pidió el profe. Si cualquier merge duplica filas, `validate=` o el `assert` cortan la ejecución.

### 4.4 [NUEVA] Flags de calidad y población

```python
df["tiene_items"] = df["cantidad_items"].notna()
df["en_ventana"] = df["order_purchase_timestamp"].between(VENTANA_INI, VENTANA_FIN)

df["flag_fecha_entrega_faltante"] = (df["order_status"] == "delivered") & df["order_delivered_customer_date"].isna()
df["flag_secuencia_fechas_invalida"] = (
    (df["order_approved_at"] < df["order_purchase_timestamp"]) |
    (df["order_delivered_carrier_date"] < df["order_purchase_timestamp"]) |
    (df["order_delivered_customer_date"] < df["order_delivered_carrier_date"]) |
    (df["order_delivered_customer_date"] < df["order_purchase_timestamp"])
)
df["flag_flete_cero"] = df["flete_cero_items"].fillna(0) > 0
df["flag_multi_review"] = df["n_reviews"].fillna(0) > 1
df["flag_pago_inconsistente"] = (df["pagado"] - (df["facturacion"] + df["costo_envio"])).abs() > 1.0
df["flag_estado_muestra_chica"] = df.groupby("customer_state")["order_id"].transform("count") < MIN_PEDIDOS_ESTADO

df["en_poblacion_p1"] = (df["order_status"] == "delivered") & df["tiene_items"] & df["en_ventana"]
df["en_poblacion_p3"] = (df["en_poblacion_p1"] & ~df["flag_secuencia_fechas_invalida"]
                         & ~df["flag_fecha_entrega_faltante"])

# Métricas derivadas mínimas para P3 (no es feature engineering: son la definición de la pregunta)
df["dias_entrega"] = (df["order_delivered_customer_date"] - df["order_purchase_timestamp"]).dt.days
df["dias_atraso"] = (df["order_delivered_customer_date"] - df["order_estimated_delivery_date"]).dt.days
df["entregado_tarde"] = df["dias_atraso"] > 0

flags = [c for c in df.columns if c.startswith("flag_") or c.startswith("en_poblacion")]
df[flags].sum().to_frame("filas_afectadas").assign(pct=lambda d: d["filas_afectadas"] / len(df) * 100)
```

```python
# Antes / después por pregunta: cuántos pedidos quedan y por qué salieron
embudo = pd.Series({
    "orders (raw)": len(df),
    "delivered": (df["order_status"] == "delivered").sum(),
    "+ con ítems": ((df["order_status"] == "delivered") & df["tiene_items"]).sum(),
    "+ en ventana (P1/P2)": df["en_poblacion_p1"].sum(),
    "+ fechas válidas (P3)": df["en_poblacion_p3"].sum(),
}, name="pedidos")
embudo.to_frame().assign(pct_del_total=lambda d: d["pedidos"] / len(df) * 100)
```

```python
# Sensibilidad: ¿cuánto cambia la facturación promedio si NO declarábamos la población? (feedback 1)
comp = pd.DataFrame({
    "avance (sum/nunique, todos)": df.groupby("state_name")["facturacion"].sum() / df.groupby("state_name")["order_id"].nunique(),
    "final (media, población P1)": df[df["en_poblacion_p1"]].groupby("state_name")["facturacion"].mean(),
})
comp["dif_%"] = (comp.iloc[:, 1] / comp.iloc[:, 0] - 1) * 100
comp.sort_values("dif_%", ascending=False).head(10)
```

> Esta última celda **demuestra con números** el problema que señaló el profe: la tabla analítica incluía cancelados y no disponibles, y la facturación los sumaba.

### 4.5 [NUEVA] Contrato con Pandera

```python
ESTADOS_VALIDOS = states["state_code"].tolist()
STATUS_VALIDOS = ["delivered", "shipped", "canceled", "unavailable", "invoiced",
                  "processing", "created", "approved"]

contrato_orders = pa.DataFrameSchema(
    {
        "order_id": pa.Column(str, unique=True, nullable=False, checks=pa.Check.str_length(32, 32)),
        "customer_id": pa.Column(str, unique=True, nullable=False),
        "customer_unique_id": pa.Column(str, nullable=False),
        "order_status": pa.Column(str, pa.Check.isin(STATUS_VALIDOS)),
        "customer_state": pa.Column(str, pa.Check.isin(ESTADOS_VALIDOS)),
        "order_purchase_timestamp": pa.Column("datetime64[ns]", nullable=False,
            checks=pa.Check.in_range(pd.Timestamp("2016-09-01"), pd.Timestamp("2018-10-31 23:59:59"))),
        "order_delivered_customer_date": pa.Column("datetime64[ns]", nullable=True),
        "facturacion": pa.Column(float, pa.Check.gt(0), nullable=True),
        "costo_envio": pa.Column(float, pa.Check.ge(0), nullable=True),
        "cantidad_items": pa.Column(float, pa.Check.ge(1), nullable=True),
        "review_score": pa.Column(float, pa.Check.isin([1, 2, 3, 4, 5]), nullable=True),
        "en_poblacion_p1": pa.Column(bool),
    },
    checks=[
        # Regla de negocio: en la población P1 todo pedido tiene facturación y fecha de entrega
        pa.Check(lambda d: d.loc[d["en_poblacion_p1"], "facturacion"].notna().all(),
                 error="P1 con facturación nula"),
        pa.Check(lambda d: len(d) == N_ORDERS, error="Cambió el grano/cantidad de pedidos"),
    ],
    strict=False,
    coerce=False,
)

orders_curated = contrato_orders.validate(df, lazy=True)
print("✔ Contrato válido:", orders_curated.shape)
```

> En el informe, el contrato va resumido como tabla: **columna · tipo · nulos permitidos · regla**.

### 4.6 [NUEVA] Exportación

```python
orders_curated.to_parquet(CURATED_DIR / "orders_curated.parquet", index=False)
cur["geolocation"].to_parquet(CURATED_DIR / "geolocation_dedup.parquet", index=False)
pd.DataFrame(log).to_csv(CURATED_DIR / "bitacora_curacion.csv", index=False)
print(os.listdir(CURATED_DIR))
```

---

## 5. Análisis de las preguntas (modificar los 4 gráficos existentes)

Cambios comunes a **todos** los gráficos actuales:

1. **Usar `orders_curated[orders_curated["en_poblacion_p1"]]`** en vez de `orders_analysis`.
2. **Título con población y n**: `f"... (pedidos entregados 2017-01 a 2018-08, n = {n:,})"`.
3. **Etiquetas con unidades** (R$, pedidos, días).
4. **Eliminar los 3 merges repetidos con `states`**, que ya está en la curada.
5. **Celda markdown de lectura** debajo de cada uno (plantilla evidencia/explicación/decisión/límite).
6. Para promedios: mostrar **mediana** además de media, o barras de error (IC 95 %), porque la distribución es asimétrica.

### 5.1 P1: pedidos por estado (reemplaza la celda actual)

```python
p1 = orders_curated[orders_curated["en_poblacion_p1"]]
n = len(p1)

by_state = (p1.groupby(["customer_state", "state_name"])
    .agg(pedidos=("order_id", "count"),
         clientes_unicos=("customer_unique_id", "nunique"),
         facturacion_total=("facturacion", "sum"),
         facturacion_media=("facturacion", "mean"),
         facturacion_mediana=("facturacion", "median"))
    .reset_index()
    .sort_values("pedidos", ascending=False))
by_state["%_pedidos"] = by_state["pedidos"] / n * 100
by_state["%_acum"] = by_state["%_pedidos"].cumsum()
by_state["muestra_chica"] = by_state["pedidos"] < MIN_PEDIDOS_ESTADO
display(by_state)

fig, ax = plt.subplots(figsize=(10, 8))
sns.barplot(data=by_state, x="pedidos", y="state_name",
            hue="muestra_chica", dodge=False, palette={False: "steelblue", True: "lightgray"}, ax=ax)
ax.set(title=f"Pedidos entregados por estado del cliente (2017-01 a 2018-08, n = {n:,})",
       xlabel="Cantidad de pedidos", ylabel="Estado")
ax.legend(title=f"< {MIN_PEDIDOS_ESTADO} pedidos")
plt.tight_layout(); plt.show()
```

> Mostrar **los 27 estados** (gris los de muestra chica) en vez de cortar en top 15, para no esconder la población.

### 5.2 P1: facturación media por estado, con IC

```python
p1_ok = p1[~p1["flag_estado_muestra_chica"]]
orden = p1_ok.groupby("state_name")["facturacion"].median().sort_values(ascending=False).index
plt.figure(figsize=(10, 8))
sns.pointplot(data=p1_ok, y="state_name", x="facturacion", order=orden,
              estimator=np.median, errorbar=("ci", 95), linestyle="none")
plt.title(f"Facturación mediana por pedido y estado (IC 95 % bootstrap)\n"
          f"Población: pedidos entregados 2017-01 a 2018-08 en estados con ≥ {MIN_PEDIDOS_ESTADO} pedidos")
plt.xlabel("Facturación por pedido (R$, solo mercadería)"); plt.ylabel("Estado")
plt.tight_layout(); plt.show()
```

> **Lectura sugerida:** los estados del Norte/Nordeste tienen ticket mayor, pero con IC más anchos. **Decisión:** no rankear estados cuyos IC se solapan. **Límite:** un ticket mayor puede reflejar el mix de categorías y no poder adquisitivo.

### 5.3 P2: costo de envío por estado (absoluto y relativo)

```python
p1 = p1.assign(flete_sobre_precio=lambda d: d["costo_envio"] / d["facturacion"])
fig, axes = plt.subplots(1, 2, figsize=(15, 8), sharey=True)
orden = p1_ok.groupby("state_name")["costo_envio"].median().sort_values(ascending=False).index
sns.boxplot(data=p1[~p1["flag_estado_muestra_chica"]], y="state_name", x="costo_envio",
            order=orden, showfliers=False, ax=axes[0])
axes[0].set(title="Flete por pedido", xlabel="R$")
sns.boxplot(data=p1[~p1["flag_estado_muestra_chica"]], y="state_name", x="flete_sobre_precio",
            order=orden, showfliers=False, ax=axes[1])
axes[1].set(title="Flete / precio del pedido", xlabel="Proporción")
fig.suptitle("Costo de envío por estado del cliente. Población: P1 en estados con ≥ 100 pedidos (sin outliers en el dibujo)")
plt.tight_layout(); plt.show()
```

### 5.4 P3: atraso y satisfacción (gráfico nuevo)

```python
p3 = orders_curated[orders_curated["en_poblacion_p3"]]
fig, axes = plt.subplots(1, 2, figsize=(14, 5))
(p3.groupby("customer_state")["entregado_tarde"].mean().sort_values(ascending=False) * 100).plot.bar(ax=axes[0])
axes[0].set(title=f"% de pedidos entregados tarde por estado (n = {len(p3):,})", ylabel="% tarde", xlabel="Estado")
p3.groupby("entregado_tarde")["review_score"].value_counts(normalize=True).unstack().T.plot.bar(ax=axes[1])
axes[1].set(title="Distribución de review_score: a tiempo vs. tarde", xlabel="Puntaje (ordinal)", ylabel="Proporción")
plt.tight_layout(); plt.show()
```

> **Lectura:** se espera una proporción mucho mayor de puntajes 1 cuando el pedido llega tarde. **Límite:** es una asociación, no causalidad (puede haber productos de baja calidad que además llegan tarde).

---

## 6. Controles ejecutables (sección nueva)

Lo pidió el profe ("todavía no hay; empiecen con assert sobre claves únicas y FK") y la consigna ("capaces de fallar"). Todos se agrupan en **una función** para poder reusarla en el desafío.

```python
def controles(d: dict[str, pd.DataFrame], curada: pd.DataFrame | None = None) -> pd.DataFrame:
    """Ejecuta todos los controles. Devuelve un reporte y no corta (para el desafío).
    Usar `assert_controles` para la versión que falla."""
    r = []
    def chk(nombre, cond_ok, detalle=""):
        r.append({"control": nombre, "ok": bool(cond_ok), "detalle": detalle})

    # Claves únicas
    chk("PK orders.order_id", d["orders"]["order_id"].is_unique)
    chk("PK customers.customer_id", d["customers"]["customer_id"].is_unique)
    chk("PK products.product_id", d["products"]["product_id"].is_unique)
    chk("PK sellers.seller_id", d["sellers"]["seller_id"].is_unique)
    chk("PK order_items(order_id, order_item_id)",
        not d["order_items"].duplicated(["order_id", "order_item_id"]).any())
    chk("PK payments(order_id, payment_sequential)",
        not d["order_payments"].duplicated(["order_id", "payment_sequential"]).any())

    # Claves foráneas
    for hijo, col, padre, pcol in [
        ("orders", "customer_id", "customers", "customer_id"),
        ("order_items", "order_id", "orders", "order_id"),
        ("order_items", "product_id", "products", "product_id"),
        ("order_items", "seller_id", "sellers", "seller_id"),
        ("order_payments", "order_id", "orders", "order_id"),
        ("order_reviews", "order_id", "orders", "order_id"),
    ]:
        huerfanos = (~d[hijo][col].isin(d[padre][pcol])).sum()
        chk(f"FK {hijo}.{col} → {padre}", huerfanos == 0, f"{huerfanos} huérfanos")

    # Validez
    chk("price > 0", (d["order_items"]["price"] > 0).all(),
        f"{(d['order_items']['price'] <= 0).sum()} inválidos")
    chk("freight_value >= 0", (d["order_items"]["freight_value"] >= 0).all())
    chk("review_score en 1..5", d["order_reviews"]["review_score"].between(1, 5).all())
    chk("order_status en dominio", d["orders"]["order_status"].isin(STATUS_VALIDOS).all())
    chk("customer_state en dominio", d["customers"]["customer_state"].isin(ESTADOS_VALIDOS).all())
    ts = pd.to_datetime(d["orders"]["order_purchase_timestamp"], errors="coerce")
    chk("purchase parseable y en rango", ts.notna().all() and ts.between("2016-09-01", "2018-10-31 23:59:59").all())

    # Completitud con significado
    od = d["orders"]
    del_sin_fecha = ((od["order_status"] == "delivered") & od["order_delivered_customer_date"].isna()).mean()
    chk("delivered sin fecha entrega < 0,1 %", del_sin_fecha < 0.001, f"{del_sin_fecha:.3%}")

    # Grano de la curada
    if curada is not None:
        chk("curada: 1 fila por pedido", curada["order_id"].is_unique and len(curada) == len(d["orders"]),
            f"{len(curada)} vs {len(d['orders'])}")
    return pd.DataFrame(r)

def assert_controles(d, curada=None):
    rep = controles(d, curada)
    fallidos = rep[~rep["ok"]]
    assert fallidos.empty, f"Controles fallidos:\n{fallidos.to_string()}"
    return rep

assert_controles(raw, orders_curated)
```

---

## 7. Desafío: casos que violan reglas (sección nueva)

La consigna pide **al menos 3 reglas violadas** y explicar **qué detecta y qué no detecta** cada control.

```python
import copy

def inyectar(raw):
    bad = {k: v.copy() for k, v in raw.items()}
    # Caso 1: order_id duplicado (unicidad)
    bad["orders"] = pd.concat([bad["orders"], bad["orders"].iloc[[0]]], ignore_index=True)
    # Caso 2: ítem con precio negativo (validez)
    bad["order_items"].loc[0, "price"] = -50.0
    # Caso 3: ítem que referencia un producto inexistente (integridad)
    bad["order_items"].loc[1, "product_id"] = "x" * 32
    # Caso 4: fecha de compra fuera de rango (validez temporal)
    bad["orders"].loc[2, "order_purchase_timestamp"] = "2030-01-01 00:00:00"
    # Caso 5: estado inexistente (dominio)
    bad["customers"].loc[3, "customer_state"] = "XX"
    # Caso 6 (NO detectable): precio plausible pero erróneo (×10)
    bad["order_items"].loc[4, "price"] = bad["order_items"].loc[4, "price"] * 10
    # Caso 7 (NO detectable): fecha de entrega anterior a la compra en UN pedido
    bad["orders"].loc[5, "order_delivered_customer_date"] = "2016-01-01 00:00:00"
    return bad

bad = inyectar(raw)
reporte = controles(bad)
reporte[~reporte["ok"]]
```

```python
# Verificar que el contrato Pandera también falla
try:
    assert_controles(bad)
except AssertionError as e:
    print("✔ El flujo falló como se esperaba\n", str(e)[:800])
```

Tabla para el informe (completar con la salida):

| Caso inyectado | Regla violada | ¿Lo detecta? | Control que lo detecta | Qué **no** detecta |
|---|---|---|---|---|
| `order_id` duplicado | Unicidad PK | ✔ | `PK orders.order_id` | No dice cuál de las dos filas es la correcta. |
| `price = −50` | Validez | ✔ | `price > 0` | Un precio positivo pero erróneo (caso 6) pasa. |
| `product_id` inexistente | Integridad FK | ✔ | `FK order_items.product_id` | Un `product_id` válido pero equivocado (otro producto existente) pasa. |
| Compra en 2030 | Rango temporal | ✔ | `purchase en rango` | Una fecha dentro del rango pero corrida unos días pasa. |
| Estado `XX` | Dominio | ✔ | `customer_state en dominio` | Un estado válido pero equivocado (SP en vez de RJ) pasa. |
| Precio ×10 | Exactitud | ✘ | — | Requeriría comparar contra el precio histórico del producto (control estadístico, z-score por `product_id`). |
| Entrega < compra en 1 pedido | Orden temporal | ✘ con umbral | La regla de secuencia es un **flag**, no un `assert` (ya hay violaciones reales) | Detectarlo exige un umbral: "no más de N violaciones" o "% no supera el baseline". Proponer ese control como mejora. |

> Incluir los casos **no detectados** a propósito es lo que hace "Destacado" este punto: muestra que entienden los límites de cada control.

---

## 8. Cierre en el notebook (markdown final)

1. **Respuestas a P1, P2 y P3** en 3–4 líneas cada una, con números.
2. **Usos habilitados / con revisión / no recomendados** (mismo texto que §1.6).
3. **Limitaciones:** muestra de un solo marketplace, anonimización, ventana temporal y ausencia de datos de promociones/vouchers.
4. **Declaración de IA** (misma tabla que §1.7).
5. **Pendientes para el entregable 2** (ver §10).

---

## 9. Checklist final antes de entregar

- [ ] *Restart & Run All* sin errores en un entorno limpio (Colab **y** local).
- [ ] `requirements.txt`, `README.md` y `.gitignore` en el repo; **ningún CSV crudo commiteado**.
- [ ] `raw` nunca se modifica (buscar `inplace=True` y `dfs[...] =`; no debe quedar ninguno sobre raw).
- [ ] Todos los `merge` tienen `validate=` y la bitácora muestra 99.441 en cada paso.
- [ ] Cada gráfico tiene título, unidades, población (n) y una celda de lectura.
- [ ] Hipótesis H1–H9 contrastadas una por una (y las 2 principales destacadas).
- [ ] Perfil de calidad cubre las 7 dimensiones de la consigna.
- [ ] Contrato Pandera pasa sobre la curada y falla sobre `bad`.
- [ ] Desafío con ≥ 3 reglas violadas y tabla de "qué detecta / qué no".
- [ ] Informe ≤ 6 páginas (sin referencias/anexos), con preguntas reformuladas, resultados y figuras.
- [ ] Declaración de IA en informe y notebook.
- [ ] Referencias: Kaggle (URL), pandas, Pandera, seaborn/matplotlib, fuentes de eventos externos (Black Friday, paro de camioneros 2018, BCB si se usa).
- [ ] Cada integrante puede explicar cualquier celda (Parcial 1 y Prueba Final).

---

## 10. Anotado para el entregable 2 (no hacer ahora)

- **Feature engineering** ("IMPORTANTE" en el feedback 2): distancia vendedor–cliente (haversine con geolocation), día de la semana/hora de compra, flags de feriados, ticket relativo a la categoría, recurrencia por `customer_unique_id`, densidad de producto (peso/volumen), largo y sentimiento del comentario, etc.
- **Geoespacial micro:** agregar `geolocation` por `zip_code_prefix` (mediana de lat/long), mapas por ciudad, flujos vendedor → cliente y outliers de coordenadas fuera de Brasil.
- **Normalización de ciudades** cruzando `customers`/`sellers`/`geolocation`.
- **Control estadístico** para detectar precios anómalos por producto (el caso 6 del desafío).
