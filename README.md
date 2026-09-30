# Laboratorio #1 – Derivados Financieros
## Portafolio LLY · ORCL · GD y cobertura del 85% con opciones listadas

Inversión de **USD 10 millones** a **2 años** sobre un portafolio de tres acciones del mercado estadounidense, con datos desde el **01/10/2021**, tasa de dividendos de cada empresa y un análisis de cobertura al **85%** mediante estrategias con opciones y puntos de control trimestrales.

**Fecha de valoración:** 25/09/2026 · **Vencimiento de las opciones:** 21/01/2028 · **Notebook:** [`Laboratorio1_Derivados.ipynb`](Laboratorio1_Derivados.ipynb)

---

## Contenido

1. [Resumen ejecutivo y tesis de mercado](#1-resumen-ejecutivo-y-tesis-de-mercado)
2. [Fuentes y preparación de los datos](#2-fuentes-y-preparación-de-los-datos)
3. [Construcción del portafolio media-varianza](#3-construcción-del-portafolio-media-varianza)
4. [Simulación y métricas de riesgo](#4-simulación-y-métricas-de-riesgo)
5. [Selección de opciones y criterios de liquidez](#5-selección-de-opciones-y-criterios-de-liquidez)
6. [Valoración binomial europea y americana](#6-valoración-binomial-europea-y-americana)
7. [Diseño de coberturas](#7-diseño-de-coberturas)
8. [Seguimiento trimestral y comparación cubierto/no cubierto](#8-seguimiento-trimestral-y-comparación-cubiertono-cubierto)
9. [Conclusiones, limitaciones y recomendación final](#9-conclusiones-limitaciones-y-recomendación-final)
10. [Anexo de reproducibilidad](#10-anexo-de-reproducibilidad)

---

## 1. Resumen ejecutivo y tesis de mercado

Presentamos el análisis de una inversión de USD 10 millones durante un horizonte de dos años, distribuida en tres acciones del mercado estadounidense: Eli Lilly (LLY), Oracle (ORCL) y General Dynamics (GD). El portafolio se construye con el modelo de media-varianza, con una restricción de participación entre 15% y 50% por acción, y la configuración final es **33,1% en LLY, 16,9% en ORCL y 50,0% en GD**.

**Tesis de mercado.** Los tres negocios responden a motores distintos (farmacéutica, software/nube y defensa), con correlaciones bajas (0,13–0,19), lo que justifica una cartera diversificada de largo. La tesis es alcista moderada, pero con dudas sobre la repetición del crecimiento extraordinario de LLY (+34% anual en la muestra): por eso no se elige el máximo Sharpe, que depende de esa extrapolación, sino la mínima varianza. Además, la volatilidad implícita de las opciones está cerca del nivel de largo plazo del GARCH y por encima de la volatilidad actual: el mercado no espera que la calma de hoy dure, lo que refuerza la necesidad de un piso de protección.

**Costos de la cobertura.** La estrategia A (*Protective Put*) requiere una prima neta de USD 925.290, equivalente al 9,25% del portafolio. La estrategia B (*Collar*) reduce este desembolso inicial a USD 277.275 (2,77%), porque la venta de las *calls* financia parte de la prima de las *puts*. La estrategia C (*bear put spread* sobre LLY) reduce el costo de proteger ese activo de USD 357.240 a USD 232.320, una reducción cercana al 35%.

Nuestra estrategia completa C tiene un costo de USD 800.370 (8,00% del portafolio). Reduce el costo de proteger LLY, pero deja de proteger caídas superiores a aproximadamente 25% del spot (put vendida en 890), por lo que se acepta una mayor exposición en escenarios extremos a cambio de disminuir el costo de la cobertura. Bajo los supuestos del modelo es la estrategia más eficiente: **USD 1,50 de reducción del VaR99 por cada dólar de costo económico**.

## 2. Fuentes y preparación de los datos

Los precios diarios de LLY, ORCL y GD se obtuvieron de *Yahoo Finance* mediante la librería *yfinance* (v1.7.0) de Python, para el periodo del 1 de octubre de 2021 al 25 de septiembre de 2026 (último cierre completo). Descarga realizada el 27/09/2026 a las 07:21 UTC.

| Ticker | Empresa | Sector | Industria | Mercado | Observaciones | Faltantes |
|---|---|---|---|---|---:|---:|
| LLY | Eli Lilly and Company | Healthcare | Drug Manufacturers – General | NYSE | 1.251 | 0 |
| ORCL | Oracle Corporation | Technology | Software – Infrastructure | NYSE | 1.251 | 0 |
| GD | General Dynamics Corporation | Industrials | Aerospace & Defense | NYSE | 1.251 | 0 |

**Decisiones de datos**

- **Adj Close** (ajustado por dividendos y *splits*) para los retornos, porque mide el retorno total que recibe el accionista.
- **Close** (solo ajustado por *splits*) como spot de las opciones, porque los contratos se liquidan contra el precio de mercado. Los dividendos se tratan aparte con la tasa $q$; mezclarlos contaría dos veces el mismo dinero y subvaloraría las puts.
- **Retornos logarítmicos** $r_t = \ln(P_t/P_{t-1})$: son aditivos en el tiempo, son la variable que modela el MGB ($\ln S$ es normal) y la entrada natural del GARCH.
- **Faltantes:** *forward fill* de máximo 3 días hábiles y eliminación de fechas sin dato común. En la muestra no hubo faltantes: 1.251 fechas utilizadas y 1.250 retornos.

**Tasa libre de riesgo por plazo (FRED, 24/09/2026).** Los rendimientos Treasury (BEY) se convierten a tasa continua $r_{cc} = 2\ln(1+y/2)$ y se interpolan para el vencimiento exacto de cada opción.

| Plazo | Serie FRED | Rendimiento (BEY) | Tasa continua |
|---|---|---:|---:|
| 3 meses | DGS3MO | 4,24% | 4,20% |
| 6 meses | DGS6MO | 4,34% | 4,29% |
| 1 año | DGS1 | 4,51% | 4,46% |
| 2 años | DGS2 | 4,87% | **4,81%** (Sharpe) |

**Dividendos (últimos 12 meses).**

| Ticker | Spot $S_0$ (USD) | Dividendos 12m por acción | $q$ continua |
|---|---:|---:|---:|
| LLY | 1.183,46 | 6,69 | 0,56% |
| ORCL | 137,10 | 2,00 | 1,45% |
| GD | 336,72 | 6,18 | 1,82% |

## 3. Construcción del portafolio media-varianza

**Estadísticas anualizadas por activo** ($\mu_i = 252\,\bar r_i$, $\Sigma = 252\,\text{Cov}(r)$):

| Ticker | Retorno anual | Volatilidad anual | Sharpe | Asimetría | Curtosis exceso |
|---|---:|---:|---:|---:|---:|
| LLY | 33,98% | 32,61% | 0,894 | 0,16 | 7,88 |
| ORCL | 9,90% | 42,93% | 0,119 | 1,24 | 17,48 |
| GD | 12,80% | 20,73% | 0,385 | 0,08 | 5,21 |

**Matriz de correlaciones:**

| | LLY | ORCL | GD |
|---|---:|---:|---:|
| LLY | 1,000 | 0,133 | 0,194 |
| ORCL | 0,133 | 1,000 | 0,172 |
| GD | 0,194 | 0,172 | 1,000 |

Se evaluaron tres configuraciones (pesos iguales, mínima varianza y máximo Sharpe) y un análisis de sensibilidad del máximo Sharpe sin el tope de 50%. El piso de 15% es la regla del laboratorio; el tope de 50% es una política de concentración del equipo.

$$\min_w\; w^\top \Sigma w \quad \text{s.a.}\quad \textstyle\sum_i w_i = 1,\;\; 0{,}15 \le w_i \le 0{,}50$$

| Configuración | w LLY | w ORCL | w GD | Retorno | Volatilidad | Sharpe | Máx. contribución al riesgo |
|---|---:|---:|---:|---:|---:|---:|---:|
| Pesos iguales | 33,3% | 33,3% | 33,3% | 18,89% | 21,80% | 0,646 | 51,03% |
| **Mínima varianza** | **33,1%** | **16,9%** | **50,0%** | **19,33%** | **19,15%** | **0,758** | **40,59%** |
| Máximo Sharpe | 50,0% | 15,0% | 35,0% | 22,95% | 21,21% | 0,855 | 67,30% |
| Máx. Sharpe sin tope 50% | 69,8% | 15,0% | 15,2% | 27,14% | 25,35% | 0,881 | 85,71% |

![Frontera eficiente](Laboratorio%201%20-%20docs/fig1_frontera_eficiente.png)

El máximo Sharpe obtiene un retorno esperado de 22,95% frente al 19,33% de la mínima varianza, pero con una volatilidad de 21,21% y una concentración del 67,30% del riesgo en LLY; sin el tope de 50% la concentración llega a 85,71%. La mínima varianza reduce la volatilidad en 2,06 puntos porcentuales y la máxima concentración en 26,71 puntos, a cambio de un menor retorno esperado.

Se renuncia a ~0,1 de Sharpe por tres razones: (1) el máximo Sharpe depende de creer que LLY repetirá sus rendimientos, y el retorno esperado es el insumo más ruidoso de Markowitz; (2) la mínima varianza solo usa volatilidades y correlaciones, que son más estables; (3) sobrepondera GD, la acción menos volátil, lo que abarata la cobertura porque la prima de una put crece con la volatilidad del subyacente.

**Contribución al riesgo** $RC_i = w_i\,(\Sigma w)_i/\sigma_p$:

| Ticker | Peso | Vol. individual | % del riesgo | Riesgo / peso |
|---|---:|---:|---:|---:|
| **LLY** | 33,1% | 32,6% | **40,6%** | 1,23 |
| ORCL | 16,9% | 42,9% | 20,7% | 1,23 |
| GD | 50,0% | 20,7% | 38,7% | 0,77 |

**LLY domina el riesgo**: aporta 40,6% de la volatilidad con 33,1% del capital. El beneficio de diversificación es claro: la volatilidad ponderada sería 28,41%, frente a 19,15% del portafolio.

![Correlaciones y contribución al riesgo](Laboratorio%201%20-%20docs/fig2_correlaciones_riesgo_precios.png)

**Del peso a las acciones y contratos.** Se compran acciones enteras y se dimensionan contratos de 100 acciones:

| Ticker | Acciones | USD invertido | Contratos (85%) | Cobertura efectiva |
|---|---:|---:|---:|---:|
| LLY | 2.799 | 3.312.504 | 24 | 85,74% |
| ORCL | 12.303 | 1.686.741 | 105 | 85,35% |
| GD | 14.849 | 4.999.955 | 126 | 84,85% |

Invertido: USD 9.999.201; efectivo residual: USD 799.

**Criterios financieros iniciales aplicados**

| Criterio | Implementación |
|---|---|
| "Pago trimestral" | Puntos de control trimestrales: las opciones se revaloran a mercado con una regla mantener / ajustar / monetizar / renovar. |
| Tasa por plazo | Curva Treasury 3M–2Y; cada opción usa la tasa de su vencimiento (4,57% a 1,32 años); el Sharpe usa la de 2 años. |
| Portafolio como subyacente | Contratos activo por activo (85% de sus acciones) y luego se agrega el P&L al portafolio. |
| Europea vs. americana | Árbol CRR con los mismos $S_0, K, T, \sigma, r, q$. |
| Sin apalancamiento | La prima se paga con caja propia, fuera de los USD 10 M; el P&L cubierto se mide contra acciones + prima y se reporta el costo de oportunidad de esa caja. |

## 4. Simulación y métricas de riesgo

**Varianza GARCH(1,1) con 10 años de historia.** $h_t = \omega + \alpha\,\varepsilon_{t-1}^2 + \beta\,h_{t-1}$, estimado por máxima verosimilitud con innovaciones *t* y normal; se elige la de menor AIC entre las estacionarias. En ORCL la versión *t* sale con $\alpha+\beta = 1{,}00$ (IGARCH, choque permanente), por lo que se usa la normal.

| Ticker | Distribución | α | β | α+β | ν | Vida media choque (días) | Vol. largo plazo | Vol. hoy |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| LLY | t | 0,108 | 0,861 | 0,970 | 3,36 | 22 | 37,5% | 25,8% |
| ORCL | normal | 0,147 | 0,830 | 0,977 | – | 30 | 50,5% | 45,8% |
| GD | t | 0,060 | 0,918 | 0,978 | 4,65 | 31 | 22,7% | 20,9% |

La dependencia entre activos se estima sobre los residuos estandarizados (CCC-GARCH): correlaciones de 0,17 a 0,26.

![Volatilidad GARCH](Laboratorio%201%20-%20docs/fig3_volatilidad_garch.png)

**MGB correlacionado con varianza GARCH por trayectoria.** 10.000 trayectorias diarias (504 días hábiles), choques correlacionados por Cholesky con marginales *t* del GARCH y dividendos acumulados como caja:

$$\ln S_{i,t+1} = \ln S_{i,t} + \Big(\tfrac{\mu_i + \tfrac12\sigma_i^2 - q_i}{252} - \tfrac12 h_{i,t}\Big) + \sqrt{h_{i,t}}\,z_{i,t}$$

**Valor del portafolio en los cierres trimestrales (USD):**

| Cierre | Fecha aprox. | P5 | P25 | Mediana | P75 | P95 | Prob. pérdida |
|---|---|---:|---:|---:|---:|---:|---:|
| t=0 | 2026-09-25 | 9.999.201 | 9.999.201 | 9.999.201 | 9.999.201 | 9.999.201 | 0,0% |
| T1 | 2026-12-23 | 9.015.866 | 9.914.791 | 10.560.580 | 11.247.617 | 12.459.927 | 28,0% |
| T2 | 2027-03-22 | 8.796.188 | 10.173.132 | 11.162.078 | 12.244.807 | 14.119.071 | 21,4% |
| T3 | 2027-06-17 | 8.881.745 | 10.511.311 | 11.793.212 | 13.234.349 | 15.789.957 | 16,5% |
| T4 | 2027-09-14 | 8.906.388 | 10.883.701 | 12.432.366 | 14.195.183 | 17.611.798 | 13,7% |
| T5 | 2027-12-10 | 8.988.215 | 11.328.717 | 13.147.617 | 15.287.627 | 19.368.974 | 11,5% |
| T6 | 2028-03-08 | 9.108.487 | 11.739.774 | 13.903.453 | 16.413.360 | 21.322.357 | 9,8% |
| T7 | 2028-06-05 | 9.264.977 | 12.267.260 | 14.645.112 | 17.620.445 | 23.440.985 | 8,2% |
| T8 | 2028-08-31 | 9.472.086 | 12.703.165 | 15.496.476 | 18.887.237 | 25.819.885 | 7,0% |

![Bandas percentiles](Laboratorio%201%20-%20docs/fig4_bandas_percentiles.png)

Son **escenarios, no un pronóstico**: a 2 años la mediana es USD 15,50 millones (+55,0%), pero 1 de cada 20 escenarios termina por debajo de USD 9,47 millones y la probabilidad de perder capital es 7,0%.

**VaR y distribución de pérdidas sin cobertura** ($\text{VaR}_{95} = -Q_{0.05}(P\&L)$, $\text{VaR}_{99} = -Q_{0.01}(P\&L)$):

| Horizonte | VaR95 | VaR99 | ES99 | VaR95 % | VaR99 % |
|---|---:|---:|---:|---:|---:|
| T1 | 983.335 | 1.694.377 | 2.119.561 | 9,83% | 16,95% |
| Venc. cobertura (1,32 años) | 987.966 | 2.420.902 | 3.206.988 | 9,88% | 24,21% |
| T8 (2 años) | 527.115 | 2.332.366 | 3.128.591 | 5,27% | 23,33% |

![Distribución del P&L y VaR](Laboratorio%201%20-%20docs/fig5_var_no_cubierto.png)

**Sensibilidad a la deriva.** El VaR a 2 años depende mucho de suponer que el pasado se repite (LLY +34% anual). Con una deriva conservadora igual a la tasa libre de riesgo, el VaR95 a 2 años pasa de USD 0,53 M a USD 3,49 M y el VaR99 de USD 2,33 M a USD 4,63 M.

**Benchmark teórico cubierto.** Una protective put ATM a 2 años (Black-Scholes, volatilidad de largo plazo del GARCH) costaría USD 1.185.624 (11,86% del portafolio) y reduciría el VaR99 a T8 en 45,3%. Solo en 12,1% de los escenarios las puts pagarían más que su prima, lo que motiva buscar protección más barata en el mercado.

![VaR cubierto teórico](Laboratorio%201%20-%20docs/fig6_var_cubierto_teorico.png)

**Vista estratégica a 10 años** (4.000 trayectorias): la probabilidad de terminar por debajo de la inversión inicial es 6,5% a 2 años y 0,2% a 10 años.

![Vista a 10 años](Laboratorio%201%20-%20docs/fig7_vista_10_anios.png)

## 5. Selección de opciones y criterios de liquidez

**Vencimiento común.** Se buscó el vencimiento más largo común a los tres activos entre 12 y 24 meses: **21/01/2028** (15,9 meses, $T$ = 1,32 años, $r$ = 4,57%). Es el mismo para los tres activos (diferencia de 0 días); los ~8 meses restantes del horizonte exigen renovar la cobertura.

| Ticker | Vencimientos disponibles en la ventana | Elegido |
|---|---|---|
| LLY | 2027-12-17, 2028-01-21, 2028-06-16 | 2028-01-21 |
| ORCL | 2027-10-15, 2027-12-17, 2028-01-21, 2028-09-15 | 2028-01-21 |
| GD | 2028-01-21 | 2028-01-21 |

**Criterios de selección.** Cotización vigente, *spread* relativo $(Ask-Bid)/Mid \le 20\%$, open interest ≥ 10 y pérdida máxima por acción $(S_0-K+\text{prima})/S_0 \le 20\%$. La compra se hace al *ask*. Se compararon **cinco strikes por activo** (80%–100% del spot); la cadena completa (404 contratos) quedó exportada en CSV.

| Ticker | Strike | K/S0 | Clase | Bid | Ask | Spread | OI | Vol. | IV americana | Pérdida máx. | Liquidez | Eficiencia VaR99 |
|---|---:|---:|---|---:|---:|---:|---:|---:|---:|---:|:---:|---:|
| LLY | 950 | 0,803 | OTM | 68,85 | 79,80 | 14,7% | 251 | 2 | 38,0% | 26,5% | ✅ | 0,23 |
| LLY | 1.010 | 0,853 | OTM | 88,85 | 100,90 | 12,7% | 32 | 5 | 37,8% | 23,2% | ✅ | 0,39 |
| LLY | 1.070 | 0,904 | OTM | – | – | – | 0 | 2 | 38,0% | 19,8% | ❌ | 0,35 |
| **LLY** | **1.120** | **0,946** | **OTM** | **132,50** | **148,85** | **11,6%** | **53** | **10** | **37,4%** | **17,9%** | ✅ | **0,27** |
| LLY | 1.180 | 0,997 | ATM | – | – | – | 0 | 1 | 33,8% | 13,2% | ❌ | 0,46 |
| ORCL | 110 | 0,802 | OTM | 16,30 | 17,10 | 4,8% | 3.254 | 9 | 54,7% | 32,2% | ✅ | 0,27 |
| ORCL | 115 | 0,839 | OTM | 18,65 | 19,30 | 3,4% | 2.465 | 2 | 54,8% | 30,2% | ✅ | 0,30 |
| **ORCL** | **125** | **0,912** | **OTM** | **23,40** | **24,10** | **2,9%** | **3.691** | **10** | **54,4%** | **26,4%** | ✅ | **0,38** |
| ORCL | 130 | 0,948 | OTM | 25,95 | 27,10 | 4,3% | 9.966 | 4 | 54,4% | 24,9% | ✅ | 0,38 |
| ORCL | 135 | 0,985 | ATM | 28,80 | 29,90 | 3,7% | 4.262 | 16 | 54,5% | 23,3% | ✅ | 0,36 |
| GD | 270 | 0,802 | OTM | 6,50 | 10,40 | 46,2% | 51 | 9 | 24,8% | 22,9% | ❌ | 0,98 |
| GD | 290 | 0,861 | OTM | 11,00 | 16,00 | 37,0% | 29 | 12 | 24,7% | 18,6% | ❌ | 0,95 |
| GD | 300 | 0,891 | OTM | 13,50 | 18,50 | 31,2% | 25 | 1 | 24,1% | 16,4% | ❌ | 1,17 |
| **GD** | **320** | **0,950** | **OTM** | **20,50** | **25,00** | **19,8%** | **34** | **1** | **23,4%** | **12,4%** | ✅ | **0,92** |
| GD | 340 | 1,010 | ATM | 29,50 | 34,00 | 14,2% | 213 | 12 | 23,1% | 9,1% | ✅ | 0,80 |

La eficiencia es la reducción del VaR99 del portafolio (cubriendo solo ese activo al 85%) por dólar de prima pagada al *ask*. Entre los strikes que cumplen liquidez y pérdida máxima ≤ 20% se elige el más eficiente. En **ORCL ningún strike cumple el límite de 20%**: con una IV de 54% la prima consume gran parte del colchón, por lo que se relajó el filtro y se tomó el más eficiente (125, marcado como excepción).

**Contratos seleccionados:** LLY put 1.120 (OTM, 94,6%), ORCL put 125 (OTM, 91,2%) y GD put 320 (OTM, 95,0%). Son strikes ligeramente OTM: protegen desde una caída de 5%–9%, sin pagar el valor adicional de la opción ATM.

![Sonrisa de volatilidad](Laboratorio%201%20-%20docs/fig8_sonrisa_iv.png)

La IV se compara entre strikes y contra la volatilidad GARCH de largo plazo; no se elige el strike de menor IV, porque uno muy OTM puede tener IV baja y no proteger nada útil.

## 6. Valoración binomial europea y americana

Árbol CRR de 500 pasos con dividendo continuo: $u=e^{\sigma\sqrt{\Delta t}}$, $d=1/u$, $p=\frac{e^{(r-q)\Delta t}-d}{u-d}$; en la americana, $f=\max\{\text{intrínseco},\;e^{-r\Delta t}[p f_u+(1-p)f_d]\}$ en cada nodo. Ambas usan los mismos $S_0$, $K$, $T$ = 1,32, $\sigma$ = IV americana del contrato, $r$ = 4,57% y $q$. Black-Scholes se reporta como control de convergencia.

| Activo | K | Prima mercado (mid) | BS europea | CRR europea | CRR americana | Ejercicio anticipado* |
|---|---:|---:|---:|---:|---:|---:|
| **LLY** put | 1.120 | 140,675 | 135,290 | 135,375 | **140,762** | 3,8% |
| **ORCL** put | 125 | 23,750 | 23,207 | 23,218 | **23,781** | 2,4% |
| **GD** put | 320 | 22,750 | 21,748 | 21,744 | **22,721** | 4,3% |
| LLY call | 1.540 | 113,700 | 113,598 | 113,617 | 113,617 | 0,0% |
| ORCL call | 165 | 28,150 | 28,182 | 28,177 | 28,183 | 0,0% |
| GD call | 440 | 10,700 | 10,745 | 10,734 | 10,734 | 0,0% |

\* (CRR americana − CRR europea) / CRR americana. La prima de mercado es el *mid*; la prima pagada en la cobertura es el *ask* (tabla de la sección 5).

En las puts de todos los strikes evaluados, el ejercicio anticipado vale entre 2,1% y 4,9% del precio: vale más cuanto más dentro del dinero y más alta es $r$, porque recibir $K$ hoy e invertirlo puede valer más que esperar. En las calls es prácticamente nulo, porque con dividendos bajos nunca conviene ejercer antes. Modelar las puts listadas como europeas subestimaría su costo; el MTM trimestral con Black-Scholes tiene un sesgo conocido y acotado por esta tabla.

## 7. Diseño de coberturas

**Dimensionamiento del 85% y sensibilidad delta.** La cobertura base compra 1 put por acción cubierta, lo que fija un piso al vencimiento, pero **no neutraliza el delta diario**:

| Ticker | Acciones | Contratos 1:1 | Cobertura efectiva | Δ put | Delta residual | Contratos delta-neutral 85% | Piso acciones cubiertas |
|---|---:|---:|---:|---:|---:|---:|---:|
| LLY | 2.799 | 24 | 85,74% | −0,318 | 72,7% | 75 | USD 2.688.000 |
| ORCL | 12.303 | 105 | 85,35% | −0,294 | 74,9% | 357 | USD 1.312.500 |
| GD | 14.849 | 126 | 84,85% | −0,315 | 73,2% | 401 | USD 4.032.000 |

Por ejemplo, las 2.400 acciones cubiertas de LLY no valdrán menos de USD 2,69 M al vencimiento, pero hoy cada dólar que cae la acción solo se compensa con ~32 centavos de la put. Neutralizar el delta exigiría 75 contratos en vez de 24; el objetivo es un piso al vencimiento, no eliminar el riesgo diario.

**Estrategias**

- **A – Protective Put (obligatoria):** acción + put comprada sobre el 85% de cada activo. Piso de protección y conserva el potencial alcista.
- **B – Collar (obligatoria):** la misma put + venta de una call OTM sobre la misma cantidad. Regla de la call: el strike líquido más bajo con $K_c/S_0 \ge 1{,}20$.
- **C – Bear Put Spread sobre LLY (overlay):** sobre la estrategia A, solo en LLY (activo que domina el riesgo y el más caro de proteger), se vende una put de strike 890 (~75% del spot): **+put 1.120 / −put 890**. Asegura las caídas normales y renuncia a cubrir caídas superiores a ~25%, tramo que la diversificación de la cartera amortigua mejor.

Straddles, strangles, butterflies o box spreads no se usan como cobertura principal: apuestan a volatilidad o a rango, o son arbitraje de tasas, no protección direccional de una cartera larga.

**Detalle del collar:**

| Ticker | K put | K call | Prima put | Prima call | Neto collar | Financiación | Piso | Techo |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| LLY | 1.120 | 1.540 | 357.240 | 259.200 | 98.040 | 73% | 82,1% del spot | +30,1% |
| ORCL | 125 | 165 | 253.050 | 276.675 | −23.625 | 109% | 73,6% del spot | +20,4% |
| GD | 320 | 440 | 315.000 | 112.140 | 202.860 | 36% | 87,6% del spot | +30,7% |

La call de GD (440) no pasa el filtro de liquidez (*spread* 33,6%); se tomó por la regla de respaldo.

**Costo por activo y por estrategia (USD):**

| Activo | A · Protective Put | B · Collar | C · Bear Put Spread |
|---|---:|---:|---:|
| LLY | 357.240 (3,57%) | 98.040 (0,98%) | 232.320 (2,32%) |
| ORCL | 253.050 (2,53%) | −23.625 (−0,24%) | 253.050 (2,53%) |
| GD | 315.000 (3,15%) | 202.860 (2,03%) | 315.000 (3,15%) |
| **Total** | **925.290 (9,25%)** | **277.275 (2,77%)** | **800.370 (8,00%)** |

En la estrategia A, **LLY recibe el mayor presupuesto de protección** (39% de la prima total).

![Payoff de las estrategias](Laboratorio%201%20-%20docs/fig9_payoffs_estrategias.png)

**Asignación del presupuesto de cobertura.** Con el mismo presupuesto de la estrategia A (≈ USD 925 mil) se compararon tres criterios:

| Criterio | Cob. LLY | Cob. ORCL | Cob. GD | Gasto | VaR99 | ES99 | ΔVaR99 por $ |
|---|---:|---:|---:|---:|---:|---:|---:|
| Base 85% uniforme | 86% | 85% | 85% | 925.290 | 1.606.327 | 1.741.654 | 0,88 |
| Valor invertido | 86% | 63% | 100% | 912.810 | 1.593.185 | 1.744.242 | 0,91 |
| Contribución al riesgo | 89% | 64% | 96% | 920.015 | 1.600.431 | 1.735.030 | 0,89 |
| **Eficiencia (VaR99/$)** | **61%** | **100%** | **100%** | **919.475** | **1.559.010** | **1.817.216** | **0,94** |

Se elige **eficiencia**: con el mismo presupuesto deja el menor VaR99 (USD 1,56 M frente a USD 1,61 M de la base). A cambio, su ES99 es el más alto de los cuatro repartos.

## 8. Seguimiento trimestral y comparación cubierto/no cubierto

En cada cierre trimestral las opciones se revaloran con Black-Scholes a la IV del contrato (*sticky strike*), la tasa del plazo remanente y $q$. Reglas:

- **Renovar** si quedan ≤ 3 meses al vencimiento o si ya venció.
- **Ajustar (roll-up)** si $S/K \ge 1{,}25$: la put quedó tan lejos que ya no protege lo ganado.
- **Monetizar** si $S/K \le 0{,}90$: la put está en el dinero y se puede realizar su ganancia.
- **Mantener** en otro caso.

**Porcentaje de escenarios en que se activa cada decisión (estrategia A):**

| Cierre | Activo | Mantener | Ajustar | Monetizar | Renovar | MTM mediano put (USD) |
|---|---|---:|---:|---:|---:|---:|
| T1 | LLY | 69% | 27% | 4% | 0% | 220.445 |
| T1 | ORCL | 54% | 30% | 16% | 0% | 210.108 |
| T1 | GD | 88% | 8% | 4% | 0% | 211.380 |
| T2 | LLY | 43% | 50% | 7% | 0% | 128.781 |
| T2 | ORCL | 39% | 39% | 22% | 0% | 170.661 |
| T2 | GD | 71% | 22% | 8% | 0% | 145.104 |
| T3 | LLY | 29% | 64% | 7% | 0% | 51.456 |
| T3 | ORCL | 32% | 43% | 25% | 0% | 127.797 |
| T3 | GD | 59% | 31% | 10% | 0% | 84.953 |
| T4 | LLY | 22% | 72% | 6% | 0% | 7.610 |
| T4 | ORCL | 27% | 45% | 28% | 0% | 76.857 |
| T4 | GD | 51% | 38% | 11% | 0% | 27.956 |
| T5–T8 | Todos | 0% | 0% | 0% | 100% | ≈ 0 |

En el primer trimestre LLY mantiene la cobertura en 69% de los escenarios; en T4, en 72% de los escenarios LLY subió tanto que conviene subir el strike. En T5 (diciembre de 2027) toca renovar en el 100% de los casos porque las opciones vencen en enero de 2028. **La simulación identifica la renovación pero no la ejecuta**, por lo que de T6 a T8 el portafolio queda sin la cobertura original.

**Valor del portafolio cubierto vs. no cubierto (neto de prima, USD):**

| Cierre | Mediana sin cob. | Mediana A | Mediana B | Mediana C | P5 sin cob. | P5 A | P5 B | P5 C |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| t=0 | 9.999.201 | 9.916.298 | 9.860.379 | 9.908.135 | 9.999.201 | 9.916.298 | 9.860.379 | 9.908.135 |
| T1 | 10.560.580 | 10.300.337 | 10.176.012 | 10.344.399 | 9.015.866 | 9.238.353 | 9.498.563 | 9.205.189 |
| T2 | 11.162.078 | 10.747.853 | 10.530.111 | 10.830.782 | 8.796.188 | 9.111.031 | 9.485.680 | 9.075.170 |
| T3 | 11.793.212 | 11.245.386 | 10.903.691 | 11.354.938 | 8.881.745 | 9.127.202 | 9.570.635 | 9.123.243 |
| T4 | 12.432.366 | 11.790.404 | 11.297.933 | 11.899.648 | 8.906.388 | 9.095.635 | 9.617.858 | 9.100.431 |
| T5 | 13.147.617 | 12.432.808 | 11.693.230 | 12.555.092 | 8.988.215 | 9.068.569 | 9.678.965 | 9.101.134 |
| T6 | 13.903.453 | 13.173.332 | 12.267.324 | 13.292.121 | 9.108.487 | 9.150.972 | 9.572.864 | 9.153.751 |
| T7 | 14.645.112 | 13.967.718 | 13.014.946 | 14.088.899 | 9.264.977 | 9.231.298 | 9.536.553 | 9.284.976 |
| T8 | 15.496.476 | 14.818.370 | 13.829.731 | 14.940.240 | 9.472.086 | 9.418.778 | 9.555.059 | 9.457.586 |

![Cubierto vs no cubierto por trimestre](Laboratorio%201%20-%20docs/fig10_cubierto_vs_no_trimestral.png)

**VaR del portafolio cubierto con contratos reales (USD):**

| Horizonte | Estrategia | Prima neta | VaR95 | VaR99 | ES99 | Δ VaR95 | Δ VaR99 | Δ ES99 | ΔVaR99 / $ prima | Costo económico | ΔVaR99 / $ costo econ. |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| T1 | Sin cobertura | – | 983.335 | 1.694.377 | 2.119.561 | – | – | – | – | – | – |
| T1 | A · Protective Put | 925.290 | 760.848 | 1.146.657 | 1.310.972 | +22,6% | +32,3% | +38,1% | 0,59 | 221.591 | 2,47 |
| T1 | B · Collar | 277.275 | 500.638 | 765.536 | 897.399 | +49,1% | +54,8% | +57,7% | 3,35 | 423.483 | 2,19 |
| T1 | C · Bear Put Spread | 800.370 | 794.012 | 1.268.442 | 1.599.219 | +19,3% | +25,1% | +24,5% | 0,53 | 186.975 | 2,28 |
| Venc. (1,32 a) | Sin cobertura | – | 987.966 | 2.420.902 | 3.206.988 | – | – | – | – | – | – |
| Venc. (1,32 a) | A · Protective Put | 925.290 | 917.440 | 1.606.327 | 1.741.654 | +7,1% | +33,6% | +45,7% | 0,88 | 608.786 | 1,34 |
| Venc. (1,32 a) | B · Collar | 277.275 | 299.447 | 958.312 | 1.093.639 | +69,7% | +60,4% | +65,9% | 5,27 | 1.956.916 | 0,75 |
| Venc. (1,32 a) | **C · Bear Put Spread** | 800.370 | 852.599 | 1.664.907 | 2.218.179 | +13,7% | +31,2% | +30,8% | 0,94 | 503.691 | **1,50** |
| T8 (2 años) | Sin cobertura | – | 527.115 | 2.332.366 | 3.128.591 | – | – | – | – | – | – |
| T8 (2 años) | A · Protective Put | 925.290 | 580.423 | 1.976.016 | 2.616.828 | −10,1% | +15,3% | +16,4% | 0,39 | 608.786 | 0,59 |
| T8 (2 años) | B · Collar | 277.275 | 444.142 | 1.994.793 | 2.962.158 | +15,7% | +14,5% | +5,3% | 1,22 | 1.956.916 | 0,17 |
| T8 (2 años) | C · Bear Put Spread | 800.370 | 541.615 | 2.053.423 | 2.702.806 | −2,8% | +12,0% | +13,6% | 0,35 | 503.691 | 0,55 |

![Distribución del P&L al vencimiento de la cobertura](Laboratorio%201%20-%20docs/fig11_pnl_estrategias.png)

**Lectura.** El costo económico es la caída del P&L esperado frente a no cubrirse (prima + *spread* + alza cedida por las calls vendidas). Por dólar de prima, el collar parece la mejor opción (5,27) y es el que más reduce el VaR99 (−60,4%) y el ES99; pero al incluir el alza cedida es el más caro (USD 1,96 M de costo económico). Por dólar de costo económico gana el **bear put spread (1,50)**, seguido de la protective put (1,34). A 2 años (T8) las reducciones caen y la protective put incluso empeora el VaR95 (−10,1%), porque las opciones vencen a los 16 meses y el tramo restante queda descubierto: la renovación no es opcional.

## 9. Conclusiones, limitaciones y recomendación final

El ejercicio permitió diseñar una cobertura objetivo del 85% (efectiva de 84,85%–85,74%) para un portafolio compuesto por LLY, ORCL y GD, incorporando criterios de protección, costo, liquidez y eficiencia frente al VaR99. Las estrategias presentan distintos intercambios entre costo y protección: la *protective put* conserva el potencial alcista con un mayor costo, el *collar* reduce el desembolso pero limita la valorización, y el *bear put spread* reduce el costo de protección a cambio de limitarla ante caídas extremas.

**Limitaciones**

- Los resultados dependen de la deriva: con la deriva histórica (LLY +34% anual) el crecimiento esperado "tapa" parte de la cola; con deriva igual a la tasa libre de riesgo el VaR99 a 2 años sube de USD 2,33 M a USD 4,63 M.
- La cobertura vence a los 1,32 años; la simulación identifica la renovación en T5 pero no la ejecuta, por lo que T6–T8 quedan sin la cobertura original.
- En ORCL ningún strike cumplió la pérdida máxima de 20% y se relajó el filtro; la call de GD del collar no pasó el filtro de liquidez.
- El MTM usa Black-Scholes con IV constante por strike (*sticky strike*), que subestima levemente el valor americano de las puts (2%–5%).
- En ORCL el GARCH-*t* resultó no estacionario y se usó la especificación normal, que captura peor las colas.
- La cadena de opciones es una foto del 25–27/09/2026; algunos strikes no tenían cotización viva.

### Recomendación final

La cartera de mínima varianza obtiene un retorno esperado de 19,33% con una volatilidad de 19,15%, utilizando la diversificación entre LLY, ORCL y GD. Sobre esta configuración se recomienda la **estrategia C (protective put en ORCL y GD + bear put spread en LLY)**, que presenta la mayor eficiencia de reducción del VaR99 por unidad de costo económico (1,50), con un costo de primas de USD 800.370 (8,00%) pagado con caja propia. Se recomienda además **renovar la cobertura en el punto de control de diciembre de 2027 (T5)** para proteger los meses restantes del horizonte. La propuesta no busca maximizar el retorno esperado, sino combinar una cartera diversificada con un mecanismo de protección que controle las pérdidas poco frecuentes pero de gran magnitud, manteniendo exposición al mercado.

La cartera es viable bajo los supuestos establecidos: construimos el portafolio, cuantificamos su relación riesgo-retorno, diseñamos una cobertura cercana al 85%, valoramos las opciones, simulamos escenarios y medimos la reducción del riesgo. No afirmamos que el portafolio vaya a generar necesariamente ese retorno en los próximos dos años: el 19,33% es un retorno esperado obtenido bajo el modelo, no un rendimiento garantizado.

## 10. Anexo de reproducibilidad

- **Notebook:** `Laboratorio1_Derivados.ipynb` (ejecución completa de los tres bloques).
- **Semilla:** `SEED = 42` (10.000 trayectorias para 2 años; 4.000 para la vista a 10 años con `SEED + 1`).
- **Datos exportados** en `Laboratorio 1 - docs/`: `cadenas_opciones.csv` (cadena del 21/01/2028), `puts_candidatas.csv`, `curva_treasury.csv`, `metadatos_activos.csv` y las figuras `fig1`–`fig11`.
- **Reproducir con la misma cadena de opciones:** `MODO_CADENA = "csv"`; con `"vivo"` se descarga la cadena del día (conviene correrla con el mercado abierto).
- **Cambiar de portafolio:** `CRITERIO = "max_sharpe"` re-ejecuta todo con la cartera de máximo Sharpe.
- **Parámetros clave:** capital USD 10 M, horizonte 2 años, cobertura 85%, contrato 100 acciones, 252 días/año, pesos 15%–50%, *spread* ≤ 20%, OI ≥ 10, pérdida máxima ≤ 20%, call del collar con $K_c/S_0 \ge 1{,}20$, put vendida del spread ≈ 75% del spot.

| Librería | Versión |
|---|---|
| Python | 3.12.14 |
| numpy | 2.5.2 |
| pandas | 3.0.5 |
| scipy | 1.18.1 |
| arch | 8.0.0 |
| yfinance | 1.7.0 |
| matplotlib | 3.11.2 |

```bash
pip install numpy pandas scipy arch yfinance matplotlib
jupyter notebook Laboratorio1_Derivados.ipynb
```

---

## Referencias

- FRED – Federal Reserve Bank of St. Louis (2026). *Market Yield on U.S. Treasury Securities at Constant Maturity* (DGS3MO, DGS6MO, DGS1, DGS2).
- Yahoo Finance (2026). Precios históricos, dividendos y cadenas de opciones de LLY, ORCL y GD, vía `yfinance`.
- Bollerslev, T. (1990). Modelling the coherence in short-run nominal exchange rates: a multivariate generalized ARCH model. *The Review of Economics and Statistics*, 72(3), 498–505.
- Cox, J., Ross, S. y Rubinstein, M. (1979). Option pricing: a simplified approach. *Journal of Financial Economics*, 7(3), 229–263.
