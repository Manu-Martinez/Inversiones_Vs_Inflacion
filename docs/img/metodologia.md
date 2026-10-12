# Metodología

Este documento explica cómo se calcula cada indicador del proyecto y qué decisiones se tomaron en el camino.

## Objetivo

Medir cuánto rindió, en términos **reales** (descontando inflación), cada alternativa de ahorro e inversión disponible para un argentino, en ventanas de 1, 3, 6 y 12 meses.

## Ventanas de análisis

- 1 mes (30 días), 3 meses (90 días), 6 meses (180 días) y 12 meses (365 días).
- Las mismas ventanas se aplican a todas las categorías, para que sean comparables entre sí.

## Rendimiento real

El rendimiento real se calcula con la ecuación de Fisher:

```
rendimiento real = (1 + rendimiento nominal) / (1 + inflación del período) − 1
```

La inflación del período es la inflación acumulada de la misma ventana, a partir de la serie mensual de ArgentinaDatos.

**Ejemplo (tomado del dashboard):** una ON con rendimiento nominal de 19,95% a 12 meses y una inflación acumulada de 33,56% tiene un rendimiento real de `1,1995 / 1,3356 − 1 = −10,19%`.

## Cómo se calcula el rendimiento nominal por categoría

- **Plazo fijo a 30 días:** a partir de la serie de tasas de depósitos a 30 días del BCRA, publicada por ArgentinaDatos.
- **Fondos comunes de mercado de dinero (FCI):** a partir de la TNA publicada por CAFCI para cada ventana: `rendimiento del período = TNA × días / 365`. Se analizan tres fondos con nombre de marca reconocible: Cocos Ahorro, Mercado Fondo y Ualintec Ahorro Pesos.
- **CEDEARs y acciones locales:** variación del precio en pesos entre el inicio y el fin de la ventana, con datos de yfinance (tickers con sufijo `.BA`).
- **Obligaciones Negociables (ONs):** variación del `ultimoPrecio` entre el inicio y el fin de la ventana, con la serie histórica de la API de IOL.
- **Acciones de EE.UU.:** variación del precio en dólares con datos de yfinance. Ver la sección siguiente para la conversión a pesos.

## Dos ejes de análisis: pesos y dólares

Un rendimiento en dólares no es comparable contra la inflación en pesos. Por eso el proyecto separa dos ejes:

1. **Instrumentos en pesos** (plazo fijo, FCI, CEDEARs, acciones locales, ONs): se comparan directamente contra la inflación argentina.
2. **Acciones de EE.UU. en dólares:** se muestran con su rendimiento en USD, como tabla informativa.

Para incluir a las acciones de EE.UU. en la comparación contra inflación, se convierten a pesos con el dólar MEP del mismo período:

```
rendimiento en pesos = (1 + rendimiento en USD) × (1 + variación del dólar MEP) − 1
```

El dólar MEP se toma de la serie histórica de ArgentinaDatos. Las filas convertidas se marcan con el indicador `instrumento_dolarizado = True`, para distinguirlas de los instrumentos genuinamente en pesos.

## Instrumentos sin historia suficiente

Para las ONs, se busca el dato más cercano a la fecha de inicio de la ventana. Si ese dato está a **más de 15 días** de la fecha objetivo, el instrumento no existía en esa ventana y se registra sin dato (`NaN`). Esto evita anclar el cálculo al primer valor disponible, que daría un número engañoso.

Por ejemplo, una ON emitida hace 8 meses no tiene rendimiento a 12 meses.

## Convención de unidades

- Todos los rendimientos se guardan como **fracción decimal** (`0.1656`, no `16.56`).
- El formato porcentual se aplica solo en la capa de presentación (Looker Studio), para mantener una única fuente de verdad y evitar errores de unidades entre fuentes.

## Tablas de salida

El pipeline escribe cuatro hojas en Google Sheets:

- `Rendimiento_Consolidado`: todos los instrumentos comparables contra la inflación, con rendimiento nominal, inflación del período y rendimiento real.
- `Acciones_USA_USD`: rendimiento de las acciones de EE.UU. en dólares, solo informativo.
- `_pipeline_log`: una fila por ejecución, con estado y cantidad de errores.
- `_detalle_errores`: una fila por ítem que no pudo procesarse.

Las columnas y los tipos de cada hoja están en [`data/schema.md`](../data/schema.md).

## Indicador "superó / no superó la inflación"

En el Resumen ejecutivo, el gráfico de instrumentos por período cuenta cuántos instrumentos tuvieron un rendimiento real positivo (superaron la inflación) y cuántos negativo, en cada ventana.

## Decisiones de fuentes

- **CAFCI:** la API anterior, que usaba ArgentinaDatos, dejó de funcionar en abril de 2026. Se migró a los endpoints nuevos de CAFCI, que requieren un `User-Agent` de navegador.
- **ONs: de IAMC a IOL.** Los informes diarios de IAMC solo ofrecían historia desde junio de 2026 y el sitio restringe el acceso automatizado, por lo que se descartó. La API de IOL ofrece series históricas completas.
- **Sin activos duplicados:** un mismo activo no aparece en dos categorías con significados distintos (por ejemplo, KO figura solo como CEDEAR).

## Limitaciones

- **Muestra reducida:** entre 3 y 5 instrumentos por categoría, elegidos por representatividad y disponibilidad de datos. No representa el universo completo.
- **ONs sin cupones:** el rendimiento usa el precio sin ajustar, por lo que no incluye los cupones cobrados y subestima el retorno total.
- **Fondos:** el rendimiento se deriva de la TNA publicada, no de la variación del valor de la cuotaparte.
- **Un solo período:** los resultados describen una ventana puntual, no un ciclo completo. El rendimiento pasado no anticipa el futuro.
- **Rendimientos brutos:** no se descuentan impuestos ni comisiones.
- **Inflación general:** se usa el índice agregado, no la canasta de cada persona.
- **Dólar MEP:** la conversión de las acciones de EE.UU. depende de esa cotización y no refleja otros tipos de cambio.
