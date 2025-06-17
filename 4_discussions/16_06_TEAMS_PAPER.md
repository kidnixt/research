# 📊 Análisis y Clarificación de Equivalencias en Modelos de Probabilidad

**Fecha:** 17 de junio de 2025  
**Participantes:** Sergio Yovine, Franz Mayr, Matías Carrasco Piaggio, Martín Iturbide

---

## 1. 📌 Definiciones Fundamentales

### A. 🧮 Rango (`rank`) y Top-r (`top_r`)

- $\text{rank}(\delta) : \Sigma_s \rightarrow \mathbb{N}$  asigna un orden a cada símbolo $\sigma\in\Sigma_s$  según su probabilidad $\delta(\sigma)$. 
    - Se asume que $\text{rank}(\delta)$ es inyectiva (no hay empates). En caso de empate, se resuelve con un orden arbitrario (e.g., lexicográfico).
- $\text{top}_r(\delta) = \{ \sigma \in \Sigma_s \mid \text{rank}(\delta)(\sigma) \leq r \}$: conjunto de los $r$ símbolos más probables (olvida su orden relativo interno).

---

### B. 🔁 Equivalencia Teórica $\rho =_{\text{top}_r} \rho'$ 

Según el paper:

> Dos distribuciones $\rho$ y $\rho'$ son equivalentes bajo `top_r` si:
> 1. Tienen el **mismo soporte**: `supp(ρ) = supp(ρ')`
> 2. Sus conjuntos `top_r` son iguales: `top_r(ρ) = top_r(ρ')`



$\rho =_{\text{top}_r} \rho' \iff$ $\text{supp}(\rho) = \text{supp}(\rho') \quad \text{y} \quad \text{top}_r(\rho) = \text{top}_r(\rho')$

⚠️ **Nota:** Esta definición ignora el orden interno dentro del top-r y sólo compara la pertenencia.

---

### C. ⚙️ Implementación: `TopKProbabilityPartitionerPlus`

El siguiente código implementa una partición basada en `top_k` y el soporte:

python

CopyEdit

`class TopKProbabilityPartitionerPlus(ProbabilityPartitioner):     def __init__(self, k) -> None:         self.k = k         super().__init__()      def _get_partition(self, probability_vector):         probability_vector = np.array(probability_vector)         order = (-probability_vector).argsort()  # Sort descending         support_mask = probability_vector > 0    # Identify the support         top_k_mask = np.zeros_like(probability_vector, dtype=int)         top_k_mask[order[:self.k]] = 1           # Mark top-k         partition = top_k_mask * support_mask.astype(int)  # Combine         return partition`

🧠 **Interpretación:**  
Un símbolo está en la partición final **solo si**:

- Está entre los `k` más probables (por orden de `argsort`).
    
- Tiene probabilidad estrictamente positiva (está en el soporte).
    

---

## 2. 🧪 Análisis de Ejemplos Clave

### Ejemplo:

python

CopyEdit

`p1 = [1.0, 0.0, 0.0] p2 = [0.9, 0.1, 0.0]`

#### Para `k = 1`:

|Vector|Top-1 Mask|Soporte Mask|Partición Final|
|---|---|---|---|
|`p1`|`[1,0,0]`|`[1,0,0]`|`[1,0,0]`|
|`p2`|`[1,1,0]`|`[1,1,0]`|`[1,0,0]`|

🔍 **Resultado:** Son **equivalentes** según el código, ya que el símbolo más probable y con soporte positivo es el mismo.

#### Para `k = 2`:

|Vector|Top-2 Mask|Soporte Mask|Partición Final|
|---|---|---|---|
|`p1`|`[1,0,0]`|`[1,0,0]`|`[1,0,0]`|
|`p2`|`[1,1,0]`|`[1,1,0]`|`[1,1,0]`|

🔍 **Resultado:** **No son equivalentes**. La segunda distribución incluye un segundo símbolo con probabilidad positiva dentro del top-k.

---

## 3. 🎯 Discusión Conceptual

### 3.1. ¿Qué está midiendo realmente `TopKProbabilityPartitionerPlus`?

Esta clase implementa una **forma refinada de equivalencia `top_k`**, donde:

> Se consideran equivalentes dos distribuciones si y solo si:
> 
> - Coinciden en sus símbolos más probables (top-k), **y**
>     
> - Dichos símbolos están presentes en el soporte (probabilidad > 0).
>     

Esto difiere sutilmente de la definición matemática, que compara los `top_k` **suponiendo igualdad de soportes globales**.

---

### 3.2. ⚠️ Conflicto entre Teoría y Práctica

Se discutió la precondición `supp(p) = supp(p')` como base para definir cualquier equivalencia `E`.

- ❌ **Problema**: En el caso de `top_k`, esta precondición **puede ser restrictiva o incluso inconsistente** con la intención de comparar solo los elementos más probables.
    
- ✅ **Solución adoptada**: Considerar una proyección previa (`samptop_k`) que mantiene sólo los `k` elementos más probables, luego compara.
    

> 🗣️ _"top_k naturalmente induce una equivalencia, aunque no necesariamente exige igualdad de soporte completo."_

---

## 4. 📌 Terminología Adicional: Prefix vs Word

Propuesta de distinción útil en el análisis:

|Término|Condición|Analogía|
|---|---|---|
|Generable Prefix (`u`)|`P(u) > 0`|"Reachable"|
|Generable Word (`u$`)|`P₍$₎(u) > 0` (palabra finalizada)|"Terminating path"|

✅ Aceptada por todos los participantes. Útil para interpretar prefijos y finales en autómatas probabilísticos o modelos generativos.

---

## 5. ✅ Conclusiones y Decisiones Finales

### 🧾 Decisión:

Para la congruencia asociada a `1_L` (función indicadora de aceptación), **se mantiene la precondición de igualdad de soporte completo**.

- Aunque dos distribuciones puedan coincidir en su `top_k`, si sus soportes globales difieren, **no se consideran equivalentes bajo esta congruencia**.
    
- Esta es una elección **consciente y deliberada**, que prioriza:
    
    - Coherencia algebraica
        
    - Simplicidad de implementación en cuantización
        
    - Consistencia con otras particiones utilizadas (e.g., `QuantizationPartitionerPlus`)
        

---

### 🔜 Próximos Pasos

- 📦 En la presentación, se usará la definición de equivalencia inducida por el código (`TopKProbabilityPartitionerPlus`), donde la intersección con el soporte es clave.
    
- 🧪 Se deja abierta la posibilidad de refinar esta equivalencia más adelante, eliminando la dependencia del soporte si se desea una partición puramente basada en top-k rankings.