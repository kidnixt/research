
### **Flujo de `last_token_weight(sequence)`**

1. **Llamada principal:**

```python
last_token_weight(sequence)
```

- Devuelve una lista de probabilidades de los _últimos tokens_ de todas las secuencias que terminan en estados finales.
- Internamente hace:

```python
list(chain.from_iterable([
    self._last_token_weight_from(sequence.value, y, y.initial_weight)
    for y in self.weighted_states if y.initial_weight > 0
]))
```

- **Itera sobre todos los estados iniciales** (`initial_weight > 0`).
- Para cada estado inicial, llama a `_last_token_weight_from` con el estado y su peso inicial.

---

2. **Función recursiva `_last_token_weight_from(sequence_value, state, weight)`**

- **Base case:** si la secuencia está vacía (`not sequence_value`), retorna `[weight]`.
- **Caso terminal:** si el primer símbolo de la secuencia es el `terminal_symbol`, retorna `[state.final_weight]`.
- **Caso general:** si hay más símbolos por procesar:
    - Recupera las transiciones posibles del estado actual para el símbolo actual:
        ```python
        transitions = state.transitions_list_for(sequence_value[0])
        ```
        
    - Desempaqueta:
        ```python
        next_states, weights = zip(*transitions)
        ```
        
    - Recurre para cada siguiente estado y peso:
        ```python
        return chain.from_iterable(
            map(lambda x, y: self._last_token_weight_from(sequence_value[1:], x, y),
                next_states, weights)
        )
        ```
        
- Esto genera **todas las posibles trayectorias** que corresponden a esa secuencia.
    

---

3. **Resumen visual del flujo de `_last_token_weight_from`**
```
_last_token_weight_from(seq, state, weight)
│
├─ si seq vacía → [weight]
├─ si seq[0] == terminal_symbol → [state.final_weight]
└─ sino:
   ├─ transitions = state.transitions_list_for(seq[0])
   ├─ next_states, weights = zip(*transitions)
   └─ para cada (next_state, w):
       llamar recursivamente _last_token_weight_from(seq[1:], next_state, w)
   └─ chain todos los resultados en una sola lista
```

---

4. **Ineficiencia principal**

- Cada llamada recursiva hace **una nueva llamada por cada transición**.
- Usa `chain.from_iterable(map(...))` lo cual **genera una gran cantidad de objetos intermedios**.
- Para secuencias largas y autómatas con muchos estados/transiciones, esto **explota combinatoriamente**, porque calcula **todas las trayectorias posibles**, incluso si solo necesitamos la probabilidad final de un token.
- No hay memoización ni acumulación de pesos por ruta; todo se recalcula recursivamente.

---

5. **Uso típico desde `ProbabilisticModel`:**

```python
last_token_probabilities(seq, required_suffixes)
    -> iterar sobre required_suffixes
        -> last_token_probability(seq + suffix)
            -> PDFA.last_token_probability
                -> WeightedAutomaton.last_token_weight
                    -> _last_token_weight_from
```

- Cada `last_token_probabilities` genera **una lista de llamadas recursivas** para cada sufijo.
- Para N sufijos y secuencias de longitud L, la complejidad es aproximadamente `O(N * (#paths in automaton for seq + suffix))`.

