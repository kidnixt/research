### 🧠 ¿Qué está pasando?

Tu código de sampleo hace esto:

- A cada paso, toma el último token (`probs = softmax(logits[:, -1, :])`).
- Filtra únicamente los tokens `"0"` a `"9"` (y opcionalmente `eos`).
- Luego samplea **por token individual**, y repite.

🔴 **Pero aquí está el punto crítico**:

El modelo **no genera dígitos como tokens individuales en todos los casos**. Algunos números como `"10"` o `"9"` pueden estar **tokenizados como un solo token** dependiendo del tokenizer de Qwen. Si tú sampleás por tokens individuales, pero luego decodificás (como si fueran caracteres), introduces un sesgo.


### ✅ Solución: Samplear secuencias de dígitos como en el wrapper

Vamos a replicar exactamente lo que hacés cuando calculás las probabilidades como en el wrapper:

1. **Tokenizás los strings `"0"` a `"9"`** como palabras, y te fijás cuáles tokens los componen.
2. Luego calculás su probabilidad como el producto de probabilidades **secuenciales**, no independientes.
3. El wrapper considera **palabras completas** y su secuencia de tokens.

---

La diferencia entre "como se hacía antes" (calculando las probabilidades de todos los tokens en una sola pasada del modelo) y "como se hace ahora" (paso a paso, token a token) es **más que una cuestión de consistencia**: es una diferencia fundamental en términos de **precisión, control y fidelidad al comportamiento autoregresivo del modelo**.

## 🔍 Comparación de técnicas

|Criterio|Técnica "antes" (una sola pasada)|Técnica "ahora" (paso a paso, token por token)|
|---|---|---|
|**Fidelidad al sampling autoregresivo**|❌ Baja — no refleja cómo el modelo genera texto|✅ Alta — exactamente como el modelo genera texto|
|**Consistencia con `ProbabilisticModel`**|❌ No es compatible|✅ Compatible|
|**Interpretabilidad**|❌ Difusa — se mezclan logits de distintos tokens|✅ Clara — logits precisos del siguiente token|
|**Control sobre el proceso**|❌ Difícil de modificar o debuggear|✅ Cada paso es claro, intervenible|
|**Costo computacional**|✅ Más eficiente (una sola inferencia)|❌ Más costoso (una inferencia por token)|
|**Escalabilidad**|✅ Más rápido en batch|❌ Más lento, pero más preciso|

## 🧠 ¿Cuál es mejor?

**Para tu caso actual**, en el que:

- Quieres comparar distribuciones empíricas con probabilidades exactas.
- Estás trabajando con un wrapper tipo `ProbabilisticModel`.
- Necesitás que el muestreo refleje las probabilidades reales.
- Estás haciendo investigación/análisis fino del modelo.

👉 **La técnica paso a paso (token a token) es claramente mejor.**

---

## ⚠️ ¿Cuándo usar la versión "rápida" (una sola pasada)?

- Si solo te interesa generar texto rápidamente (sin estudiar las probabilidades internas).
- Si no necesitas precisión token a token, sino simplemente muestras completas.
- Si estás generando texto largo en producción.

En esos casos, sacrificar algo de precisión puede valer la pena por la eficiencia.

---

### 🧠 ¿Cuál técnica es mejor?

#### ✅ Técnica actual (wrapper-style):

- ✔️ Calcula correctamente las probabilidades de salida **condicionales** basadas en todo el historial anterior.
- ✔️ Refleja lo que el modelo realmente "piensa" que debería venir después.
- ✅ Es la forma **teóricamente correcta** de evaluar modelos autoregresivos token por token.
- ❌ Ligeramente más costosa computacionalmente que usar solo el último logit (pero mucho más precisa).

#### 🆚 Técnica anterior (solo `logits[:, -1, :]`):

- ❌ Ignora tokens anteriores si no se usa correctamente el contexto total.
- ❌ Puede dar probabilidades inconsistentes si hay ambigüedad en el token que sigue.
- ❌ No coincide con lo que hace el wrapper ni el cálculo completo.


# 📘 Reporte Técnico: Consistencia de Probabilidades entre el Wrapper de Qwen3 y el Muestreo Autoregresivo

## 1. Introducción

En este proyecto se utiliza el modelo de lenguaje Qwen3-1.7B para generar números flotantes entre 0 y 1. Para analizar su comportamiento probabilístico, se emplean dos enfoques complementarios:

- Un **wrapper probabilístico** que implementa la interfaz de `Pythautomata` para extraer modelos estocásticos (como un PDFA).
- Un **script de muestreo autoregresivo**, que genera muestras directamente desde el modelo LLM.

Ambas estrategias deberían reflejar el mismo proceso subyacente de generación. Sin embargo, inicialmente se observó una **inconsistencia significativa** entre las distribuciones resultantes, lo que motivó un análisis técnico y una corrección del procedimiento de muestreo.

---

## 2. ¿Cómo calcula probabilidades el wrapper?

El wrapper para Qwen3 implementado sigue una lógica detallada y rigurosa para obtener probabilidades condicionales:

- Dado un prefijo (por ejemplo, `"0."`, `"0.3"`, etc.), se construye una secuencia de entrada completa concatenando un *prompt fijo* con el prefijo observado.
- El wrapper consulta al modelo Qwen3 para obtener la distribución de probabilidad del siguiente token.
- Luego, filtra únicamente los tokens relevantes para el alfabeto de interés: los dígitos `"0"` a `"9"` y el token de fin de secuencia (`<|endoftext|>`).
- Para cada símbolo del alfabeto, calcula su probabilidad en función del token ID correspondiente, considerando penalizaciones en caso de que esté compuesto por múltiples sub-tokens.
  
Este enfoque es **determinista, explícito y renormaliza correctamente** las probabilidades sobre el conjunto relevante de símbolos, asegurando que se interprete al modelo como una distribución válida para el análisis formal.

---

## 3. ¿Qué problema se detectó en el muestreo original desde Qwen3?

El script de muestreo original realizaba un proceso autoregresivo, generando números paso a paso. En cada paso:

- Obtenía los logits del último token del modelo.
- Aplicaba `softmax` para obtener probabilidades.
- Filtraba los tokens correspondientes a los dígitos y al fin de secuencia.
- Realizaba una muestra aleatoria según esas probabilidades.

El problema fundamental estaba en que **las probabilidades no se renormalizaban** tras el filtrado. Es decir, se tomaban las probabilidades directamente del `softmax` general, sin ajustar para que sumaran 1 dentro del subconjunto relevante.

Esto provocaba que la muestra se hiciera sobre una distribución **sesgada**, donde los valores eran proporcionales a las probabilidades "crudas" del modelo, pero **no reflejaban correctamente la distribución condicional sobre el conjunto `{0–9, <eos>}`**.

---

## 4. Descubrimiento de la inconsistencia

Para verificar la validez de ambos enfoques, se realizó un experimento comparativo:

- Se evaluó la distribución del primer dígito después de un punto decimal (`"."`) usando el wrapper (`last_token_probability`).
- Luego, se generaron múltiples muestras desde el modelo autoregresivamente, registrando el primer dígito generado.
  
Los resultados mostraron que **las distribuciones eran diferentes**, confirmando que el script de muestreo no estaba reproduciendo la misma distribución aprendida por el modelo que el wrapper sí capturaba correctamente.

---

## 5. Solución implementada

Se ajustó el script de muestreo para replicar exactamente el mismo procedimiento del wrapper:

- Se filtran los logits del modelo para quedarse únicamente con los correspondientes a los tokens del alfabeto.
- Se aplica `softmax` **solamente sobre esos logits filtrados**, asegurando que la distribución resultante esté renormalizada y sume exactamente 1.
- Las muestras se generan a partir de esta distribución válida, en concordancia con lo que hace el wrapper.

Con esta corrección, las distribuciones obtenidas por el script de muestreo comenzaron a coincidir casi exactamente con las del wrapper.

---

## 6. Conclusiones

- El enfoque del **wrapper** es más adecuado para representar la distribución aprendida por el modelo sobre un alfabeto restringido, porque **renormaliza correctamente** y se ajusta a un marco formal.
- El muestreo original era incorrecto desde el punto de vista probabilístico, ya que usaba probabilidades sin renormalizar tras el filtrado.
- El nuevo muestreo corrige esto, replicando el procedimiento del wrapper y garantizando **consistencia en el análisis y la generación**.
- Esta corrección es fundamental cuando se desea interpretar los comportamientos del modelo a través de autómatas estocásticos u otras herramientas formales.

---

## 7. Recomendación

Se recomienda que **todo procedimiento de muestreo que se base en subconjuntos del vocabulario del modelo realice una renormalización explícita** de las probabilidades, tal como lo hace el wrapper de Qwen3. Esto asegura consistencia, interpretabilidad y fidelidad al modelo subyacente.

