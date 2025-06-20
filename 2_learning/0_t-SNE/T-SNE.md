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