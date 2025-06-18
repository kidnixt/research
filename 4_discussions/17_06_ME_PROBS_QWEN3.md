# 📘 Reporte Técnico: Consistencia entre Wrapper Qwen3 y Muestreo Autoregresivo

## 1. 🧠 Contexto

Se trabaja con el modelo Qwen3-1.7B para generar números flotantes entre 0 y 1, y analizar la distribución probabilística de los dígitos generados.

Para ello, se usan dos métodos:

- Un **wrapper probabilístico** que implementa la interfaz `ProbabilisticModel` de Pythautomata, para calcular probabilidades token a token de forma exacta y formal.
- Un **script de muestreo autoregresivo** que genera secuencias paso a paso desde el modelo.

Ambos métodos deberían producir distribuciones consistentes, ya que reflejan el mismo proceso generativo autoregresivo.

---

## 2. 🔍 Problema Detectado

El muestreo original del script:

- Obtiene los logits del último token y calcula su softmax.
- Filtra tokens permitidos (`"0"` a `"9"` y `<eos>`).
- **Pero no renormaliza** las probabilidades tras filtrar el vocabulario.
- Luego muestrea el siguiente token basado en esas probabilidades no normalizadas.

Este procedimiento causa que la distribución del dígito siguiente (p.ej. primer dígito tras un punto decimal) no coincida con la distribución real implícita por el modelo ni con la calculada por el wrapper.

### Hist LM
![[Pasted image 20250618143844.png]]

### Hist PDFA

![[Pasted image 20250618143924.png]]


### Printeo de probs después del prompt "."

Se aprecia que no coincide con el histograma del LM

![[Pasted image 20250618144004.png]]

---

## 3. ✅ Solución Implementada

Para alinear el muestreo con el wrapper se corrigió el script para que:

- Después de filtrar los tokens de interés, se **renormalicen las probabilidades** (softmax solo sobre los logits filtrados).
- El muestreo se haga a partir de esta distribución correcta, que suma 1 sobre el conjunto restringido.
- Se samplea token a token, reflejando la naturaleza autoregresiva exacta del modelo.

---

## 4. 🧠 Por qué el wrapper es correcto

El wrapper sigue este enfoque:

- Dado un prefijo, construye la secuencia de entrada completa y consulta la distribución del siguiente token.
- Considera el alfabeto restringido (`"0"`-`"9"` + `<eos>`).
- Calcula la probabilidad de cada símbolo como producto de probabilidades de sus tokens (en caso de símbolos multitérmino).
- Renormaliza correctamente las probabilidades sobre el alfabeto.
- Así obtiene probabilidades condicionales exactas, compatibles con la definición formal de un modelo probabilístico autoregresivo.

---

## 5. ⚖️ Comparación técnica entre enfoques

|Aspecto|Muestreo Original (una pasada)|Muestreo Corregido (token a token)|
|---|---|---|
|Fidelidad al proceso autoregresivo|❌ Baja — no refleja generación real|✅ Alta — refleja generación exacta|
|Renormalización post-filtrado|❌ No hay|✅ Sí|
|Consistencia con wrapper|❌ No|✅ Sí|
|Interpretabilidad|❌ Difusa|✅ Clara|
|Costo computacional|✅ Más eficiente|❌ Más costoso|

---

## 6. 🎯 Conclusión

Para análisis fino, interpretación formal y comparación con autómatas estocásticos:

- **El muestreo token a token con renormalización** es fundamental para asegurar que las distribuciones reflejen fielmente lo aprendido por el modelo.
- Usar probabilidades sin renormalizar tras filtrado genera sesgos que distorsionan el análisis.
- El wrapper implementa el método teóricamente correcto, y el muestreo debe replicar su lógica para ser consistente.

---

## 7. ⚠️ Cuándo usar la versión rápida sin token por token

- Generación de texto rápida en producción, sin necesidad de analizar probabilidades precisas.
- Cuando no interesa la interpretación token a token, sino solo muestras completas.
- Para grandes volúmenes donde la eficiencia es prioritaria frente a la exactitud fina.

---

## 8. 🔑 Recomendación final

Siempre que se muestree restringiendo el vocabulario, **renormalizar explícitamente las probabilidades** sobre el subconjunto seleccionado es imprescindible para garantizar:

- Consistencia con el modelo.
- Interpretabilidad formal.
- Validez estadística.

Este principio fue confirmado y aplicado con éxito al corregir el script de muestreo para que coincida con el wrapper de Qwen3.