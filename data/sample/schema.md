# Esquema de datos

El pipeline escribe cuatro hojas en el Google Sheet `Procesamiento`. Esta es la descripción de cada una.

**Convenciones**
- Los rendimientos y la inflación se guardan como **fracción decimal** (`0.1656` equivale a 16,56%). El formato porcentual se aplica solo en Looker Studio.
- Las fechas de las hojas de control se guardan como texto ISO 8601 (por ejemplo, `2026-09-16T04:53:12.387881`), con la hora del entorno de ejecución (en Colab, normalmente UTC).

---

## `Rendimiento_Consolidado`

Tabla principal del análisis. Contiene todos los instrumentos comparables contra la inflación argentina (los que están en pesos y las acciones de EE.UU. convertidas a pesos con el dólar MEP). Una fila por instrumento y ventana de tiempo.

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | entero | Identificador de la fila. |
| `categoria` | texto | `plazo_fijo`, `fci`, `cedear`, `accion_local_ar`, `on` o `accion_usa`. |
| `simbolo` | texto | Ticker o nombre del instrumento (por ejemplo, `AAPL`, `Depósito 30 días`, `Cocos Ahorro`). |
| `periodo` | texto | Ventana: `1 mes`, `3 meses`, `6 meses` o `12 meses`. |
| `dias` | entero | Días de la ventana: 30, 90, 180 o 365. |
| `rendimiento_periodo` | decimal (fracción) | Rendimiento nominal del instrumento en la ventana, en pesos. |
| `instrumento_dolarizado` | booleano | `True` si el instrumento cotiza en dólares y se convirtió a pesos con el dólar MEP. `False` en el resto. |
| `inflacion_periodo` | decimal (fracción) | Inflación acumulada de la misma ventana. |
| `rendimiento_real` | decimal (fracción) | `(1 + rendimiento_periodo) / (1 + inflacion_periodo) − 1`. |
| `orden_periodo` | entero | Orden de la ventana (1 a 4) para ordenar los gráficos del dashboard. |

**Notas**
- Si un instrumento no existía en una ventana (por ejemplo, una ON emitida hace pocos meses), esa fila no se genera y el motivo queda registrado en `_detalle_errores`.
- Para `accion_usa`, `rendimiento_periodo` ya incluye el efecto del dólar MEP.

---

## `Acciones_USA_USD`

Tabla informativa con el rendimiento de las acciones de EE.UU. en **dólares**, sin conversión. No se compara contra la inflación argentina.

| Columna | Tipo | Descripción |
|---|---|---|
| `categoria` | texto | Siempre `accion_usa`. |
| `simbolo` | texto | Ticker (por ejemplo, `SPY`, `AMZN`, `TSLA`, `NVDA`). |
| `periodo` | texto | `1 mes`, `3 meses`, `6 meses` o `12 meses`. |
| `dias` | entero | 30, 90, 180 o 365. |
| `rendimiento_periodo_usd` | decimal (fracción) | Rendimiento del precio en dólares en la ventana. |
| `orden_periodo` | entero | Orden de la ventana (1 a 4). |

---

## `_pipeline_log`

Una fila por ejecución del pipeline.

| Columna | Tipo | Descripción |
|---|---|---|
| `fecha_hora` | texto (ISO 8601) | Momento de la ejecución. |
| `funcion` | texto | Nombre del pipeline ejecutado. |
| `fuente` | texto | Fuente de datos. `multiple` cuando el pipeline combina varias. |
| `filas_procesadas` | entero | Filas escritas en la tabla consolidada. En una corrida sin errores, 92. |
| `estado` | texto | `OK`, `OK_CON_ERRORES` o `ERROR`. |
| `detalle` | texto | Cantidad de errores (por ejemplo, `0 errores`) o el mensaje de la excepción si la corrida falló. |

---

## `_detalle_errores`

Una fila por ítem que no pudo procesarse. Si una corrida no tiene errores, no se escribe nada.

| Columna | Tipo | Descripción |
|---|---|---|
| `fecha_hora` | texto (ISO 8601) | Momento del error. |
| `funcion` | texto | Función donde ocurrió. |
| `item` | texto | Instrumento afectado (por ejemplo, `KO`). |
| `tipo_error` | texto | Valores observados: `sin_datos` (la fuente devolvió vacío) y `sin_historia_suficiente` (el instrumento no existía en esa ventana). |
| `detalle` | texto | Descripción legible del problema. |

---

## Relación entre las hojas

```
Fuentes (API) ──► Pipeline ──► Rendimiento_Consolidado ──┐
                     │                                   ├──► Looker Studio
                     ├──────► Acciones_USA_USD ──────────┘
                     └──────► _pipeline_log
                              _detalle_errores
```
