# 📊 Análisis y Clarificación de Equivalencias en Modelos de Probabilidad

**Fecha:** 17 de junio de 2025  
**Participantes:** Sergio Yovine, Franz Mayr, Matías Carrasco Piaggio, Martín Iturbide

---

## 1. 📌 Definiciones Fundamentales

### A. 🧮 Rango (`rank`) y Top-r (`top_r`)

- $\text{rank}(\delta) : \Sigma_s \rightarrow \mathbb{N}$  asigna un orden a cada símbolo $\sigma\in\Sigma_s$  según su probabilidad $\delta(\sigma)$. 
    - Se asume que $\text{rank}(\delta)$ es inyectiva (no hay empates). En caso de empate, se resuelve con un orden arbitrario (e.g., lexicográfico).
    - Si existen empates en $\delta(\sigma)$, se asume un desempate fijo, por ejemplo, orden lexicográfico. Esto garantiza que $\text{rank}$ sea una función bien definida.

- $\text{top}_r(\delta) = \{ \sigma \in \Sigma_s \mid \text{rank}(\delta)(\sigma) \leq r \}$: conjunto de los $r$ símbolos más probables (olvida su orden relativo interno).

---

### B. 🔁 Equivalencia Teórica $\rho =_{\text{top}_r} \rho'$ 

Según el paper:

> Dos distribuciones $\rho$ y $\rho'$ son equivalentes bajo $\text{top}_r$ si:
> 1. Tienen el **mismo soporte**: $\text{supp}(\rho) = \text{supp}(\rho')$
> 2. Sus conjuntos $\text{top}_r$ son iguales: $\text{top}_r(\rho) = \text{top}_r(\rho')$



$\rho =_{\text{top}_r} \rho' \iff$ $\text{supp}(\rho) = \text{supp}(\rho') \quad \text{y} \quad \text{top}_r(\rho) = \text{top}_r(\rho')$

⚠️ **Nota:** Esta definición ignora el orden interno dentro del top-r y sólo compara la pertenencia.
- Es decir, dos distribuciones pueden tener los mismos símbolos más probables pero en distinto orden, y aún así ser equivalentes bajo esta definición.


---

### C. ⚙️ Implementación: `TopKProbabilityPartitionerPlus`

El siguiente código implementa una partición basada en $\text{top}_r$ y el soporte:

``` python
class TopKProbabilityPartitionerPlus(ProbabilityPartitioner):
    def __init__(self, k) -> None:         
        self.k = k         
        super().__init__()      

    def _get_partition(self, probability_vector):         
        probability_vector = np.array(probability_vector)
        order = (-probability_vector).argsort()  

        # Sort descending         
        support_mask = probability_vector > 0    
        # Identify the support         
        top_k_mask = np.zeros_like(probability_vector, dtype=int)
        top_k_mask[order[:self.k]] = 1           
        # Mark top-k         
        partition = top_k_mask * support_mask.astype(int)  
        # Combine         
        return partition
```


Desde el punto de vista matemático, la clase implementa:

$$
\text{partition}(\rho) = \text{top}_r(\rho) \cap \text{supp}(\rho)
$$

Es decir, sólo se retienen los símbolos que están **tanto** entre los $r$ más probables **como** en el soporte.

🧠 **Interpretación:**  
Un símbolo está en la partición final **solo si**:
- Está entre los $k$ más probables (por orden de $\text{argsort}$).
- Tiene probabilidad estrictamente positiva (está en el soporte).

---

## 2. 🧪 Análisis de Ejemplos Clave

### Ejemplo:

p1 = [1.0, 0.0, 0.0]
p2 = [0.9, 0.1, 0.0]

#### Para k = 1:

| Vector | Top-1 Mask | Soporte Mask | Partición Final |
| ------ | ---------- | ------------ | --------------- |
| p1     | [1,0,0]    | [1,0,0]      | [1,0,0]         |
| p2     | [1,1,0]    | [1,1,0]      | [1,0,0]         |

✅ **Resultado:** Son **equivalentes** según el código, ya que el símbolo más probable y con soporte positivo es el mismo.

#### Para k = 2:

| Vector | Top-2 Mask | Soporte Mask | Partición Final |
| ------ | ---------- | ------------ | --------------- |
| p1     | [1,0,0]    | [1,0,0]      | [1,0,0]         |
| p2     | [1,1,0]    | [1,1,0]      | [1,1,0]         |

❌ **Resultado:** **No son equivalentes**. La segunda distribución incluye un segundo símbolo con probabilidad positiva dentro del top-k.

---

## 3. 🎯 Discusión Conceptual

### 3.1. ¿Qué está midiendo realmente `TopKProbabilityPartitionerPlus`?

Esta clase implementa una **forma refinada de equivalencia** $\text{top}_k$ donde:

> Se consideran equivalentes dos distribuciones $\rho$ y $\rho'$  si y solo si:
> - Coinciden en sus símbolos más probables $\text{top}_k$, **y**
> - Dichos símbolos están presentes en el soporte (probabilidad > 0).

Esto difiere sutilmente de la definición matemática, que compara los $\text{top}_k$ **suponiendo igualdad de soportes globales**.

---

### 3.2. ⚠️ Conflicto entre Teoría y Práctica

Se discutió la precondición $\text{supp}(\rho) = \text{supp}(\rho')$ como base para definir cualquier equivalencia $E$.

- ❌ **Problema**: En el caso de $\text{top}_k$, esta precondición **puede ser restrictiva o incluso inconsistente** con la intención de comparar solo los elementos más probables.
- ✅ **Solución adoptada**: Considerar una proyección previa ($\text{samptop}_k$) que mantiene sólo los $k$ elementos más probables, luego compara.
	- donde $\text{samptop}_k(\rho)$ denota la restricción de $\rho$ al conjunto $\text{top}_k(\rho)$.


> 🗣️ _"top_k naturalmente induce una equivalencia, aunque no necesariamente exige igualdad de soporte completo."_

---

## 4. 📌 Terminología Adicional: Prefix vs Word

Propuesta de distinción útil en el análisis:

| Término              | Condición                            | Analogía           |
| -------------------- | ------------------------------------ | ------------------ |
| Generable Prefix (u) | $P(u) > 0$                           | "Reachable"        |
| Generable Word (u$)  | $P_{\$}(u) > 0$ (palabra finalizada) | "Terminating path" |

✅ Aceptada por todos los participantes. Útil para interpretar prefijos y finales en autómatas probabilísticos o modelos generativos.

---

## 5. ✅ Conclusiones y Decisiones Finales

### 🧾 Decisión:

Para la congruencia asociada a $1_L$ (función indicadora de aceptación), **se mantiene la precondición de igualdad de soporte completo**. 
$\text{supp}(\rho) = \text{supp}(\rho')$

- Aunque dos distribuciones puedan coincidir en su `top_k`, si sus soportes globales difieren, **no se consideran equivalentes bajo esta congruencia**.
- Esta es una elección **consciente y deliberada**, que prioriza:
    - Coherencia algebraica
    - Simplicidad de implementación en cuantización
    - Consistencia con otras particiones utilizadas (e.g., `QuantizationPartitionerPlus`)


---

### 🔜 Próximos Pasos

- 📦 En la presentación, se usará la definición de equivalencia inducida por el código (`TopKProbabilityPartitionerPlus`), donde la intersección con el soporte es clave.
- 🧪 Se deja abierta la posibilidad de refinar esta equivalencia más adelante, eliminando la dependencia del soporte si se desea una partición puramente basada en top-k rankings.

---

## 1. **Preguntas de fondo (filosofía del modelado)**

### ❓ ¿Qué significa realmente que dos distribuciones sean "equivalentes"?

- ¿Debe esa equivalencia preservar información estructural completa (como el soporte)?
    
- ¿O basta con que preserven propiedades funcionales específicas (como el top-k para una tarea de clasificación)?
    

### ❓ ¿Hasta qué punto las definiciones matemáticas deben adaptarse a las necesidades de implementación?

- ¿Es válido ajustar una definición teórica si eso facilita una implementación más robusta, eficiente o útil en la práctica?
    

---

## 🧪 2. **Preguntas prácticas / de implementación**

### ❓ ¿Debe la equivalencia `top_k` depender del soporte completo?

- En muchos casos reales (e.g., sampling, generación de texto), **solo importa el top-k con probabilidad positiva**. ¿Tiene sentido exigir igualdad de soporte completo?
    

### ❓ ¿Cómo afecta esta definición a tareas downstream?

- Por ejemplo: ¿Dos vectores que difieren solo en símbolos muy improbables deben considerarse distintos?
    
- ¿Qué impacto tiene eso en cuantización, clustering o reducción de modelos?
    

---

## 🔄 3. **Posibles caminos futuros / decisiones abiertas**

### ❓ ¿Podríamos definir una familia de equivalencias parametrizadas?

- Por ejemplo, `EquivTopK_strict` (requiere igualdad de soporte) vs `EquivTopK_loose` (ignora soporte y se enfoca solo en top-k).
    

### ❓ ¿Queremos que las particiones sean _compatibles entre sí_?

- Es decir, ¿deberíamos exigir que dos particiones (top-k y cuantización, por ejemplo) se puedan comparar o componer coherentemente?
    

---

## 🎯 Conclusión clave que podés llevarte

> **La elección de qué considerar "equivalente" entre distribuciones no es solo una cuestión matemática, sino también epistemológica y práctica.**

Es una decisión que **afecta directamente el tipo de modelos que vas a construir, comparar, y cómo vas a interpretarlos.** Elegir bien esta equivalencia es tan importante como elegir una buena métrica o loss function.