# Qwen 1.7B (WIP)
## Consideraciones para adaptar de GPT-2 a Qwen3:

1. Qwen usa **tokenización diferente** (por defecto `Tokenizer` de tipo BPE o tiktoken-like).
2. Algunos tokens (como `bos_token`, `eos_token`) pueden estar ausentes o distintos.
3. `Qwen` puede usar `ChatTemplate`, por lo que **evitaremos eso** y trabajaremos con entradas crudas (`trust_remote_code=True` y `use_fast=False` para evitar plantillas de conversación).
4. El modelo requiere ser cargado con `return_dict=True` y suele tener `pad_token_id` como `eos_token_id`.