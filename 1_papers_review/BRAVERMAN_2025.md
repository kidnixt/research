# Towards a Probabilistic Framework for Analyzing and Improving LLM-Enabled Software

## Abstract

- Los sistemas basados en LLMs son potentes pero **no deterministas.** Es difícil confiar plenamente en lo que generan o verificar formalmente su comportamiento.
- La **idea central** del paper es: no mirar los *outputs* del LLM como cadenas de texto, sino como ***clusters of semantically equivalent outputs***. 
	- Por ejemplo: todas las respuestas que formalizan una misma propiedad en un código.
- ***"Transference Models (TMs):*** concepto definido por los autores, son **LLM center modules** que reciben una entrada (ej. una descripción) y generan una salida (ej. especificación formal). Analizar como se comportan los TMs bajo el LLM es la base del trabajo.
- La **aplicación concreta** elegida en el paper es, dado un *docstring*, el modelo debe generar **precondiciones** y **postcondiciones** formales en Dafny.
- Al mirar no solo qué responde el modelo, sino cómo reparte la probabilidad entre distintas interpretaciones, se pueden **identificar patrones de error sistemáticos**, por ejemplo, cuando el modelo siempre genera una postcondición débil. 

> Los sistemas que usan LLMs son díficiles de verificar y depurar. Proponen un enfoque no analiza solo las respuesta, sino la distribución de probabilidad **sobre clases semánticas**. Esto los ayuda a identificar errores sistemáticos, como respuestas incorrectas con alta confianza. Lo ilustran con el problema de transformar lenguaje natural en especificaciones formales (*autoformalización*). 


## Introducción

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

## Preliminares

### Large Language Models

**¿Qué es un LLM desde el punto de vista técnico?**

Desde el punto de vista técnico, un LLM, básicamente es una **función probabilística paramétrica**. Es decir:
- Toma un input $x$ (un texto anterior, también llamado "contexto").
- Y devuelve una distribución de probabilidad sobre posibles próximos tokens $t_k$:
$$ P(t_k | x)$$ Esto significa que el modelo no genera "una respuesta", sino una **distribución sobre todas las respuestas posibles,** token por token.

---

 **¿Cómo se entrena un LLM?**

A través de _next-token prediction_ (predicción del próximo token). Se entrena con **corpus masivos** de texto, ajustando sus parámetros para minimizar un error (usualmente entropía cruzada) entre:

- lo que el modelo predice como próximo token, y
- lo que realmente sigue en el texto de entrenamiento.

⚠️ Esto refuerza la idea de que el modelo aprende **probabilidades de continuación**, no reglas explícitas o verdades lógicas.

---
**¿Cómo se usa un LLM en la práctica?**

Una vez entrenado, el modelo puede usarse para generar texto. Para hacerlo, necesita un **decoding strategy**, como:

- _Greedy_: elige siempre el token más probable.
- _Sampling_: elige aleatoriamente según la distribución.
- _Top-k_ / _Top-p_: limita la elección a los tokens más probables.
- _Temperature_: controla cuán "creativo" es el modelo (más temperatura = más aleatoriedad).

Esto es crucial porque **el mismo input puede generar diferentes salidas** si el decoding es estocástico.

---
**Conexión con el paper**

Los autores enfatizan que si bien muchas aplicaciones usan el LLM como una caja negra, ellos quieren **modelar esta distribución** para entender el comportamiento del sistema. No les interesa solo “qué salida dio”, sino:

> **¿Con qué probabilidad eligió esa salida sobre otras posibles equivalentes?**

Esto los lleva a analizar _distribuciones sobre clases semánticas_, no solo sobre tokens.


### Autoformalization

**¿Qué es la autoformalización?**

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
**¿Por qué es importante?**

Porque automatizar esta transformación ayuda a tareas críticas como:

- **Verificación automática de programas** (probar que el código cumple con lo especificado).
- **Pruebas formales** (testings guiados por propiedades).
- **Documentación ejecutable** (donde los comentarios se vuelven parte verificable del programa).

Esta es una **tarea dura**, tradicionalmente realizada por expertos. Usar LLMs para esto es ambicioso, pero también muy prometedor.

---

**¿Por qué eligieron Dafny?**

- Dafny es un lenguaje diseñado para **verificación automática** de programas, con soporte para precondiciones, postcondiciones, invariantes, etc.
- Tiene un **verificador SMT incorporado**, que permite comprobar si las condiciones formales son lógicamente correctas.
- Es usado frecuentemente en papers académicos para tareas de autoformalización y synthesis.

Además, como el objetivo del paper es evaluar distribuciones sobre significados formales, usar Dafny permite chequear automáticamente **equivalencia semántica** entre dos especificaciones (vía SMT).

---
**¿Qué rol juega esta tarea en el paper?**

Es el **caso de estudio** principal que se usa para aplicar el marco probabilístico. Todo el análisis (alineación, concentración, mejoras al TM) se ilustra a partir de esta tarea. Lo interesante es que:

- La tarea tiene un **ground truth claro** (las especificaciones correctas están dadas).
- Se puede generar muchas salidas (por el LLM) y verificar si son **equivalentes** entre sí.
- Es posible categorizar errores comunes (por ejemplo, _postcondición débil_, _omisión de propiedad clave_, _error de sintaxis_).

![[Pasted image 20250613102054.png]]

### Transference Models (TMs)

**¿Qué es un Transference Model?**

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
**¿Por qué los TMs son estocásticos?**

Porque dependen de un LLM, y los LLMs son modelos **probabilisticos:** Dada una entrada $i$, el LLM puede producir distintas salidos $o$, cada una con una probabilidad. Formalmente:
$$T: \mathcal{I} \times \mathcal{O} \rightarrow \mathbb{R} \quad\text{donde}\quad T(i,o) = P(o|i)$$

O sea, el TM **define una distribución** sobre salidas posibles para cada entrada. Esto depende fuertemente de:
- El LMM usado.
- El prompt.
- La **estrategia de decodificación** (ej. sampling con temperatura, top-k, etc).

---

**¿Qué incluye un TM?**

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

**¿Por qué es tan importante este concepto?**

Porque los autores **NO** están interesados en estudiar al LLM directamente, sino en estudiar **los sistemas construidos con LLMs**. Es decir:

> "El LLM no es el sistema, sino una parte de un módulo más grande que transforma datos para una tarea."

Entonces el **objeto de análisis** del paper no es el modelo GPT o Gemini, sino el **TM** como componente de software. El marco propuesto evalúa cómo el TM se comporta probabilísticamente:
- Genera siempre la salida correcta?
- Qué tanta certeza tiene?
- Hay errores frecuentes con alta probabilidad?

---
**Modelo funcional formal**

Para reforzar: el TM se modela como una función probabilística:

$$T(i, \cdot) = \text{una distribución de probabilidad sobre salidas }o \text{ dado un input } i  $$
En otras palabras: **el TM actúa como una caja negra estocástica** que recibe una entrada y escupe una distribución sobre posibles resultados.

Lo que ellos proponen es: **muestrear esta caja negra** muchas veces con el mismo input, agrupar las salidas en clases de significado equivalentes y analizar esa distribución.