<p align="center"><img src="hi-thanks-for-reading-my-filename.png" alt="hi-thanks-for-reading-my-filename.png" style="height: 200px; width:200px;"/></p>

<p align="center"><code>Llama 2 7B on Apple M2 fine-tuned to revive Rick</code></p>

### Dear diary

Get [llama.cpp](https://github.com/ggerganov/llama.cpp)

```
git clone https://github.com/ggerganov/llama.cpp.git
```

[Build for M1/M2](https://replicate.com/blog/run-llama-locally)

```
cd llama.cpp
LLAMA_METAL=1 make
cd ..
```

Get [Llama 2 7B](https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML)

```
mkdir models
curl -L "https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML/resolve/main/llama-2-7b-chat.ggmlv3.q3_K_M.bin" -o models/llama-2-7b-chat.ggmlv3.q3_K_M.bin
```

I used `llama-2-7b-chat.ggmlv3.q3_K_M.bin` - Llama 2 7B quantized to 3 bits of size 3.28 GB, with 5.78 GB max RAM required

. According to model card on HF:

> New k-quant method. Uses GGML_TYPE_Q4_K for the attention.wv, attention.wo, and feed_forward.w2 tensors, else GGML_TYPE_Q3_K

Run it

```
./llama.cpp/main -m ./models/llama-2-7b-chat.ggmlv3.q3_K_M.bin \
  --color \
  --ctx_size 2048 \
  -n -1 \
  -ins -b 256 \
  --top_k 10000 \
  --temp 0.2 \
  --repeat_penalty 1.1 \
  -t 8
```

#### Sources

https://towardsdatascience.com/make-your-own-rick-sanchez-bot-with-transformers-and-dialogpt-fine-tuning-f85e6d1f4e30

https://replicate.com/blog/run-llama-locally

https://github.com/ggerganov/llama.cpp

https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML

###### Author: [Jędrzej Paweł Maczan](https://maczan.pl)
