<p align="center"><img src="./assets/hi-thanks-for-reading-my-filename.png" alt="hi-thanks-for-reading-my-filename.png" style="height: 200px; width:200px;"/></p>

<p align="center"><code>Llama 2 7B on Apple M2 fine-tuned to revive Rick</code></p>

### Dear diary

Get [llama.cpp](https://github.com/ggerganov/llama.cpp) (all commands are ran in [terminal](https://en.wikipedia.org/wiki/Terminal_emulator)) (you need [git](https://git-scm.com/))

```
git clone https://github.com/ggerganov/llama.cpp.git
```

[Build for M1/M2](https://replicate.com/blog/run-llama-locally) (you need [make](https://www.gnu.org/software/make/))

```
cd llama.cpp
LLAMA_METAL=1 make
cd ..
```

Get [Llama 2 7B](https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML) (you need [curl](https://curl.se/))

```
mkdir models
curl -L "https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML/resolve/main/llama-2-7b-chat.ggmlv3.q3_K_M.bin" -o models/llama-2-7b-chat.ggmlv3.q3_K_M.bin
```

I used `llama-2-7b-chat.ggmlv3.q3_K_M.bin` - Llama 2 7B quantized to 3 bits of size 3.28 GB, with 5.78 GB max RAM required

According to model card on Hugging Face

> New k-quant method. Uses GGML_TYPE_Q4_K for the attention.wv, attention.wo, and feed_forward.w2 tensors, else GGML_TYPE_Q3_K

Wise thing probably

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

Now when I prompt the model with "Pretend you're Rick Sanchez from Rick and Morty cartoon.", it responds with a really good response

> Woah, dude! _adjusts sunglasses_ What's up? It's me, Rick Sanchez, the coolest, most brilliant scientist in the multiverse! _takes a hit from a bong_ You know, I've been working on some pretty wild stuff lately. Like, have you heard of my latest invention? _coughs_ It's a device that can manipulate reality itself! _adjusts sunglasses_ With this baby, I can make anything happen! _laughs maniacally_ But enough about that. What's your problem, dude? Need some help with something? _cracks knuckles_ Just give me a shout and I'll be there in a jiffy! _adjusts sunglasses again_ Don't worry, man, I'm here to save the day... or at least make it more interesting. _winks_

It sounds a bit like Rick, right? but I hope we can make it better

Let's [figure out](https://gist.github.com/younesbelkada/9f7f75c94bdc1981c8ca5cc937d4a4da) how to [fine-tune it](https://towardsdatascience.com/fine-tune-your-own-llama-2-model-in-a-colab-notebook-df9823a04a32)

I find [this dataset with Rick and Morty transcript](https://huggingface.co/datasets/Prarabdha/Rick_and_Morty_Transcript), which we may use for fine-tuning

Btw clone this repository, if you haven't do this yet

```
git clone https://github.com/jmaczan/c-137.git
```

Ok, so I'm going to use [QLoRA](https://github.com/artidoro/qlora), which is an efficient way of fine-tuning by quantizing weights (shortening weights to n-bits). Don't expect me to provide more details, I'm still learning as well!

Wait a second. [A quick look on mlabonne/guanaco-llama2-1k dataset](https://colab.research.google.com/drive/1Ad7a9zMmkxuXTOh1Z7-rNSICA4dybpM2?usp=sharing) makes me think whether the transcript needs such pre-processing as well? So all segments likely need to be in a format

```
<s>[INST] {human_text} [/INST] {assistant_text} </s>
```

or

```
<s>[INST] {human_text} [/INST] </s>
```

What is human text and assistant text in case of the transcript? That's some decision to make. [We have info about speaker and their line](https://huggingface.co/datasets/Prarabdha/Rick_and_Morty_Transcript/viewer/Prarabdha--Rick_and_Morty_Transcript/train), so maybe we can loop through all rows and merge all non-Rick dialogues to human_text and once Rick is speaking, put his line to assistant_text and repeat it until we reach end of the transcript. But I dislike these descriptions of actions which are in dialogues and they are indistinguishable from the words a character says. I find [another dataset with transcription on Kaggle](https://www.kaggle.com/datasets/andradaolteanu/rickmorty-scripts) which seems to not have these actions descriptions

#### Sources

https://replicate.com/blog/run-llama-locally

https://github.com/ggerganov/llama.cpp

https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML

https://towardsdatascience.com/fine-tune-your-own-llama-2-model-in-a-colab-notebook-df9823a04a32

https://huggingface.co/datasets/Prarabdha/Rick_and_Morty_Transcript

https://www.kaggle.com/datasets/andradaolteanu/rickmorty-scripts

https://gist.github.com/younesbelkada/9f7f75c94bdc1981c8ca5cc937d4a4da

https://github.com/artidoro/qlora

Inspiration: https://towardsdatascience.com/make-your-own-rick-sanchez-bot-with-transformers-and-dialogpt-fine-tuning-f85e6d1f4e30

###### Author: [Jędrzej Paweł Maczan](https://maczan.pl)
