## EXPLICACIÓN DE LAS DOS ECUACIONES

### Ecuación original (Meier et al.)

La formulación original del efecto mutacional es:

∑t∈T[log⁡P(xt=stmt∣x∖T)−log⁡P(xt=stwt∣x∖T)]\sum_{t \in T} \left[ \log P(x_t = s_t^{mt} \mid x_{\setminus T}) - \log P(x_t = s_t^{wt} \mid x_{\setminus T}) \right]t∈T∑​[logP(xt​=stmt​∣x∖T​)−logP(xt​=stwt​∣x∖T​)]

**Qué significa cada término:**

- TTT: conjunto de posiciones mutadas
    
- xtx_txt​: token en la posición ttt
    
- stmts_t^{mt}stmt​: residuo mutado
    
- stwts_t^{wt}stwt​: residuo _wild-type_
    
- x∖Tx_{\setminus T}x∖T​: contexto restante (todas las demás posiciones)
    

**Interpretación:**  
El modelo compara, posición por posición, cuánto más probable es el residuo mutado que el original, **dado el mismo contexto**. La suma permite extender esto a mutaciones múltiples.

Esta ecuación asume un **vocabulario puramente secuencial** (aminoácidos).

---

### Modificación introducida por SaProt

SaProt redefine la probabilidad de un residuo porque su vocabulario es el **producto cartesiano**:

V×FV \times FV×F

donde:

- VVV: alfabeto de residuos (aminoácidos)
    
- FFF: alfabeto estructural (tokens 3Di de Foldseek)
    

Por eso, SaProt utiliza:

∑t∈T[log⁡∑f∈FP(xt=stmtf∣x∖T)−log⁡∑f∈FP(xt=stwtf∣x∖T)]\sum_{t \in T} \left[ \log \sum_{f \in \mathcal{F}} P(x_t = s_t^{mt} f \mid x_{\setminus T}) - \log \sum_{f \in \mathcal{F}} P(x_t = s_t^{wt} f \mid x_{\setminus T}) \right]t∈T∑​​logf∈F∑​P(xt​=stmt​f∣x∖T​)−logf∈F∑​P(xt​=stwt​f∣x∖T​)​

**Qué cambia aquí:**

- Cada residuo ya no es un único token
    
- Un residuo puede aparecer asociado a **múltiples tokens estructurales**
    
- La probabilidad del residuo se obtiene **sumando sobre todos los estados estructurales posibles**
    

---

### Diferencia conceptual clave

- **Ecuación original:**  
    → “¿Qué tan probable es este aminoácido?”
    
- **Ecuación SaProt:**  
    → “¿Qué tan probable es este aminoácido, considerando todos los contextos estructurales compatibles?”
    

Esto hace que el score de SaProt sea:

- más **robusto estructuralmente**,
    
- pero también **menos directamente interpretable** como una simple probabilidad de residuo.
    

Importante:  
👉 **La lógica del log odds ratio no cambia**, solo cambia **cómo se define la probabilidad del residuo**.

---