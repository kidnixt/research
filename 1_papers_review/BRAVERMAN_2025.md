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

Autoformalización es el proceso de **convertir texton**