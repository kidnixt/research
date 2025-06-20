# t-Distributed Stochastic Neighbor Embedding — Explicación Matemática Detallada

---

## 1. Objetivo de t-SNE

Dado un conjunto de puntos $\{x_1, x_2, \dots, x_N\}$ en un espacio de alta dimensión $\mathbb{R}^D$, queremos encontrar una representación en baja dimensión $\{y_1, y_2, \dots, y_N\}$, con $y_i \in \mathbb{R}^d$ (usualmente $d=2$), que preserve las relaciones de similitud local entre los puntos.

---

## 2. Construcción de probabilidades en el espacio original (alta dimensión)

Para cada par de puntos $x_i, x_j$, definimos la similitud como una probabilidad condicional que mide cuán probable es que $x_i$ "escoja" a $x_j$ como vecino, basada en una distribución Gaussiana centrada en $x_i$:

$$
p_{j|i} = \frac{\exp\left(-\frac{\|x_i - x_j\|^2}{2 \sigma_i^2}\right)}{\sum_{k \neq i} \exp\left(-\frac{\|x_i - x_k\|^2}{2 \sigma_i^2}\right)}
$$

- $\sigma_i$ es la desviación estándar del kernel Gaussiano para el punto $x_i$.  
- La probabilidad $p_{i|i} = 0$ por definición.

Para hacer la distribución simétrica y que las distancias mutuas tengan la misma importancia:

$$
p_{ij} = \frac{p_{j|i} + p_{i|j}}{2N}
$$

Con esto, $\sum_{i \neq j} p_{ij} = 1$.

---

## 3. Construcción de probabilidades en el espacio embebido (baja dimensión)

Ahora definimos probabilidades $q_{ij}$ que miden la similitud entre los puntos embebidos $y_i, y_j \in \mathbb{R}^d$.

Pero, en lugar de usar una Gaussiana (como en el espacio original), se usa una distribución **t de Student con 1 grado de libertad (Cauchy)** para combatir el problema del "crowding":

$$
q_{ij} = \frac{\left(1 + \|y_i - y_j\|^2\right)^{-1}}{\sum_{k \neq l} \left(1 + \|y_k - y_l\|^2\right)^{-1}}
$$

De nuevo, $q_{ii} = 0$.

---

## 4. Objetivo de optimización

Queremos que la distribución $Q = \{q_{ij}\}$ en baja dimensión se parezca lo máximo posible a la distribución $P = \{p_{ij}\}$ del espacio original.

Para eso minimizamos la divergencia de Kullback-Leibler (KL) de $P$ respecto a $Q$:

$$
C = KL(P \| Q) = \sum_{i \neq j} p_{ij} \log \frac{p_{ij}}{q_{ij}}
$$

Minimizar $C$ implica que las probabilidades de vecindad en el espacio embebido se ajusten para parecerse a las del espacio original.

---

## 5. Cálculo de gradientes para optimización

Para minimizar $C$, se usa gradiente descendente. El gradiente respecto a $y_i$ es:

$$
\frac{\partial C}{\partial y_i} = 4 \sum_{j} (p_{ij} - q_{ij})(y_i - y_j) \left(1 + \|y_i - y_j\|^2\right)^{-1}
$$

La constante 4 viene de derivaciones del kernel t-distribuido y facilita la convergencia.

---

## 6. Ajuste de $\sigma_i$ y parámetro *perplexity*

El ancho $\sigma_i$ del kernel Gaussiano se ajusta para que la distribución local tenga una **perplejidad** deseada:

$$
\text{Perplexity}(P_i) = 2^{H(P_i)}
$$

donde $H(P_i)$ es la entropía de Shannon de la distribución $P_i = \{p_{j|i}\}_j$:

$$
H(P_i) = - \sum_j p_{j|i} \log_2 p_{j|i}
$$

En la práctica, para cada punto $i$, se busca $\sigma_i$ tal que la perplexity sea cercana a un valor fijado (por ejemplo 30), usando búsqueda binaria.

---

## 7. Resumen del algoritmo t-SNE

1. Para cada punto $x_i$, calcular las probabilidades condicionales $p_{j|i}$ con Gaussiana y ajustar $\sigma_i$ para lograr la perplexity deseada.  
2. Construir la matriz simétrica $p_{ij}$.  
3. Inicializar aleatoriamente los puntos $y_i$ en $\mathbb{R}^d$.  
4. Calcular $q_{ij}$ usando la distribución t de Student.  
5. Minimizar $KL(P \| Q)$ con gradiente descendente para encontrar la mejor ubicación $y_i$.  
6. Retornar la representación en baja dimensión.

---

## 8. Comentarios finales

- La elección de la distribución t en baja dimensión evita que puntos distantes se aglomeren (problema de crowding).  
- La optimización es no convexa, por eso los resultados pueden variar.  
- El método se enfoca en preservar **similitudes locales** más que globales.

---
