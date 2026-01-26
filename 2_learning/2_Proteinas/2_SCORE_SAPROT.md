# Explicación del score mutacional y sus ecuaciones

## ¿Qué es un residuo?

En el contexto de proteínas, un **residuo** es una **unidad individual de aminoácido dentro de una cadena polipeptídica**.  
Se habla de “residuo” (y no simplemente de “aminoácido”) porque, una vez que un aminoácido se incorpora a la proteína mediante un enlace peptídico, pierde un grupo químico (agua) y pasa a formar parte de una estructura mayor.

Cada residuo está caracterizado por:

- su **identidad química** (Ala, Asp, Gly, etc.), 
- su **posición** en la secuencia,
- y su **entorno estructural** dentro de la proteína plegada.

Una mutación puntual consiste en **reemplazar un residuo por otro en una posición específica** de la secuencia.

---

## Ecuación original del efecto mutacional (Meier et al.)

La formulación original del efecto de una mutación propuesta por Meier et al. se define como:

$$
\sum_{t \in T}
\left[
\log P(x_t = s_t^{mt} \mid x_{\setminus T})
-
\log P(x_t = s_t^{wt} \mid x_{\setminus T})
\right]
$$

---

### Significado de cada término

- $T$: conjunto de posiciones mutadas
- $t$: una posición específica dentro de la secuencia
- $x_t$: token en la posición $t$
- $s_t^{mt}$: residuo mutado en la posición $t$
- $s_t^{wt}$: residuo wild-type en la posición $t$
- $x_{\setminus T}$: el contexto restante, es decir, todos los demás tokens de la secuencia que **no** están mutados

---

### Interpretación conceptual

Esta ecuación compara, **posición por posición**, cuánto más probable es el residuo mutado frente al residuo original, **dado exactamente el mismo contexto**.

El uso del logaritmo transforma la razón de probabilidades en una diferencia aditiva, y la suma sobre ( T ) permite extender el cálculo a mutaciones múltiples.

Conceptualmente, la pregunta que responde esta ecuación es:

> “Dado el resto de la proteína, ¿el modelo considera más probable el residuo mutado o el residuo original en esta posición?”

Esta formulación asume un **vocabulario puramente secuencial**, donde cada posición está asociada a un único token correspondiente a un aminoácido.

---

## Modificación introducida por SaProt

SaProt modifica esta formulación porque **no trabaja con un vocabulario puramente secuencial**.  
En su lugar, utiliza un vocabulario conjunto definido como:

$$
V \times F
$$

donde:

- $V$ : alfabeto de residuos (aminoácidos)    
- $F$ : alfabeto estructural (tokens 3Di generados por Foldseek)

Esto implica que cada token representa **simultáneamente**:

- un residuo,    
- y una descripción discretizada de su entorno estructural local.

---

### Ecuación utilizada por SaProt

La definición del score mutacional en SaProt es:

$$
\sum_{t \in T}
\left[
\log \sum_{f \in \mathcal{F}}
P(x_t = s_t^{mt} f \mid x_{\setminus T})
-
\log \sum_{f \in \mathcal{F}}
P(x_t = s_t^{wt} f \mid x_{\setminus T})
\right]
$$

---

### Qué cambia respecto a la ecuación original

En esta formulación:

- Un residuo **ya no corresponde a un único token**    
- Un mismo residuo puede aparecer asociado a **múltiples tokens estructurales**
- La probabilidad de un residuo se define como la **suma de las probabilidades de todos los tokens estructurales compatibles con ese residuo**

Es decir, SaProt no pregunta por la probabilidad de un token específico, sino por la **probabilidad total de observar ese residuo en cualquier estado estructural plausible**.

---

## Diferencia conceptual clave

- **Ecuación original (Meier et al.)**  
    → _“¿Qué tan probable es este aminoácido en esta posición?”_    
- **Ecuación de SaProt**  
    → _“¿Qué tan probable es este aminoácido en esta posición, considerando todos los entornos estructurales compatibles?”_

La lógica del **log odds ratio** se mantiene intacta: en ambos casos se compara mutante contra wild-type bajo el mismo contexto.  Lo que cambia es **cómo se define la probabilidad del residuo**.

---

## Consecuencias para la interpretación del score

Debido a esta redefinición:

- el score de SaProt es **más robusto frente a variaciones estructurales locales**,    
- pero también es **menos directamente interpretable** como una simple probabilidad de residuo aislado.

El score refleja una **preferencia estadística integrada sobre estructura y secuencia**, y no una probabilidad puntual asociada a un único estado estructural.

---