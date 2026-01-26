## 1. Presentación del flujo de trabajo

El trabajo se organiza como un análisis exploratorio que combina información estructural y de secuencia para estudiar el efecto de mutaciones puntuales en proteínas ya caracterizadas experimentalmente. El punto de partida es una proteína de referencia (forma salvaje, o _wild type_) para la cual existen variantes mutantes generadas y estudiadas en el laboratorio. Estas mutaciones no son hipotéticas ni propuestas por el modelo, sino casos reales que sirven como ancla experimental.

Dado que muchos modelos actuales requieren o se benefician del uso de información estructural, se utiliza una estructura tridimensional de la proteína de referencia obtenida mediante AlphaFold. Esta estructura se emplea como una representación plausible del estado plegado de la proteína, y se mantiene fija como contexto al evaluar distintas mutaciones puntuales. Es importante remarcar que esta aproximación **no busca modelar explícitamente los cambios estructurales inducidos por cada mutación**, sino evaluar la compatibilidad de una mutación con un entorno estructural dado.

Sobre esta base, se introducen mutaciones puntuales específicas y se obtienen scores producidos por el modelo, los cuales pretenden capturar el efecto relativo de cada mutación en el contexto de la secuencia y la estructura proporcionadas. Estos scores se analizan de manera comparativa, es decir, observando diferencias entre mutaciones dentro del mismo sistema y bajo las mismas condiciones de entrada, evitando interpretaciones absolutas o universales.

El objetivo central de este enfoque **no es validar experimentalmente las predicciones del modelo**, ni afirmar que un determinado score corresponde directamente a una propiedad biofísica medible, como estabilidad térmica o actividad catalítica. En esta etapa, el trabajo se limita deliberadamente a evaluar si el modelo es capaz de reproducir **tendencias relativas** entre mutaciones que ya fueron observadas en la práctica experimental, y a identificar posibles desacoples entre lo que el modelo considera plausible y lo que ocurre en el laboratorio.

Solo una vez entendidas estas correspondencias —y, especialmente, sus límites— puede pensarse en utilizar este tipo de herramientas como apoyo exploratorio para priorizar o descartar mutaciones candidatas. Incluso en ese escenario, cualquier predicción debe interpretarse como una hipótesis preliminar que requiere validación experimental independiente. El foco del trabajo está, por lo tanto, en la **interpretación crítica de los scores** y en la comprensión de qué tipo de información están codificando, más que en la optimización directa de proteínas.

---

## 2. ¿Qué está midiendo realmente el score?

SaProt define el efecto de una mutación mediante un **log odds ratio**, comparando la probabilidad asignada por el modelo a la secuencia mutada frente a la secuencia _wild-type_, **desde una perspectiva evolutiva**, no experimental.

Tal como indica explícitamente la documentación oficial de SaProt:

> _“A positive score means the mutation is better than the wild type from an evolution perspective (the larger the better).”_

Esto implica que el score cuantifica **cuán compatible es una mutación con los patrones estadísticos aprendidos por el modelo**, que reflejan **presiones evolutivas implícitas en los datos de entrenamiento**, y **no** propiedades físicas directas como:

- estabilidad estructural real,
- energía libre de plegamiento,
- actividad catalítica,
- afinidad de unión,
- o fitness biológico medido experimentalmente.

En otras palabras, el score responde a la pregunta:

> _“¿Qué tan plausible le resulta esta mutación al modelo, dado el contexto secuencial y estructural, en comparación con el residuo original?”_


---

### Formulación del score: log odds ratio

SaProt adopta como base la formulación propuesta por **Meier et al.**, donde el efecto mutacional se expresa como una suma de diferencias de log-probabilidades entre mutante y _wild-type_ en las posiciones mutadas.

La idea central es sencilla pero potente:  
si el modelo asigna **mayor probabilidad** al residuo mutado que al original, **en el mismo contexto**, entonces la mutación es considerada “mejor” **desde la perspectiva del modelo**.

Sin embargo, SaProt introduce una modificación clave respecto a la formulación original, debido a que **su vocabulario no es solo secuencial**, sino **estructura-aware**.

Esto nos lleva a un punto crítico de interpretación.

---

### ¿Por qué el score de SaProt no es simplemente “log-likelihood”?

Aunque el score se construye a partir de log-probabilidades, **no debe interpretarse como una log-likelihood global de la proteína**, ni como una energía, ni como una probabilidad normalizada de que la proteína exista.

En particular:

- El score **solo compara mutante vs wild-type**, no evalúa la proteína en aislamiento.
- El contexto se mantiene fijo ($x_{\setminus T}$), lo que refuerza que es una **comparación local condicionada**.
- En SaProt, la probabilidad de un residuo **se obtiene sumando probabilidades de múltiples tokens estructurales**, no de un único token.



---

### Qué **NO** estamos afirmando con este score

Para evitar interpretaciones incorrectas, dejamos explícitamente establecido que **en este trabajo**:

- ❌ No afirmamos que una mutación con score positivo sea más estable termodinámicamente
- ❌ No afirmamos mejora funcional
- ❌ No afirmamos ventaja evolutiva real en un organismo
- ❌ No validamos el score contra datos experimentales

Lo único que afirmamos es:
> **El score de SaProt refleja una preferencia estadística del modelo, aprendida a partir de datos evolutivos y estructurales, condicionada al contexto de la proteína.**

---
### Por qué aun así el score es útil

A pesar de estas limitaciones, este tipo de score es extremadamente útil como:

- **herramienta exploratoria**,
- **prior computacional**,
- **filtro inicial de mutaciones**,
- o señal comparativa en estudios _in silico_.

Especialmente en escenarios de **zero-shot prediction**, el score permite identificar mutaciones que el modelo considera más compatibles con el espacio de proteínas “naturales” que ha aprendido.