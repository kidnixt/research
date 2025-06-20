# t-Distributed Stochastic Neighbor Embedding

## 🧩 Paso 1: ¿Qué es t-SNE y para qué sirve?

### 🔷 1.1 ¿Cuál es el problema?

Cuando trabajamos con datos, muchas veces tienen muchas **dimensiones**.  
Ejemplos:

- Una imagen de 28x28 píxeles (como las de MNIST) tiene **784 dimensiones**.
- Un embedding de una palabra en NLP puede tener **300 dimensiones**.
- Un genoma puede tener miles de características.

📉 El problema es:

> No podemos visualizar ni entender fácilmente datos en 300 o 1000 dimensiones.  
> Por eso usamos **reducción de dimensionalidad**.


### 🔷 1.2 ¿Qué es reducción de dimensionalidad?

Es una técnica para transformar los datos de un **espacio de muchas dimensiones** a uno de **menos dimensiones** (por ejemplo, 2D o 3D), **conservando lo más posible la estructura original**.

- Si lo reducís a 2D, podés **graficarlo**.
- Si la estructura original tenía **clusters**, queremos que se vean igual en 2D.

### 🔷 1.3 ¿Qué es t-SNE entonces?

**t-SNE = t-Distributed Stochastic Neighbor Embedding**  
Es una técnica **no lineal** que:

✅ Intenta mantener puntos **cercanos** juntos en el mapa 2D.  
🚫 No le importa tanto mantener puntos **lejanos separados**.

> En otras palabras: t-SNE **preserva relaciones locales**, no globales.

### 🎯 Aplicaciones típicas

- Ver cómo se agrupan palabras o imágenes.
- Ver si un modelo de clasificación separa bien las clases.
- Exploración visual de datos no etiquetados.


---

## 🔍 Paso 2: ¿Cómo funciona t-SNE? (Intuición)

### 🧠 Idea principal

t-SNE trata de encontrar una representación en **2D (o 3D)** en la que las **distancias entre puntos reflejen las similitudes que tenían en el espacio original (de alta dimensión)**.


### 🔹 2.1 Paso 1: Medir similitudes en el espacio original

- En el espacio original (por ejemplo, 300 dimensiones), t-SNE se pregunta:
    > “¿Qué tan cerca están el punto A y el punto B?”
- Pero en lugar de medir la distancia directamente, convierte eso en una **probabilidad**:
    > “¿Cuál es la probabilidad de que A elija a B como su vecino?”

Esto se hace para todos los pares de puntos, y así se construye una **distribución de probabilidades**.

📌 Esto se llama la distribución en **alta dimensión**.

### 🔹 2.2 Paso 2: Ubicar los puntos en 2D

- Ahora t-SNE pone **todos los puntos al azar en 2D**.
- A partir de esa ubicación en 2D, calcula **nuevas probabilidades de vecindad**.  
    Igual que antes, pero en dos dimensiones.

📌 Esto se llama la distribución en **baja dimensión**.

### 🔹 2.3 Paso 3: Hacer que las dos distribuciones se parezcan

Ahora viene la parte clave:

👉 t-SNE compara la distribución en alta dimensión y la de baja dimensión, y **mueve los puntos en 2D** para que la distribución de similitudes se parezca lo más posible a la original.

Esto lo hace **iterativamente**, como un algoritmo de optimización.  
(Usa algo parecido al gradiente descendente.)

### 💡 ¿Por qué se llama “t-Distributed”?

En el espacio 2D, la forma en que mide la similitud usa una **distribución t de Student** (en lugar de una gaussiana como en alta dimensión).  
Esto ayuda a evitar que puntos lejanos se amontonen y mejora la visualización.

--- 

## ⚖️ Paso 3: Ventajas y desventajas de t-SNE

### 🟢 Ventajas

#### 1. Muy buenas visualizaciones

- Si tenés datos complejos en muchas dimensiones, t-SNE suele generar **representaciones 2D muy intuitivas**.
- Muestra estructuras que otros métodos (como PCA) no detectan.

#### 2. Descubre relaciones locales

- Conserva la **proximidad de puntos similares**.
- Ideal para descubrir _clusters_ o agrupaciones que no conocías.

#### 3. No requiere etiquetas

- Es un método **no supervisado**.
- Podés usarlo con datos sin clasificar para explorar su estructura.

#### 4. Flexible

- Funciona bien con imágenes, texto (embeddings), genómica, datos financieros, etc.

---

### 🔴 Desventajas

#### 1. No es interpretable

- El eje 1 y el eje 2 **no tienen significado fijo**.
- No podés decir: “si sube en X, pasa tal cosa”.  
    ➤ Solo representa relaciones relativas.

#### 2. No escala bien

- Computacionalmente **pesado** en datasets grandes (>10,000 puntos).
- Hay variantes como **Barnes-Hut t-SNE** o **FIt-SNE** para escalarlo mejor.

#### 3. Resultados no repetibles

- Si no fijás la semilla (`random_state`), puede dar visualizaciones distintas cada vez.
- Incluso con misma semilla, **hay cierta variabilidad**.

#### 4. No conserva distancias globales

- t-SNE no trata de mantener la forma total del espacio.
- Por eso, dos clusters lejos en el gráfico podrían no estar lejos en la realidad.

#### 5. Sensibilidad a hiperparámetros

- Especialmente al `perplexity` (controla el “tamaño del vecindario”).
- Cambiarlo cambia mucho el resultado visual.

---

### ❗Errores comunes

1. **Usar t-SNE para clasificación**  
    ✖️ No es un clasificador. Solo es útil para ver patrones.
2. **Interpretar distancias grandes como significativas**  
    ✖️ En t-SNE, **le importa lo local**, no lo global.
3. **No escalar los datos antes de aplicarlo**  
    ✖️ t-SNE asume que todas las dimensiones tienen magnitudes comparables.
4. **Comparar visualizaciones de distintos datasets directamente**  
    ✖️ Como es una técnica probabilística, no podés comparar visualizaciones de forma directa (como sí podrías con PCA).