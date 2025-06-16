# Qwen 1.7B (WIP)
## Consideraciones para adaptar de GPT-2 a Qwen3:

1. Qwen usa **tokenización diferente** (por defecto `Tokenizer` de tipo BPE o tiktoken-like).
2. Algunos tokens (como `bos_token`, `eos_token`) pueden estar ausentes o distintos.
3. `Qwen` puede usar `ChatTemplate`, por lo que **evitaremos eso** y trabajaremos con entradas crudas (`trust_remote_code=True` y `use_fast=False` para evitar plantillas de conversación).
4. El modelo requiere ser cargado con `return_dict=True` y suele tener `pad_token_id` como `eos_token_id`.


 bos_token (`str`, *optional*):
            The beginning of sequence token. Not applicable for this tokenizer.

https://github.com/huggingface/transformers/blob/main/src/transformers/models/qwen2/tokenization_qwen2.py


### 🧠 ¿Qué tokenizer usa Qwen3-1.7B?

**Tokenizer:** `Qwen2Tokenizer`

- Es una versión modificada del `tiktoken` de OpenAI.
    
- **No es BPE clásico ni WordPiece**, como en GPT-2 o BERT.
    
- Usa un **tokenizer basado en byte-level merges**, muy eficiente para multilenguaje y codificaciones como números, puntuación, etc.

![[Pasted image 20250616105630.png]]

https://huggingface.co/Qwen/Qwen3-1.7B/blob/main/tokenizer_config.json

Tambien se puede ver si printeas el type del tokenizer