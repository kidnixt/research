# Towards a Probabilistic Framework for Analyzing and Improving LLM-Enabled Software

## Abstract

- Los sistemas basados en LLMs son potentes pero **no deterministas.** Es difícil confiar plenamente en lo que generan o verificar formalmente su comportamiento.
- La **idea central** del paper es: no mirar los *outputs* del LLM como cadenas de texto, sino como ***clusters of semantically equivalent outputs***. 
	- Por ejemplo: todas las respuestas que formalizan una misma propiedad en un código.
- ***"Transference Models (TMs):*** concepto definido por los autores, son **LLM center modules** que reciben una entrada (ej. una descripción) y generan una salida (ej. especificación formal). Analizar como se comportan los TMs bajo el LLM es la base del trabajo.
- La **aplicación concreta** elegida en el paper es, dado un *docstring*, el modelo debe generar **precondiciones** y **postcondiciones** formales en Dafny.
- Al mirar no solo qué responde el modelo, sino cómo reparte la probabilidad entre distintas interpretaciones, se pueden **identificar patrones de error sistemáticos**, por ejemplo, cuando el modelo siempre genera una postcondición débil. 

> Los sistemas que usan LLMs son díficiles de verificar y depurar. Proponen un enfoque no analiza solo las respuesta, sino la distribución de probabilidad **sobre clases semánticas**. Esto los ayuda a identificar errores sistemáticos, como respuestas incorrectas con alta confianza. Lo ilustran con el problema de transformar lenguaje natural en especificaciones formales (*autoformalización*). 


## I. Introducción

Los LLMs (como GPT-4 y Gemini) funcionan muy bien en muchas tareas gracias a que pueden seguir instrucciones (*"instruction following"*) y aprender del contexto. Esto incluye estrategias como:
- **Zero-shot:** Darle una instrucción y esperar que entiende qué hacer sin ejemplos.
- **Few-shot:** Darle algunos ejemplos en el prompt.

Esta flexibilidad hace que los LLMs se usen cada vez más en aplicaciones reales (*downstream tasks* = tareas derivadas), simplemente dándoles prompts. La **alucinación**, es uno de los principales problemas que esto trae. Esto es, el modelo genera cosas que suenan bien, pero son falsas o inválidas. 

Se necesita **entender el comportamiento del sistema.** En el ámbito de ingeniería de software (de estas cosas), el comportamiento estocástico del modelo es visto más como un problema (por ejemplo, tests que fallan a veces y otras no: *flaky tests*), y no como algo que se pueda analizar o aprovechar

---
⚠️ Se está ignorando que el LLM no da una sola respuesta, sino una **distribución sobre muchas posibles**??

--- 

La propuesta central es:
1. Identificar qué partes del sistema están usando LLMs para transformar información (los **Transference Models**, TMs).
2. Luego, hay que modelar no las salidas en sí, sino la **distribución de probabilidad sobre clases de significados equivalentes** (lo que llaman *meaning classes*).


**TESIS:** Si entendés cómo el LLM reparte probabilidad entre interpretaciones posibles, podemos analizar y mejorar el sistema.


---

**HIPÓTESIS:**
Si el modelo da una respuesta equivocada **con alta confianza** (concentración en una clase errónea), probablemente esa clase de error **tiene una explicación semántica clara**, y por lo tanto se puede **diagnosticar y corregir** el TM.

> Cuando el modelo se equivoca con confianza, dice mucho sobre sus puntos débiles y se puede usar eso para mejorar(lo).


### Mis preguntitas:

- El agrupamiento de clases, no depende muchísimo de la tarea??
- Si una tarea no tiene una única "clase correcta", cómo se define la alineación??
- Es viable para tareas con millones de posibles salidas?

## II. Preliminares

### A. Large Language Models

#### ¿Qué es un LLM desde el punto de vista técnico?

Desde el punto de vista técnico, un LLM, básicamente es una **función probabilística paramétrica**. Es decir:
- Toma un input $x$ (un texto anterior, también llamado "contexto").
- Y devuelve una distribución de probabilidad sobre posibles próximos tokens $t_k$:
$$ P(t_k | x)$$ Esto significa que el modelo no genera "una respuesta", sino una **distribución sobre todas las respuestas posibles,** token por token.

---

#### ¿Cómo se entrena un LLM?

A través de _next-token prediction_ (predicción del próximo token). Se entrena con **corpus masivos** de texto, ajustando sus parámetros para minimizar un error (usualmente entropía cruzada) entre:

- lo que el modelo predice como próximo token, y
- lo que realmente sigue en el texto de entrenamiento.

⚠️ Esto refuerza la idea de que el modelo aprende **probabilidades de continuación**, no reglas explícitas o verdades lógicas.

---
#### ¿Cómo se usa un LLM en la práctica?

Una vez entrenado, el modelo puede usarse para generar texto. Para hacerlo, necesita un **decoding strategy**, como:

- _Greedy_: elige siempre el token más probable.
- _Sampling_: elige aleatoriamente según la distribución.
- _Top-k_ / _Top-p_: limita la elección a los tokens más probables.
- _Temperature_: controla cuán "creativo" es el modelo (más temperatura = más aleatoriedad).

Esto es crucial porque **el mismo input puede generar diferentes salidas** si el decoding es estocástico.

---
#### Conexión con el paper

Los autores enfatizan que si bien muchas aplicaciones usan el LLM como una caja negra, ellos quieren **modelar esta distribución** para entender el comportamiento del sistema. No les interesa solo “qué salida dio”, sino:

> **¿Con qué probabilidad eligió esa salida sobre otras posibles equivalentes?**

Esto los lleva a analizar _distribuciones sobre clases semánticas_, no solo sobre tokens.

---

### B .Autoformalization

#### ¿Qué es la autoformalización?

Autoformalización es el proceso de **convertir texto natural en especificaciones formales.** Es decir, traducir algo como:

> `"""Devuelve el índice del primer elemento igual a x en el array."""`

en algo como (en lenguaje formal como **Dafny**)

``` dafny
requires a != null
ensures forall i :: 0 <= i < result ==> a[i] != x
ensures a[result] == x
```

Esta tarea requiere **comprensión semántica**, **capacidad de inferencia lógica** y conocimiento del dominio del código. No es simplemente traducir palabra por palabra.


---
#### ¿Por qué es importante?

Porque automatizar esta transformación ayuda a tareas críticas como:

- **Verificación automática de programas** (probar que el código cumple con lo especificado).
- **Pruebas formales** (testings guiados por propiedades).
- **Documentación ejecutable** (donde los comentarios se vuelven parte verificable del programa).

Esta es una **tarea dura**, tradicionalmente realizada por expertos. Usar LLMs para esto es ambicioso, pero también muy prometedor.

---

#### ¿Por qué eligieron Dafny?

- Dafny es un lenguaje diseñado para **verificación automática** de programas, con soporte para precondiciones, postcondiciones, invariantes, etc.
- Tiene un **verificador SMT incorporado**, que permite comprobar si las condiciones formales son lógicamente correctas.
- Es usado frecuentemente en papers académicos para tareas de autoformalización y synthesis.

Además, como el objetivo del paper es evaluar distribuciones sobre significados formales, usar Dafny permite chequear automáticamente **equivalencia semántica** entre dos especificaciones (vía SMT).

---
#### ¿Qué rol juega esta tarea en el paper?

Es el **caso de estudio** principal que se usa para aplicar el marco probabilístico. Todo el análisis (alineación, concentración, mejoras al TM) se ilustra a partir de esta tarea. Lo interesante es que:

- La tarea tiene un **ground truth claro** (las especificaciones correctas están dadas).
- Se puede generar muchas salidas (por el LLM) y verificar si son **equivalentes** entre sí.
- Es posible categorizar errores comunes (por ejemplo, _postcondición débil_, _omisión de propiedad clave_, _error de sintaxis_).

![[Pasted image 20250613102054.png]]

---
### C. Transference Models (TMs)

#### ¿Qué es un Transference Model?

Es una **abstracción de ingeniería de software** que encapsula cómo un sistema basado en LLM **transforma inputs en outputs** para una tarea particular.

- Es una **unidad funcional** dentro de un sistema más grande.
- Se apoya en un LLM para llevar a cabo una transformación (por ejemplo: docstring -> especificación Dafny).
- Puede ser tan simple como un prompt, o más complejo (con herramientas externas, reintentos, submódulos, etc)

``` python 
def doc2spec(docstring):
    prompt = "Given this docstring, write the pre/post conditions: " + docstring
    return LLM(prompt)
```

Este `doc2spec` sería un **TM**.

---
#### ¿Por qué los TMs son estocásticos?

Porque dependen de un LLM, y los LLMs son modelos **probabilisticos:** Dada una entrada $i$, el LLM puede producir distintas salidos $o$, cada una con una probabilidad. Formalmente:
$$T: \mathcal{I} \times \mathcal{O} \rightarrow \mathbb{R} \quad\text{donde}\quad T(i,o) = P(o|i)$$

O sea, el TM **define una distribución** sobre salidas posibles para cada entrada. Esto depende fuertemente de:
- El LMM usado.
- El prompt.
- La **estrategia de decodificación** (ej. sampling con temperatura, top-k, etc).

---

#### ¿Qué incluye un TM?

Un TM puede incluir no solo el llamado al LLM, sino también:

- **Preprocesamiento** de la entrada (por ejemplo, limpiar el docstring).
- **Razonamiento externo** (RAG, vectores, heurísticas).
- **Postprocesamiento** (validar la salida con el compilador).
- **Loops correctivos**, donde se itera sobre la salida si hubo errores.
- **Cadena de TMs** (uno genera una subparte, otro compone, etc.).

En el paper, muestran un **TM con múltiples etapas**:

1. Prompt inicial para generar la especificación.
2. Si Dafny da error, se corrige usando el mensaje del compilador.
3. Además, usan un paso estilo RAG para buscar ejemplos similares (doc2vec + retrieval).

---

#### ¿Por qué es tan importante este concepto?

Porque los autores **NO** están interesados en estudiar al LLM directamente, sino en estudiar **los sistemas construidos con LLMs**. Es decir:

> "El LLM no es el sistema, sino una parte de un módulo más grande que transforma datos para una tarea."

Entonces el **objeto de análisis** del paper no es el modelo GPT o Gemini, sino el **TM** como componente de software. El marco propuesto evalúa cómo el TM se comporta probabilísticamente:
- Genera siempre la salida correcta?
- Qué tanta certeza tiene?
- Hay errores frecuentes con alta probabilidad?

---
#### Modelo funcional formal

Para reforzar: el TM se modela como una función probabilística:

$$T(i, \cdot) = \text{una distribución de probabilidad sobre salidas }o \text{ dado un input } i  $$
En otras palabras: **el TM actúa como una caja negra estocástica** que recibe una entrada y escupe una distribución sobre posibles resultados.

Lo que ellos proponen es: **muestrear esta caja negra** muchas veces con el mismo input, agrupar las salidas en clases de significado equivalentes y analizar esa distribución.

---
### D. Distribution over Semantic Domains

Dado que los LLMs son funciones de estimación de densidad sobre secuencias de tokens, los autores coinciden con otros trabajos que la probabilidad debería considerarse asignada a conceptos, no a cadenas de texto. Muchas veces hay múltiples (incluso infinitas) cadenas que representan una misma idea (por ejemplo, especificaciones lógicamente equivalentes). En teoría, esos corresponde a sumar las probabilidades asignadas a todas las cadenas dentro de cada clase de equivalencia. 

#### ¿Cuál es el problema de trabajar con cadenas de texto?

Los LLMs predicen secuencias de tokens (palabras o símbolos). Eso genera una **distribución sobre strings como:**

```
"Returns the index of x" → 20%  
"Find the position of x" → 18%  
"Locate x in the array" → 12%  
```

**Pero** estas cadenas pueden tener el **mismo significado**, aunque estén escritas distinto, entonces:

> Analizar solo la cadena más frecuente o el token más probable es **insuficiente** para entender qué está "pensando" el modelo.

---

#### ¿Qué proponen entonces?

Agrupar las salidas en **clases semánticas equivalentes** (lo que llaman *meaning classes*). Cada clase agrupa todas las salidas que, aunque tengan forma distinta, expresan la **misma idea.** Por ejemplo si dos fórmulas Dafny dicen: 

``` dafny
ensures exists i :: a[i] == x && forall j < i :: a[j] != x
```

``` dafny
ensures the returned index is the first occurrence of x
```

Estas pueden considerarse **semánticamente equivalentes** (si ambas implican lo mismo). Por tanto, deberían pertenecer a la misma clase. Entonces la **probabilidad total del "concepto"** debería ser:

$$P(\text{clase}) = \sum_{\text{strings en la clase}} P(\text{string})$$


---
#### ¿Por qué este punto es crucial?

Porque al trabajar con LLMs:
- El modelo puede generar miles de variantes textuales **de una misma respuesta semántica**
- Si evaluás solo por exact matching o string similarityu, **subestimás la probabilidad del concepto correcto.**

👉 Entonces, los autores abogan por construir distribuciones **no sobre cadenas**, sino sobre **dominios semánticos abstractos**.

Este cambio de perspectiva es **clave** para su marco probabilístico. Les permite:

- Medir correctamente alineación con la verdad.
- Detectar errores semánticos aunque el texto “suene bien”.
- Evaluar si el modelo es “confuso” o “seguro” semánticamente, no solo superficialmente.

---
### E. Computing Empirical Distributions of TMs

Dado que los TMs se comportan como procesos estocásticos y no siempre se tiene acceso directo a las probabilidades del *next-token*, se propone aproximar la distribución de salidas del TM usando una distribución categórica empírica sobre las *meaning classes*. Estas clases se obtiene **re-ejecutando** el TM múltiples veces con el mismo input y agrupando las salidas segun una relación de equivalencai semántica (por ejemplo, usando un SMT solver).

---

#### ¿Qué problemas enfrentan?

Idealmente, para cada input $i$, querríamos conocer la distribucion completa $P(o | i)$, es decir, cuánta probabilidad asigna el TM a cada posible output. Pero eso **no es accesible directamente** por varias razones:

- La mayoría de las APIs de LLM **no exponen la distribución completa de tokens.**
- Incluso si lo hicieran, reconstruir la distribución sobre *meaning classes*, supongo yo es **incomputable**.
- El espacio de posibles salidas $o \in \mathcal{O}$ es **enorme o infinito**, especialmente si se cuenta texto natural. 

---

#### ¿Qué solución proponen?

Usar una **distribución empírica** construida por muestreo:

1. Se re-ejecuta el TM varias veces con la **misma entrada** $i$. ⚠️ Cada ejecución genera una salida distinta por el carácter estocástico del LLM (sampling, temperatura, etc.).
2. Se recolectan todas las salidas $o_1, o_2, \dots, o_n$.
3. Se agrupan esas salidaas en **clases semánticamente equivalentes**— _meaning classes_ 
	1. En autoformalización, esto se hace verificando **equivalencia lógica** de las pre/post condiciones generadas.
	2. Si dos salidas son equivalentes según un STM solver, pertenencen a la msima clase.
4. Se cuenta cuántas veces apareció cada clase. Eso forma una **distribución categórica empírica:**

$$\hat{P}_i(c) = \frac{\text{cantidad de veces que se generó la clase } c}{\text{total de ejecuciones}}$$
Esta es la distribución sobre **conceptos**, no sobre cadenas de texto.

---
#### ¿Cómo se agrupan las salidas?

Esto depende fuertemente de la tarea. En autoformalización, dos especificaciones son equivalentes si:

- Tienen la misma **semántica lógica** (no importa si están escritas distinto).
- Esto se evalúa usando **SMT solvers** (como Z3 o el backend de Dafny).

📌 Importante: si la salida tiene **error sintáctico**, se considera su propia clase (no se agrupa con ninguna otra).

---

#### ¿Qué significa "comportamiento estocástico del TM"?

El TM puede comportarse como una caja negra:  
Cada vez que le das un input iii, responde con una salida diferente ooo, dependiendo de:

- El modelo base (GPT, Gemini, etc.).
- El prompt.
- Los hiperparámetros del decoding (temperatura, top-k, etc.).

Entonces, lo modelan como un **proceso aleatorio**:  
Repetís muchas veces y ves qué patrones emergen.

Esto también permite estudiar **certeza del modelo**: si siempre responde igual, hay concentración; si reparte entre muchas clases, hay dispersión.

---

#### ¿Por qué es importante este paso? 

Porque **toda la evaluación de alineación y concentración** se basa en esta distribución empírica:

- Si la clase correcta tiene la mayor frecuencia ⇒ **distribución alineada.**
- Si una clase incorrecta domina ⇒ **concentración pero desalineada (peligroso).**
- Si la distribución está dispersa ⇒ **incertidumbre o ambigüedad**

---
#### Mis Preguntitas:

- ¿Cuántas muestras son suficientes para estimar bien la distribución? (En el estudio usan 30).
- ¿Qué pasa si el solver SMT falla o es muy lento? ¿Cómo escalar esto?
- ¿Cómo definir equivalencia semántica en tareas donde no hay lógica formal?
- ¿Qué impacto tienen los hiperparámetros del LLM (p. ej. temperatura = 0.7 vs. 1.0) sobre la distribución?

---

## III.  The Framework 
### A. Assumptions, Hypothesis and Rationale

Los autores presentan **su hipótesis central**, los supuestos que hacen sobre los TMs y los fundamentos que justifican su enfoque probabilístico. Quieren responder esta pregunta:

> ¿Por qué analizar la distribución de significados generada por un TM es una buena forma de evaluar y mejorar sistemas con LLMs?.

---
#### Supuesto 1:

> La mayoria de los sistemas LLM pueden modelarse con uno o más TMs

Significa que cualquier sistema que use un LLM (incluso los complejos) puede representarse como un conjunto de **transformaciones de entrada a salida** que dependen del modelo.

- Un chatbot → TM: prompt → respuesta.
- Un sistema de generación de código → TM: docstring → código.
- Un sistema de razonamiento encadenado → varios TMs combinados.

---
#### Supuesto 2:

> Las tareas de los TMs pueden entenderse como tareas humanas específicas.

Esto quiere decir que cada TM **representra una tarea clara y humana-interpretable** (como generar especificaciones, traducir código, etc.), y sus salidas pueden evaluarse en términos de *clases de resultados equivalentes.* Esto es clave porque:

- Permite definir si **una salida es correcta o no.**
- Permite agrupar salidas **semánticamente equivalentes** (*meaning classes*)
- Permite evaluar el modelo de forma significativa, no solo superficial.

---
#### Hipótesis Principal

> El comportamiento de un TM se puede caracterizar como una **distribución probabilística** sobre conceptos.

Es decir, *dado un input $i$, el TM genera una distribución sobre distintas **clases de significados posibles** (no solo strings)*

Y lo que importa no es solo si genera alguna respuesta correcta, sino **cuánta probabilidad se le asigna.** Esta visión lleva a **dos nociones clave:**
- **Alineación:** ¿la clase más probable es la correcta?
- **Concentración:** ¿la distribución está concentrada en una clase o es difusa?

---
#### Hipótesis Complementaria

> Los casos **concentrados** y **desalineados** (o sea, equivocación con mucha confianza) son los más peligrosos.

Estos son los que:
- Pueden **engañar mecanismos de detección de alucinación** basados en certeza (e.g. si algo tiene alta probabilidad, parece confiable).
- Producen **errores sistemáticos** difíciles de detectar.
- Son candidatos ideales para el análisis y reingeniería del TM.

---
#### Relación con *refinamiento* en verificación formal

Proponen que mejorar un TM se parece a **refinar un programa** en el sentido de subtipado de comportamiento (como Liskov/Wing). Un TM mejorado, debería:

- Generar más outputs correctos con alta certeza.
- Reducir los casos donde se confunde con alta certeza.
- No romper los casos donde ya funcionaba bien.

Esto sugiere que **una mejora no debe dañar lo que ya funciona,** sino ampliar y corregir su comportamiento, una visión **ingenieril** y **formal** del proceso de mejora. 

---
#### Hipótesis adicional

> Cuando una clase incorrecta es la **dominante,** su **"tipo de error"** se puede **verbalizar en términos específicos de la tarea**

Por ejemplo, en autoformalización, hay tipos de errores conocidos:
- Precondición muy débil
- Postcondición incorrecta
- Sintaxis inválida.

Cuando un TM se equivoca con alta certeza, muchas veces lo hace de una forma que **esta dentro de una de estas categorías conocidas**. Esto permite **diagnosticar fallos automáticamente** y **guiar ajustes específicos**.

---

### B. Defining Improvement

#### ¿Qué es una distribución alineada?

Para un input $i$, se generan múltiples outputs y se agrupan en **clases de significado.**

- Se dice que la distribución es **alineada** si la clase de significado **con mayor probabilidad** (la clase ganadora), es **correcta.**
- Si la clase correcta está presente pero no es la más probable ⇒ **desalineada**.
- Si la clase correcta **ni siquiera aparece** en la muestra ⇒ **fuertemente desalineada**.

---
#### ¿Qué es una distribución concentrada?

Una distribución se considera **concentrada** si la clase ganadora tiene **probabilidad mayor o igual que la suma del resto:**

$$P(\text{clase ganadora}) \geq \sum_{\text{c}\neq\text{ganadora}} P(\text{c})$$

Alternativamente, podríamos decir que tiene más del 50% de la masa total. Esto es una **aproximación pragmática** a la noción de "certeza semántica" del modelo. 

---
#### Clasificación de outputs según estas dos dimensiones

Esto genera una **matriz 2x2** de escenarios para un input:

|                 | **Concentrada** | **No Concentrada** |
| --------------- | --------------- | ------------------ |
| **Alineada**    | ✅ Ideal         | ⚠️ Incierto        |
| **Desalineada** | ❌ Peligroso     | ❓ Ambiguo          |
- **Concentrada + alineada**: el TM sabe lo que hace.
- **Concentrada + desalineada**: error con alta confianza → muy peligroso.
- **No concentrada**: el modelo está confundido o disperso → menos riesgoso pero requiere atención.

---
#### ¿Qué significa que un TM mejora a otro?

Supongamos que tenemos dos TMS, $t$ y $t'$, y queremos decir que $t'$ **mejora a** $t$. La mejora se define **punto por punto** sobre un conjunto de inputs de prueba.

**Requisitos:**
- $t'$ genera **más casos alineados y concentrados**.
- $t'$ genera **menos casos concentrados y desalineados**.

Eso significa que el nuevo TM:

- Es más seguro (se equivoca menos con alta certeza).
- Es más confiable (tiene más confianza en respuestas correctas).

---
#### Definición general de mejora

El texto menciona la **definición probabilísitca generalizada**, sobre todo el dominio de inputs, usando una distribución $D$ sobre los inputs:

**Mejor con respecto a $D$**:

1. Con **alta probabilidad**, un input $i \sim D$ tiene:
	1. Mayor probabilidad de alineación + concentración con $t'$ que con $t$.
2. Si un input ya estaba bien con $t$, también lo está con $t'$ ⇒ **no regresión**.
3. El número de inputs que son **concentrados y desalineados** debe ser menor o igual en $t'$.

---
## IV. Illustrative Results on Autoformalization

Muestran, con un experimento concreto, que:
- Es posible construir y analizar distribuciones de significado generadas por un TM.
- Los conceptos de alineación y concentración son útiles para **diagnosticar errores sistemáticos**.
- Estas observaciones pueden guiar **mejoras efectivas** en el TM.

---
#### Dataset CloverBench

Consta de **62 programas simples** (estilo textbook). Cada uno tiene:
- Una implementación en **Dafny**
- Su **docstring** (input al TM)
- Especificacioens formales (pre/postconditions) como **ground truth**
- La firma del método.

📌 El TM toma el docstring + firma, y debe generar las especificaciones correctas.

---
#### Cómo ejecutan el TM

- El TM se ejecuta **30 veces por input** para capturar su comportamiento estocástico.
- Usan el modelo **Gemini 1.5 Flash** con:
	- Contexto de1000 tokens.
	- top-k = 40
	- top-p = 0.95
	- Temperatura = 0.7
- Si se genera una salida inválida (sintácticamente incorrecta), se la trata como una **clase aparte** (no se agrupa con ninguna otra)
- Las clases de salida se agruapn usando un **solver SMT** para deterctar equivalencia semántica entre especificaciones. 

Este setup permite construir una distribución empírica $\hat{P_i}(c)$ para cada input $i$, sobre las clases de significado $c$.

---
### Table I: Resultados de Alineación y Concentración

Resumen de cuántas entradas (de las 62) caen en cada una de las 4 categorías combinando alineación y concentración:

|                                 | **Concentrada** | **No concentrada** |
| :------------------------------ | :-------------: | :----------------: |
| **Alineada**                    |       46        |         3          |
| Correcta pero no es la ganadora |        2        |         3          |
| Correcta no generada            |        4        |         4          |

Lo que muesta esta tabla:
- En **49 casos** (46 + 3), la distribución estaba alineada (la clase correcta fue la más probable)
- En **54 casos**, la clase correcta apareció.
- En **46 casos**, la distribución fue **alineada y concentrada**

Esto **confirma que el análisis empírico capta cuándo el modelo realmente “entiende” la tarea**.

---
#### Comparación con "zero-temp"

Mencionan que si se ejecuta el TM en modo determinista (temperatura = 0), se obtienen **47 outputs alineados**.

Pero esto puede ser engañoso:
- Algunos outputs correctos eran en realidad parte de **distribuciones muy dispersas**.
- Otros outputs eran **incorrectos**, aunque parecían seguros.

**Solo viendo la primera respuesta no se revela el comportamiento real del modelo**. Se necesitan muestras múltiples.

---
### Table II: Tipos de erorres en casos desalineados.

De los **14 casos desalineados**, 11 corresponden a errores conocidos de formalización:

|Tipo de error|Concentrado|No concentrado|Total|
|---|---|---|---|
|Postcondición débil|2|3|5|
|Postcondición incorrecta|1|2|3|
|Error de sintaxis|1|2|3|
|Precondición débil|1|0|1|
|Precondición fuerte|1|0|1|

Esto apoya una de las hipótesis centrales del paper: **los errores del TM pueden clasificarse semánticamente**, lo que permite guiar su reingeniería.

Ejemplo concreto: en el caso _LinearSearch_, la postcondición no dice que el índice devuelto es el **primero**. Es un fallo típico: el modelo no compone correctamente subpropiedades ⇒ lo que se llama el **compositional gap**.

---
#### Mejora del TM

Hacen una modificación al TM:
- Agregan una **tarea previa** que clasifica cada oración del docstring como *precondición*, *postcondición* o *ninguna*.
- Esto ayuda a estructurar mejor la entrada antes de generar la especificación.

---
### Table III: Resultados tras la mejora

| |**Concentrada**|**No concentrada**|
|----------------------------|------------------|------------------------|
|**Alineada**|57|1|
|Correcta pero no es la ganadora|0|0|
|Correcta no generada|2|2|

- Pasan de 46 → **57 casos alineados y concentrados**.
- Los casos peligrosos (concentrados pero desalineados) bajan de 6 a 2.
- Estos dos casos (**modify_2d_array**, **on-line-max**) son ahora candidatos claros para una nueva ronda de mejora.

---
### Conclusiones de la sección

- El marco propuesto **permite identificar debilidades del TM** (tipos de errores frecuentes).
- Los conceptos de **alineación** y **concentración** son **útiles para priorizar qué inputs investigar y mejorar**.
- Pequeñas modificaciones (como reestructurar el docstring) pueden tener **gran impacto** en el comportamiento del sistema.

---
### Mis preguntitas

- ¿Cuánto mejora el TM por la modificación en sí y cuánto por el re-análisis del input?
- ¿Se podría automatizar la identificación de clases de error para hacer ajuste automático?
- ¿Cómo cambia este comportamiento con otros modelos (p. ej. GPT-4, Claude)?
- ¿Cuán sensible es el análisis al número de muestras (30 ejecuciones)?

---
## V. Related Work

### Surface Form Competition [7]

> Este concepto, introducido por Holtzman et al., muestra que a veces la respuesta más probable **no es la correcta** porque hay muchas formas de decir la respuesta correcta, pero solo una forma de decir la incorrecta.

- Apoya la idea de que **la probabilidad debería asignarse a significados**, no a cadenas individuales.
- Refuerza la necesidad de **agrupar por clases semánticas** y no quedarse con la primera respuesta generada.

---
### Detección de alucinaciones mediante entropía semántica [4]

> Farquhar et al. proponen medir **entropía sobre clases semánticas** como forma de detectar alucinaciones en LLMs.

- Se alinea con la idea de que los modelos no alucinan solo cuando tienen alta entropía.
- Este paper **va más allá**, al usar esa información para **mejorar el sistema**, no solo para detectar errores.

---
### Clover y el concepto de Transference Model [17]

> El trabajo _Clover_ introdujo el uso de LLMs para generar especificaciones verificables, y ya incluía la idea de **modelar el proceso como un TM**, aunque no de forma tan formal.

Este paper:
- Toma esa idea y la **expande formalmente**.
- Introduce nociones cuantificables como **alineación** y **concentración**.
- Analiza la distribución entera, no solo el primer output.

---
### DSPy y generación declarativa de prompts [9]

> DSPy permite construir sistemas declarativos donde los prompts son tratados como componentes ajustables del sistema.

Aunque no es el foco central del paper, DSPy representa el tipo de **ingeniería basada en LLMs** donde este marco de análisis sería útil para evaluar y refinar componentes.

---

### Conclusiones de la sección

> “Estos trabajos previos muestran que los conceptos de equivalencia semántica, distribución de outputs y comportamiento no determinista del LLM son relevantes, **pero ninguno ha propuesto un framework para sistematizar el análisis y mejora de TMs** como lo hacemos acá.”

---

## VI. Conclusions and Future Work

### Conclusiones Principales

> “Mostramos cómo se pueden recuperar distribuciones sobre clases de significado desde el comportamiento estocástico de TMs, y cómo eso permite detectar errores y guiar mejoras.”

Este es el **claim central** del paper:  
El comportamiento probabilístico de un TM **no es ruido**, sino **una fuente rica de información** para:
- Evaluar confiabilidad
- Detectar errores peligrosos (concentrados y desalineados)
- Guiar mejorar especificas y seguras.

---
### Casos relevantes: misalignment + concentración

Los autores enfatizan que **los casos más críticos** son aquellos donde:

- El TM está seguro (concentración),
- Pero equivocado (desalineación).

Estos casos son difíciles de detectar con técnicas tradicionales (como entropy o heurísticas), y por eso su análisis de distribuciones semánticas es valioso.

---

### Reconocen que aún faltan validaciones

Este es un trabajo preliminar, y los autores son honestos sobre las **limitaciones**:

- No demuestran aún **generalización** a múltiples tareas.
- No exploran distintas configuraciones de decodificación.
- No prueban la **robustez del framework** ante otros métodos de aproximación semántica.

Plantean que se necesitan muchos más RQs (_research questions_) y experimentos para afirmar impacto general.

---
### Preguntas abiertas relevantes

Los autores proponen preguntas muy concretas para futuras investigaciones:

1. **¿Cómo influye la naturaleza de la tarea** en la efectividad del análisis?
2. **¿Cuán importante es conocer la distribución real de inputs** para que el framework generalice?
3. **¿Mejorar sobre el set de prueba implica mejorar globalmente?**
4. **¿Qué tan sensible es el análisis a la estrategia de decodificación** (top-k, temperature, etc.)?
5. **¿Qué tan robusto es el método de agrupamiento semántico?**

---
### Posibles lineas futuras 

El paper sugiere varios caminos interesantes:

- Usar LLMs para **automatizar la caracterización de errores** (e.g., clasificar outputs como “postcondición débil”).
- Crear **lenguajes más expresivos** para razonar sobre inputs, outputs y distribuciones.
- Estudiar **composicionalidad** entre TMs: ¿podemos componer mejoras modulares?
- Aplicar el enfoque a tareas con **espacios semánticos más estructurados** (e.g., preferencias, rankings).
- Analizar **trade-offs entre entropía y alineación**.

---
### A. The Case of Agentic AI

> “El framework también puede aplicarse al análisis de sistemas agenticos (goal-pursuing) habilitados por LLMs.”

#### ¿Qué significa esto?

- Sistemas agenticos son aquellos que **toman decisiones en múltiples pasos** (piensan, actúan, interactúan con el entorno).
- Estos sistemas pueden modelarse como **procesos de decisión tipo MDP** (Markov Decision Process).
- El LLM implementa los _TMs_ que, dada la historia de interacciones, generan **acciones o pensamientos** (meaning classes).
- Analizando las distribuciones de estos TMs, se pueden identificar **trayectorias peligrosas** antes de que ocurran, como:
    - Invocar acciones sin autorización humana.
    - Violar restricciones del entorno de forma competente.
    - Parecer razonable mientras produce resultados inseguros.

---


## Cierre Conceptual

Este paper propone una **forma principiada, formal y probabilística** de pensar en cómo mejorar sistemas basados en LLM. En vez de tratar los errores como accidentes, los analiza como **pistas estructurales** de lo que un modelo sabe, no sabe o cree saber.

Y abre la puerta a una futura **ingeniería de sistemas LLM**, donde:

- Se mide no solo “qué dice el modelo”, sino “qué probabilidad le asigna a cada interpretación”.
- Se puede mejorar un sistema con una noción clara de progreso, sin romper lo que ya funciona.
- Se combinan conceptos de verificación formal, aprendizaje estadístico y análisis semántico.

