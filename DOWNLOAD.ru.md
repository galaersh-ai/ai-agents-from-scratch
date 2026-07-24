Скачайте модели, используемые в этом репозитории

Вы можете настроить уровень квантизации для баланса между точностью модели и размером файла:
Используйте `:Q8_0` для более высокой точности и лучшего качества вывода, но обратите внимание, что это требует больше памяти и дискового пространства.
Используйте `:Q6_K` для хорошего баланса между размером и точностью (рекомендуемый по умолчанию).
Используйте `:Q5_K_S` для меньшей модели, которая загружается быстрее и потребляет меньше памяти, но с קצת меньшей точностью.

```
npx --no node-llama-cpp pull --dir ./models hf:Qwen/Qwen3-1.7B-GGUF:Q8_0 --filename Qwen3-1.7B-Q8_0.gguf
```

```
npx --no node-llama-cpp pull --dir ./models hf:giladgd/gpt-oss-20b-GGUF/gpt-oss-20b.MXFP4.gguf
```

```
npx --no node-llama-cpp pull --dir ./models hf:unsloth/DeepSeek-R1-0528-Qwen3-8B-GGUF:Q6_K --filename DeepSeek-R1-0528-Qwen3-8B-Q6_K.gguf
```

```
npx --no node-llama-cpp pull --dir ./models hf:giladgd/Apertus-8B-Instruct-2509-GGUF:Q6_K
```

Пример **15 (маршрутизация инструментов)** также требует маленького **embedding-only** GGUF (~37MB для Q8_0). Используйте `--filename`, чтобы путь совпадал с кодом:

```
npx --no node-llama-cpp pull --dir ./models hf:CompendiumLabs/bge-small-en-v1.5-gguf:bge-small-en-v1.5-q8_0.gguf --filename bge-small-en-v1.5-q8_0.gguf
```

Загрузка chat-модели и этой embedding-модели одновременно потребует больше RAM, чем пример с одной моделью; 8GB+ системной RAM всё ещё обычно достаточно для Qwen3-1.7B плюс bge-small.
