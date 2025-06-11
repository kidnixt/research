# Towards a Probabilistic Framework for Analyzing and Improving LLM-Enabled Software

## Abstract

- Los sistemas basados en LLMs son potentes pero **no deterministas.** Es difícil confiar plenamente en lo que generan o verificar formalmente su comportamiento.
- La **idea central** del paper es: no mirar los *outputs* del LLM como cadenas de texto, sino como ***clusters of semantically equivalent outputs***. 
	- Por ejemplo: todas las respuestas que formalizan una misma propiedad en un código.
- ***"Transference Models (TMs):*** concepto definido por los autores, son **LLM center modules** que reciben una entrada (ej. una descripción) y generan una salida ()