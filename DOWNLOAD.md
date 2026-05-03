Téléchargez les models utilisés dans ce dépôt

Vous pouvez ajuster le niveau de quantization pour équilibrer précision du model et taille du fichier :
Utilisez `:Q8_0` pour une précision plus élevée et une meilleure qualité de sortie, mais notez qu'il nécessite plus de mémoire et de stockage.
Utilisez `:Q6_K` pour un bon compromis entre taille et précision (recommandé par défaut).
Utilisez `:Q5_K_S` pour un model plus petit qui se charge plus rapidement et utilise moins de mémoire, mais avec une précision légèrement inférieure.

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


