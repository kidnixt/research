### **Flujo completo de llamadas y procesos en el PDFA y WeightedAutomaton**

---

#### **1. Learner**

- El **learner** es el componente que genera hipótesis sobre el autómata y compara con el teacher.
- Flujo principal relevante:
    

```
counterexample = ...  # obtenido del EQ (Equivalence Query)
last_size = None
tree_history = []

if size == last_size:
    warnings.warn("Possible infinite loop")
last_size = size

teacher_prob = list(self._teacher.next_token_probabilities(counterexample).values())
model_prob = model.last_token_probabilities(counterexample, self._all_symbols_sorted)
ce_is_correct = self.probability_partitioner.are_in_same_partition(teacher_prob, model_prob)

while self._exhaust_counterexample and not ce_is_correct:
    self.update_tree(counterexample, model)
    model = self.tentative_hypothesis()
    models.append(model)
    model_prob = model.last_token_probabilities(counterexample, self._all_symbols_sorted)
    ce_is_correct = self.probability_partitioner.are_in_same_partition(teacher_prob, model_prob)

if verbose:
    self._tree.pretty_print()
tree_history.append(self._tree.copy())
```

**Observaciones:**

- El learner compara **las probabilidades del siguiente token** entre teacher y modelo.
- Llama repetidamente a `last_token_probabilities` del modelo para evaluar las hipótesis.
- Guarda historial de árboles (`tree_history`) y reconstruye el modelo si el CE no es correcto.
    

---

#### **2. ProbabilisticModel**

- `ProbabilisticModel` define la interfaz abstracta que debe implementar cualquier modelo probabilístico.
- Funciones principales involucradas:
    

```
last_token_probabilities(sequence, required_suffixes)
    → iterar sobre required_suffixes
        → last_token_probability(sequence + suffix)
            → PDFA.last_token_probability
```

- `last_token_probabilities` genera una lista de probabilidades para todos los sufijos requeridos.
- Internamente cada `last_token_probability` **invoca el método del PDFA**, que delega al WeightedAutomaton.
    

---

#### **3. ProbabilisticDeterministicFiniteAutomaton (PDFA)**

- Clase concreta que hereda de `WeightedAutomaton` y `ProbabilisticModel`.
- Implementaciones clave:

```
last_token_probability(sequence)
    → self.last_token_weight(sequence)[0]

sequence_probability(sequence)
    → self.sequence_weight(sequence)

log_sequence_probability(sequence)
    → self.log_sequence_weight(sequence)
```

- `last_token_probability` obtiene **el peso del último token** mediante `WeightedAutomaton.last_token_weight`.
    

---

#### **4. WeightedAutomaton**

- Componente base que maneja los pesos de estados y transicioes.
- Función clave para el flujo:
    

```
last_token_weight(sequence)
    → list(chain.from_iterable([
        _last_token_weight_from(sequence.value, state, state.initial_weight)
        for state in self.weighted_states if state.initial_weight > 0
    ]))
```

- Itera sobre todos los estados iniciales y llama a `_last_token_weight_from`.

---

#### **5. Función recursiva _last_token_weight_from**

```
_last_token_weight_from(sequence_value, state, weight):
    if sequence_value vacía:
        return [weight]
    elif sequence_value[0] == terminal_symbol:
        return [state.final_weight]
    else:
        transitions = state.transitions_list_for(sequence_value[0])
        next_states, weights = zip(*transitions)
        return chain.from_iterable(
            _last_token_weight_from(sequence_value[1:], next_state, w)
            for next_state, w in zip(next_states, weights)
        )
```

- Recorre la secuencia token por token.
- Para cada transición posible, llama recursivamente a sí misma.
- Produce una **lista combinatoria de pesos de todas las rutas posibles** que terminan en la secuencia dada.
    

---

#### **6. Flujo completo desde Learner hasta WeightedAutomaton**

```
Learner EQ loop
│
├─ teacher_prob = teacher.next_token_probabilities(seq)
├─ model_prob = model.last_token_probabilities(seq, required_suffixes)
│    │
│    ├─ ProbabilisticModel.last_token_probabilities
│    │    ├─ Itera sufijos
│    │    └─ last_token_probability(seq + suffix)
│    │          │
│    │          └─ PDFA.last_token_probability
│    │                │
│    │                └─ WeightedAutomaton.last_token_weight
│    │                      │
│    │                      └─ _last_token_weight_from (recursivo por cada transición)
│    │                             │
│    │                             └─ Genera todas las rutas posibles → lista de pesos
```

---

#### **7. Ineficiencias identificadas**

1. **Recursión combinatoria en `_last_token_weight_from`**:
    
    - Cada token genera llamadas recursivas para **todas las transiciones** del estado actual.
    - Para autómatas densos o secuencias largas, esto puede explotar combinatoriamente.
        
2. **Uso de `chain.from_iterable(map(...))`**:
    
    - Se crean muchos iterables intermedios en cada nivel de recursión.
    - La memoria y CPU crecen rápidamente con la longitud de secuencia y el número de transiciones.
        
3. **Cálculo repetido de pesos**:
    
    - No hay memoización ni acumulación dinámica.
    - Si una subsecuencia ya se calculó para un estado, se recalcula de nuevo.
        
4. **Multiplicación por `initial_weight` en cada ruta**:
    - Se hace incluso si muchas rutas tienen peso cero, generando llamadas innecesarias.
        
5. **Efecto combinado**:
    - `last_token_probabilities` en Learner → iteración sobre sufijos → `last_token_weight` → recursión → **explosión combinatoria**.
    - Esto se traduce en lentitud para EQ con secuencias largas o muchos sufijos.
        

