# Análisis y Clarificación de Equivalencias en Modelos de Probabilidad

**Fecha:** 17 de junio de 2025 **Participantes:** Sergio Yovine, Franz Mayr, Matías Carrasco Piaggio, Martín Iturbide 

---

## 1. Definiciones Clave

### A. Rango (`rank`) y Top-r (`top_r`)

- **`rank(δ): Σs → ℕ`**: Función que asigna un rango (orden) a cada símbolo `σ ∈ Σs` basado en su probabilidad `δ(σ)`.
    - **Asunción**: `rank(δ)` es inyectivo (no hay empates). Si existen, se resuelven mediante un ordenamiento arbitrario (e.g., lexicográfico).
- **`top_r(δ) = {σ ∈ Σs | rank(δ)(σ) ≤ r}`**: Conjunto de los `r` símbolos con mayor probabilidad (menor rango), olvidando su orden relativo interno.

### B. Equivalencia `ρ =top, ρ'` (Definición en Paper/Teoría)

- Dos distribuciones `ρ` y `ρ'` son "top equivalentes" si y solo si:
    1. Comparten el **mismo soporte** (`supp(ρ) = supp(ρ')`).
    2. Sus conjuntos `top_r` son iguales (`top_r(ρ) = top_r(ρ')`).

### C. Implementación `TopKProbabilityPartitionerPlus` (Código Python)

Esta clase implementa una partición (agrupación) de distribuciones de probabilidad:

- `__init__(self, k)`: Inicializa con un parámetro `k` (análogo a `r`).
- `_get_partition(self, probability_vector)`:
    1. `order = (-probability_vector).argsort()`: Ordena los índices de los símbolos de forma descendente por probabilidad.
    2. `support_mask = probability_vector > 0`: Identifica los símbolos que tienen probabilidad > 0 (el soporte).
    3. `top_k_mask = np.zeros_like(probability_vector, dtype=int)`: Crea una máscara para los `k` elementos con mayor probabilidad.
    4. `top_k_mask[order[:self.k]] = 1`: Marca con '1' los `k` elementos principales en la máscara.
    5. `partition = top_k_mask * support_mask.astype(int)`: Combina la información de `top_k` y `support`. Un símbolo estará en la partición final si es uno de los `top_k` _Y_ tiene probabilidad > 0.

## 2. Problemas y Puntos de Discusión

### 2.1. Símbolos con Probabilidad Cero en `top_r` y el Impacto del Soporte

- **Preocupación Inicial (Sergio Yovine):** La definición de `top_r` podría incluir símbolos con probabilidad 0 si `k` es mayor que la cantidad de símbolos con probabilidad > 0. Esto podría generar inconsistencias o problemas de orden.
- **Ejemplo Crucial:**
    - `p1 = [1, 0, 0]`
    - `p2 = [0.9, 0.1, 0]`
- **Análisis del Código (`TopKProbabilityPartitionerPlus`):**
    - **Para `k = 1`:**
        - `p1`: `top_k_mask = [1,0,0]`, `support_mask = [1,0,0]`. `partition = [1,0,0]`.
        - `p2`: `top_k_mask = [1,0,0]`, `support_mask = [1,1,0]`. `partition = [1,0,0]`.
        - **Conclusión:** El código las considera **equivalentes bajo `top_1`**.
    - **Para `k = 2`:**
        - `p1`: `top_k_mask = [1,0,0]`, `support_mask = [1,0,0]`. `partition = [1,0,0]`.
        - `p2`: `top_k_mask = [1,1,0]`, `support_mask = [1,1,0]`. `partition = [1,1,0]`.
        - **Conclusión:** El código las considera **NO equivalentes bajo `top_2`**.
- **Aclaración (Franz Mayr):** Se aclara que la equivalencia `top_k` se entiende como: `p1` y `p2` son `top_k` equivalentes si el **soporte de sus proyecciones sobre los `top_k` símbolos más probables** es el mismo (`supp(samptop_k(p1)) = supp(samptop_k(p2))`). Esto es consistente con la implementación del "partitioner plus".

### 2.2. Terminología: "Generable Prefix" vs "Generable Word"

- **Propuesta (Sergio Yovine):**
    - **"u is a generable prefix"**: Para `P(u) > 0`.
    - **"generable word" (o string)**: Para `P_$(u) > 0` (donde `$` indica el final de la palabra).
- **Aceptación (Franz Mayr, Matías Carrasco Piaggio):** Se considera una buena terminología, análoga a "reachable" (alcanzable).

### 2.3. Problema Principal: Precondición de Igualdad de Soporte

- **Contexto:** Para _toda_ equivalencia `E`, se había establecido la precondición `supp(pdist) = supp(pdist')`.
- **Conflicto (Sergio Yovine):** Esta precondición es lógica para la _cuantización_, pero **genera un problema fundamental para `top_r` y `rank`**, ya que impide que distribuciones con el mismo `top_r` (pero soporte diferente fuera de ese `top_r`) sean consideradas equivalentes. Esto contradice la idea de que `top_r` "naturalmente induce una equivalencia".
- **Ejemplo Clave (Revisitado):** `[1, 0, 0]` y `[0.9, 0.1, 0]`. Aunque tienen el mismo `top_1`, la precondición de soportes iguales las haría no equivalentes, lo que parece "engorroso" si el objetivo es la equivalencia por `top_k`. Sin embargo, si se aplica `samptop_1` primero (proyectando), ambas se convertirían en `[1, 0, 0]`, y entonces sí serían equivalentes.
- **Estado Actual de la Implementación (Martín Iturbide):** Los experimentos actuales utilizan "partitioners plus" (que sí consideran el soporte) y solo con "Quantization", no con "TopK".

## 3. Conclusión y Decisiones

La discusión culmina con la siguiente decisión fundamental para la coherencia del modelo:

- **Consenso Final:** Para el contexto de la congruencia que chequea `1_L` (la función indicadora), **se mantiene la precondición de que las distribuciones deben tener el mismo soporte (`supp(pdist) = supp(pdist')`) para ser consideradas equivalentes**.
- **Implicación:** Esto significa que, aunque dos distribuciones puedan tener el mismo `top_r`, si sus soportes completos difieren, no serán consideradas equivalentes bajo esta congruencia. Esto es una elección consciente para la coherencia con la congruencia que se busca, a pesar de las aparentes "contradicciones" con una interpretación más pura de `top_r` que ignora el resto del soporte.
- **Próximos Pasos:**
    - Para la presentación, se utilizará la definición de equivalencia que es consistente con el código (`TopKProbabilityPartitionerPlus`), es decir, donde la partición final es el AND entre `top_k_mask` y `support_mask`.
    - Se reconoce que el problema del soporte es un punto crítico y se evaluará si es necesario mejorarlo en el futuro para alinear aún más la teoría y la práctica.