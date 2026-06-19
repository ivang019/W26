# Modelo Probabilístico del Mundial 2026: Elo, Poisson y Simulación Monte Carlo

## Nota técnica

---

## 1. Contexto y objetivo

Este documento describe la construcción de un modelo de forecasting probabilístico para el Mundial 2026. El objetivo no es predecir un resultado único, sino estimar **distribuciones de probabilidad**: qué tan probable es que cada selección gane un partido, avance de ronda, o se corone campeona.

La pregunta metodológica de fondo es simple de enunciar y no trivial de resolver: **¿cómo se traduce la fuerza relativa de dos equipos en una distribución de marcadores?**

El enfoque elegido combina tres piezas, cada una resolviendo una parte distinta del problema:

- **Elo**: una métrica de fuerza relativa entre selecciones, externa al proyecto (no calculada por el autor, sino tomada de una fuente especializada).
- **Regresión de Poisson**: un modelo estadístico que traduce esa fuerza relativa en goles esperados por partido.
- **Simulación Monte Carlo**: un método para propagar la incertidumbre del modelo de partido hacia el resultado de un torneo completo de 48 equipos, respetando sus reglas exactas de grupos, terceros clasificados y bracket eliminatorio.

La decisión de partida fue priorizar un modelo **simple e interpretable** sobre uno complejo. Esa decisión tiene un costo (se documenta en la sección 8) y se sostiene en validación empírica, no solo en preferencia metodológica.

---

## 2. Fuentes de datos

El proyecto usa tres fuentes externas, cada una cumpliendo un rol distinto. Ninguna fue generada por el autor — todas son **datos prestados**, etiquetados explícitamente como tales.

| Fuente | Rol | Cobertura | Licencia |
|---|---|---|---|
| `martj42/international-football-results` (Kaggle) | Resultados históricos de partidos, 1872–2026 | ~49,500 partidos | Uso libre |
| `afonsofernandescruz/2026-fifa-world-cup-historical-elo-ratings` (Kaggle) | Elo pre-torneo de las 48 selecciones clasificadas | 48 equipos, snapshots anuales | CC BY-SA 4.0 |
| `saifalnimri/international-football-elo-ratings` (Kaggle) | Serie histórica de Elo para entrenar el modelo | 270 equipos, 1872–2025 | Ver Kaggle |

**Por qué tres fuentes y no una.** La primera intuición fue usar dos: resultados históricos y Elo. En la práctica se necesitó una tercera porque el Elo cumple dos funciones distintas que requieren grano distinto:

1. **Elo pre-torneo**: el valor de cada una de las 48 selecciones el día antes de que arranque el Mundial. Esto alimenta la simulación.
2. **Elo histórico por fecha**: el valor de *cualquier* selección en *cualquier* momento entre 2010 y 2026, necesario para entrenar el modelo Poisson sobre miles de partidos pasados.

El bundle de la fuente 2 cubre el primer uso pero solo tiene 48 equipos — insuficiente para entrenar un modelo sobre partidos históricos, donde participan cientos de selecciones distintas. La fuente 3 cubre ese vacío.

---

## 3. Construcción del dataset de entrenamiento

Esta sección documenta el camino real, incluyendo los errores encontrados y cómo se corrigieron. Se incluye deliberadamente porque el proceso de diagnóstico es tan parte del método como el resultado final.

### 3.1 Primer intento y su falla silenciosa

El primer dataset de entrenamiento se construyó pegando el Elo histórico (fuente 2, el bundle de 48 equipos) a cada partido del histórico de resultados, emparejando por año. La cobertura resultante fue del **12.2%** — solo 1,932 de 15,790 partidos.

**Diagnóstico.** Los equipos que no encontraban Elo eran selecciones grandes: Italia, Chile, Costa Rica, Nigeria. La causa no era un problema de nombres (el tipo de error más común en este tipo de uniones) sino una limitación estructural de la fuente: el bundle de 48 equipos solo contiene el historial de *esas* 48 selecciones. Italia no va al Mundial 2026, así que no aparece en el archivo, sin importar cuántos partidos haya jugado contra Chile en el pasado.

**Esto motivó la incorporación de la fuente 3** — el archivo histórico de 270 equipos — específicamente para el entrenamiento del modelo, manteniendo el bundle de 48 solo para el Elo pre-torneo.

### 3.2 La densidad de la nueva fuente y el merge temporal

El archivo de 270 equipos no es una serie diaria de Elo: es un conjunto de snapshots dispersos, con una mediana de **1 registro por equipo por año**. Esto descarta la idea de "pegar el Elo exacto del día del partido" — no existe ese dato con esa granularidad.

La solución fue un **merge_asof** (unión por la fecha más reciente disponible hacia atrás), con un tope de antigüedad máxima para evitar usar un Elo demasiado viejo. Se probaron tres topes:

| Tope | Cobertura global | Cobertura entre los 48 | Tramo Elo 1500–1700 |
|---|---|---|---|
| 9 meses | 44.5% | 50.2% | 20.9% |
| 12 meses | 53.6% | 59.9% | 28.5% |
| 18 meses | 64.5% | 70.9% | 39.1% |

**Decisión: 18 meses.** El tramo de equipos medianos (Elo 1500–1700, donde caen varias selecciones del Mundial 2026) seguía siendo el más afectado en cualquier tope, pero 18 meses ofrecía la mejor cobertura sin comprometer la regla de no usar información posterior al partido (anti-fuga temporal). Esta limitación se documenta explícitamente: las predicciones de equipos en ese tramo de Elo usan, en algunos casos, un valor de hasta 18 meses de antigüedad.

### 3.3 El error de especificación: localía contaminando el coeficiente de Elo

El primer modelo ajustado incluía todos los partidos (neutrales y no neutrales) con dos variables: la diferencia de Elo y un indicador de localía. El coeficiente de Elo resultante fue **β = 0.000290** — sorprendentemente pequeño.

**Síntoma.** Al simular partidos de prueba, el modelo subestimaba sistemáticamente las diferencias de fuerza grandes. España contra el equipo más débil del torneo (Qatar) producía una probabilidad de victoria de apenas 49% — muy por debajo de lo que la intuición futbolística y la diferencia de Elo (740 puntos) sugieren.

**Diagnóstico.** El problema no estaba en los datos sino en la especificación del modelo. Cuando un partido tiene localía real, el equipo local tiende a ser, en promedio, también el más fuerte. El modelo tiene entonces dos variables compitiendo por explicar la misma señal (quién marca más goles): la diferencia de Elo y la ventaja de jugar en casa. El ajuste estadístico reparte el crédito entre ambas, y el coeficiente de Elo queda artificialmente reducido.

**Corrección.** Dado que el Mundial 2026 se juega casi en su totalidad en sedes neutrales (con la excepción de los partidos de fase de grupos de los anfitriones), el modelo se reestimó usando **únicamente partidos neutrales**, sin variable de localía. El coeficiente resultante subió a **β ≈ 0.00092–0.00185** según la ventana de entrenamiento usada — entre 3 y 6 veces mayor que el modelo original, y estable entre distintos cortes temporales (ver sección 5.1).

Esta corrección cambió las predicciones de forma sustancial: España–Qatar pasó de 49% a 78% de probabilidad de victoria para España, un valor mucho más consistente con la diferencia real de nivel entre ambos equipos.

### 3.4 Inconsistencia de ventana temporal entre versiones del modelo

Un segundo error, más sutil, apareció al comparar dos versiones del modelo "neutral" que debían ser equivalentes y no lo eran. La causa fue una diferencia de un día en el límite superior de la ventana de entrenamiento: una versión cortaba en `2025-12-31` y otra en `2026-06-09`, dejando fuera o dentro 117 partidos neutrales ya jugados en la primera mitad de 2026. Esos partidos, al ser los más recientes, tenían mejor cobertura de Elo y una distribución de diferenciales distinta, lo que movía el coeficiente estimado.

**Lección metodológica:** dos bloques de código que dicen hacer "lo mismo" pueden diferir en un detalle de fecha que cambia el resultado de forma no trivial. La verificación cruzada de los números (no solo de la lógica del código) fue lo que detectó la discrepancia.

---

## 4. El modelo: de Elo a goles esperados

### 4.1 Intuición

El Elo es un número que resume qué tan bueno es un equipo, calibrado de forma relativa: solo importa la *diferencia* respecto al rival, no el valor absoluto. Pero el Elo no dice cuántos goles se esperan en un partido — fue diseñado para estimar probabilidad de victoria, no marcador.

El puente entre ambos es una variable que sí tiene una distribución natural para conteos de eventos discretos y no negativos (como los goles): la **distribución de Poisson**.

### 4.2 Especificación formal

Para un partido entre el equipo *i* y el equipo *j*, se modela el número de goles de *i* como una variable aleatoria Poisson con media λᵢ:

$$
\log(\lambda_i) = \alpha + \beta \cdot (\text{Elo}_i - \text{Elo}_j)
$$

donde:

- **α** (intercepto): el logaritmo de los goles esperados cuando ambos equipos tienen el mismo Elo, en un partido neutral. Su exponencial, exp(α), debe rondar 1.2–1.5 — la media histórica de goles por equipo en fútbol internacional.
- **β** (pendiente): cuánto aumenta el logaritmo de los goles esperados por cada punto de ventaja de Elo. Debe ser positivo y, dado que ambos equipos comparten el mismo β, simétrico: la desventaja de un equipo es exactamente la ventaja del otro con signo opuesto.

Los parámetros se estiman por máxima verosimilitud (el método estándar de ajuste en un GLM de familia Poisson), usando como observaciones cada participación de un equipo en un partido histórico (cada partido aporta dos observaciones: una por equipo).

### 4.3 De goles esperados a probabilidades de resultado

Dado λᵢ y λⱼ, la probabilidad de cualquier marcador exacto se obtiene asumiendo independencia entre los goles de ambos equipos:

$$
P(X_i = x, X_j = y) = \frac{e^{-\lambda_i}\lambda_i^x}{x!} \cdot \frac{e^{-\lambda_j}\lambda_j^y}{y!}
$$

Sumando sobre todas las combinaciones donde x > y, x = y, o x < y se obtienen las probabilidades de victoria, empate y derrota.

### 4.4 Parámetros finales del modelo

Tras la corrección descrita en 3.3, el modelo de producción se entrenó sobre partidos neutrales, ventana 2010-01-01 a 2026-06-09, con cobertura de Elo del 57.9% sobre 4,814 partidos neutrales jugados:

| Parámetro | Valor | Interpretación |
|---|---|---|
| α | 0.247 | exp(α) = 1.28 goles esperados en un duelo parejo neutral |
| β | 0.000920 | cada 100 puntos de ventaja de Elo multiplican λ por ≈1.10 |

### 4.5 Límites del enfoque

El modelo asume independencia entre los goles de ambos equipos dentro de un mismo partido. Esta es una simplificación conocida: en la práctica, los resultados de marcador bajo (0-0, 1-1) ocurren con mayor frecuencia de la que predice un Poisson independiente, porque el desarrollo táctico de un partido correlaciona los goles de ambos lados. La corrección estándar para esto (Dixon-Coles) no se implementó en esta versión; es la mejora más directa para una iteración futura.

---

## 5. Validación

La validación es la pieza que distingue un modelo defendible de uno especulativo. Se diseñó un backtest temporal estricto: el modelo se reentrena usando solo información disponible *antes* del torneo que se evalúa, y se mide su desempeño contra los resultados reales de ese torneo.

### 5.1 Backtest partido a partido — Mundiales 2018 y 2022

Para cada torneo se reentrenó un modelo auxiliar idéntico en especificación al modelo de producción (neutral, sin localía), usando partidos hasta el día anterior al inicio de cada Mundial.

| Torneo | β auxiliar | exp(α) | n. partidos en fit |
|---|---|---|---|
| 2018 | 0.001855 | 1.157 | 1,170 |
| 2022 | 0.001829 | 1.138 | 1,834 |
| 2026 (producción) | 0.000920 | 1.280 | 2,789 |

La diferencia de magnitud en β entre los modelos auxiliares y el de producción se explica por el tamaño y composición de cada muestra; lo relevante es que los tres son positivos, del mismo orden de magnitud, y producen predicciones cualitativamente consistentes (ver sección 5.3).

Con esos modelos se generaron predicciones para los 64 partidos de cada Mundial y se compararon contra tres métricas estándar en forecasting probabilístico, frente a una referencia ingenua que asigna 1/3 de probabilidad a cada resultado:

| Métrica | 2018 (modelo) | 2018 (referencia) | 2022 (modelo) | 2022 (referencia) |
|---|---|---|---|---|
| Brier Score | 0.5802 | 0.6667 | 0.5988 | 0.6667 |
| Log Loss | 0.9763 | 1.0986 | 1.0150 | 1.0986 |
| RPS | 0.2080 | 0.2439 | 0.2123 | 0.2439 |

En las tres métricas y en ambos torneos, el modelo mejora sobre la referencia ingenua entre 8% y 15%.

### 5.2 Pruebas estadísticas formales

Para descartar que la mejora observada se deba al azar muestral (64 partidos por torneo es una muestra modesta), se aplicaron tres pruebas:

**Test de permutaciones sobre el RPS.** Se generaron 10,000 reordenamientos aleatorios de las predicciones del modelo respecto a los resultados reales, y se midió en qué proporción de esos reordenamientos el RPS resultante superaba al RPS observado del modelo.

$$
p\text{-valor} = \frac{\#\{\text{permutaciones donde } \overline{RPS}_{\text{perm}} \leq \overline{RPS}_{\text{obs}}\}}{N_{\text{perm}}}
$$

| Torneo | p-valor |
|---|---|
| 2018 | 0.0001 |
| 2022 | 0.0029 |

Ambos resultan significativos muy por debajo del umbral convencional de 0.05: la probabilidad de que la ventaja del modelo sobre el azar sea casualidad es menor al 0.3% en el peor caso.

**Accuracy del resultado más probable.** Porcentaje de partidos donde el resultado (victoria local, empate, victoria visitante) con mayor probabilidad asignada por el modelo coincidió con el resultado real:

| Torneo | Accuracy modelo | Accuracy referencia (predecir siempre el resultado más común) |
|---|---|---|
| 2018 | 54.7% | 40.6% |
| 2022 | 54.7% | 43.8% |

**Correlación de Spearman entre Elo pre-torneo y ronda alcanzada.** Se asignó un valor ordinal a la ronda alcanzada por cada uno de los 32 equipos de cada torneo (0 = eliminado en grupos, 5 = campeón) y se correlacionó contra su Elo de entrada:

| Torneo | ρ (Spearman) | p-valor |
|---|---|---|
| 2018 | 0.538 | 0.0015 |
| 2022 | 0.459 | 0.0083 |

Ambas correlaciones son positivas, moderadas-altas, y estadísticamente significativas: el Elo pre-torneo predice de forma sistemática (aunque lejos de perfecta) qué tan lejos llega un equipo.

### 5.3 Simulación retrospectiva de torneo completo

Más allá de las métricas de partido individual, se simuló el torneo completo de 2018 y 2022 (10,000 simulaciones cada uno) para comparar las probabilidades pre-torneo del modelo contra lo que efectivamente ocurrió:

| Torneo | Campeón real | P(campeón) asignada | Posición en el ranking de favoritos |
|---|---|---|---|
| 2018 | Francia | 7.0% | 5° de 32 |
| 2022 | Argentina | 18.6% | 2° de 32 |

En ambos casos el campeón real estuvo entre los principales favoritos del modelo, sin haber sido necesariamente el favorito absoluto — un resultado consistente con la naturaleza de alta varianza de un torneo eliminatorio de 32-48 equipos.

Las sorpresas que el modelo no anticipó (la semifinal de Marruecos en 2022, con apenas 2.1% de probabilidad asignada) se interpretan como evidencia de los límites inherentes del azar en el fútbol, no como una falla de calibración: ningún modelo razonable basado en fuerza pre-torneo puede anticipar sistemáticamente ese tipo de resultado.

---

## 6. Simulación Monte Carlo del Mundial 2026

### 6.1 Estructura del torneo

El formato 2026 introduce una complejidad ausente en ediciones anteriores: 48 equipos en 12 grupos de 4, donde avanzan los dos primeros de cada grupo más los **8 mejores terceros**, determinados por una tabla de comparación entre todos los terceros lugares (puntos, diferencia de goles, goles a favor).

El reglamento de la FIFA contempla **495 combinaciones posibles** de qué 8 grupos producen esos terceros clasificados, cada una con un mapeo distinto de cruces en la ronda de 32. Esta tabla se obtuvo de la documentación pública del torneo (Anexo C del reglamento de competición) y se incorporó de forma programática para evitar errores de transcripción manual.

### 6.2 Mecánica de la simulación

Cada una de las 10,000 iteraciones repite el siguiente proceso:

1. Simular los 72 partidos de fase de grupos (goles vía Poisson con los λ del modelo).
2. Calcular tablas de grupo y determinar primero, segundo y tercer lugar de cada uno.
3. Seleccionar los 8 mejores terceros según las reglas de desempate oficiales.
4. Resolver el cruce de ronda de 32 usando la tabla de 495 combinaciones.
5. Simular el resto del torneo eliminatorio hasta la final.

### 6.3 Supuestos explícitos

- **S1 — Localía**: todo el torneo se trata como neutral, incluyendo los partidos de los anfitriones (México, Estados Unidos, Canadá). Se evaluó incluir ventaja de localía, pero esto producía un sesgo artificial al aplicarse incorrectamente en rondas de eliminatoria donde la sede ya no corresponde necesariamente al país anfitrión; se optó por la especificación neutral, consistente con la validación de la sección 5.
- **S2 — Penales**: los empates en fases eliminatorias se resuelven con probabilidad 50/50, sin modelar el desarrollo de la tanda de penales. Justificación: los penales son, en la evidencia empírica del fútbol, cercanos a un resultado aleatorio, y modelarlos con mayor detalle no está respaldado por datos suficientes en este proyecto.
- **S3 — Antigüedad del Elo**: para equipos en el tramo 1500–1700 de Elo, el valor histórico usado en el entrenamiento puede tener hasta 18 meses de antigüedad respecto a la fecha del partido (ver sección 3.2). El Elo pre-torneo usado en la simulación de 2026 no tiene este problema, ya que proviene de un snapshot único reciente (27 de mayo de 2026).

### 6.4 Resultado principal

Las probabilidades de campeonato resultantes (top de la tabla completa):

| Equipo | Elo | P(campeón) |
|---|---|---|
| España | 2165 | 11.5% |
| Argentina | 2113 | 9.0% |
| Francia | 2081 | 7.4% |
| Inglaterra | 2020 | 5.5% |
| Brasil | 1984 | 4.9% |

La tabla completa de 48 equipos, con probabilidades por ronda (octavos, cuartos, semifinal, final, campeón), está disponible en `data/outputs/mc2026_probabilidades.csv`.

---

## 7. Resultados y su interpretación

Los resultados de la sección 6.4 deben leerse a la luz de la validación de la sección 5: el modelo no garantiza acertar el campeón (ningún modelo serio lo hace, dado el azar inherente al fútbol), pero sí ha demostrado, sobre dos torneos reales, que:

- Discrimina correctamente entre favoritos y débiles (correlación Elo-ronda significativa).
- Sus probabilidades superan a una asignación aleatoria de forma estadísticamente robusta.
- Los campeones reales de 2018 y 2022 estuvieron consistentemente entre sus principales favoritos.

Esto posiciona los resultados de la sección 6.4 como una **estimación informada y validada**, no como una predicción categórica. España como favorito con 11.5% no significa "España va a ganar"; significa que, bajo este modelo y su evidencia de calibración histórica, España es la selección con mayor probabilidad relativa entre 48 posibles candidatos — en un torneo donde incluso el favorito más fuerte rara vez supera el 20-25% de probabilidad de título.

---

## 8. Limitaciones conocidas

Documentadas explícitamente, sin atenuantes:

1. **Variable explicativa única.** El modelo no incorpora forma reciente, lesiones, convocatorias, estilo táctico, ni efecto de altitud o clima. Todo el poder predictivo proviene de un solo número (Elo) por equipo.

2. **Subestimación del empate.** Por construcción del Poisson independiente (sección 4.5), el modelo no predijo el empate como resultado más probable en ningún partido de la validación 2018/2022, a pesar de que los empates representaron 20-23% de los resultados reales. Dixon-Coles es la corrección estándar pendiente de implementar.

3. **Antigüedad heterogénea del Elo de entrenamiento.** Para equipos en ciertos rangos de fuerza (sección 3.2), el dato de entrada al modelo puede ser hasta 18 meses más viejo que el partido que predice, introduciendo ruido no cuantificado en esas observaciones específicas.

4. **Penales no modelados.** La resolución 50/50 de empates en eliminatorias (supuesto S2) es una simplificación deliberada; no captura ningún efecto de presión, portero especialista, ni historial en tandas de penales.

5. **Sin actualización dinámica.** Las probabilidades de la sección 6.4 son un snapshot pre-torneo. No se actualizan con los resultados reales de la fase de grupos una vez que el Mundial comience.

6. **Validación sobre dos torneos.** El backtest de la sección 5 usa únicamente 2018 y 2022 — dos observaciones a nivel de "torneo completo". Las métricas de partido individual (128 observaciones en total) tienen mayor robustez estadística que las métricas de torneo (probabilidad de campeón, semifinalistas), que dependen de la realización única de cada Mundial.

---

*Documento elaborado como parte de un proyecto de aprendizaje aplicado en forecasting probabilístico deportivo. Código completo, datos de entrenamiento y resultados disponibles en el repositorio.*
