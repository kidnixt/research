## 1. Presentación del flujo de trabajo

El trabajo se organiza como un análisis exploratorio que combina información estructural y de secuencia para estudiar el efecto de mutaciones puntuales en proteínas ya caracterizadas experimentalmente. El punto de partida es una proteína de referencia (forma salvaje, o _wild type_) para la cual existen variantes mutantes generadas y estudiadas en el laboratorio. Estas mutaciones no son hipotéticas ni propuestas por el modelo, sino casos reales que sirven como ancla experimental.

Dado que muchos modelos actuales requieren o se benefician del uso de información estructural, se utiliza una estructura tridimensional de la proteína de referencia obtenida mediante AlphaFold. Esta estructura se emplea como una representación plausible del estado plegado de la proteína, y se mantiene fija como contexto al evaluar distintas mutaciones puntuales. Es importante remarcar que esta aproximación **no busca modelar explícitamente los cambios estructurales inducidos por cada mutación**, sino evaluar la compatibilidad de una mutación con un entorno estructural dado.

Sobre esta base, se introducen mutaciones puntuales específicas y se obtienen scores producidos por el modelo, los cuales pretenden capturar el efecto relativo de cada mutación en el contexto de la secuencia y la estructura proporcionadas. Estos scores se analizan de manera comparativa, es decir, observando diferencias entre mutaciones dentro del mismo sistema y bajo las mismas condiciones de entrada, evitando interpretaciones absolutas o universales.

El objetivo central de este enfoque **no es validar experimentalmente las predicciones del modelo**, ni afirmar que un determinado score corresponde directamente a una propiedad biofísica medible, como estabilidad térmica o actividad catalítica. En esta etapa, el trabajo se limita deliberadamente a evaluar si el modelo es capaz de reproducir **tendencias relativas** entre mutaciones que ya fueron observadas en la práctica experimental, y a identificar posibles desacoples entre lo que el modelo considera plausible y lo que ocurre en el laboratorio.

Solo una vez entendidas estas correspondencias —y, especialmente, sus límites— puede pensarse en utilizar este tipo de herramientas como apoyo exploratorio para priorizar o descartar mutaciones candidatas. Incluso en ese escenario, cualquier predicción debe interpretarse como una hipótesis preliminar que requiere validación experimental independiente. El foco del trabajo está, por lo tanto, en la **interpretación crítica de los scores** y en la comprensión de qué tipo de información están codificando, más que en la optimización directa de proteínas.

---

## 2. ¿Qué está midiendo realmente el score?

El score que producen este tipo de modelos **no es una medición directa de una propiedad física o bioquímica**, ni una simulación explícita del comportamiento de la proteína en condiciones experimentales. En particular, el score **no mide directamente** estabilidad termodinámica, energía libre de plegamiento, actividad catalítica, cinética enzimática ni resistencia a condiciones fisicoquímicas como temperatura o pH.

A nivel conceptual, el score refleja qué tan **compatible o plausible** resulta una mutación dentro del espacio de secuencias y estructuras que el modelo ha aprendido durante su entrenamiento. El modelo ha sido expuesto a grandes conjuntos de proteínas naturales y ha internalizado patrones estadísticos que relacionan secuencias, entornos estructurales y tipos de residuos. Cuando se evalúa una mutación, el score cuantifica hasta qué punto ese cambio encaja o no con esos patrones aprendidos.

En ese sentido, el score puede interpretarse como una medida de **consistencia estadística**: mutaciones que preservan configuraciones frecuentes o esperadas en proteínas reales tienden a recibir mejores scores, mientras que mutaciones que introducen combinaciones raras, atípicas o contradictorias con el contexto estructural suelen ser penalizadas. Esto incluye factores como el tipo de aminoácido en una posición determinada, el entorno local en la estructura, y señales implícitas de conservación evolutiva.

Lo que el score **sí captura**, de manera indirecta, es información relacionada con:

- la compatibilidad de un residuo con un entorno estructural específico,
- la frecuencia con la que mutaciones similares aparecen en proteínas naturales,
- patrones evolutivos aprendidos a partir de grandes bases de datos.

Sin embargo, el score **no distingue explícitamente** entre distintos mecanismos físicos. Por ejemplo, no separa si una penalización se debe a un problema de plegamiento global, a una perturbación local del sitio activo, o a un efecto funcional más sutil. Todos estos factores quedan colapsados en una única métrica estadística.

Por esta razón, un score desfavorable **no implica necesariamente** que una proteína no se pliegue, que sea inestable o que pierda completamente su función. Del mismo modo, un score favorable no garantiza que una mutación mejore ninguna propiedad experimentalmente medible. El valor del score reside en su uso **comparativo y contextual**, no en su interpretación como una predicción directa de comportamiento físico.

En resumen, el score debe entenderse como una señal informativa sobre cómo una mutación se posiciona dentro del espacio de proteínas que el modelo considera “naturales” o “plausibles”. Su utilidad depende de una interpretación cuidadosa, de comparaciones controladas y, en última instancia, de su contraste con datos experimentales reales. El modelo no reemplaza al experimento: como mucho, ayuda a formular preguntas más informadas.

