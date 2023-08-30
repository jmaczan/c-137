<p align="center"><img src="./assets/hi-thanks-for-reading-my-filename.png" alt="hi-thanks-for-reading-my-filename.png" style="height: 200px; width:200px;"/></p>

<p align="center"><code>Llama 2 7B fine-tuned to revive Rick</code></p>

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

I find [this neat colab](https://colab.research.google.com/drive/12dVqXZMIVxGI0uutU6HG9RWbWPXL3vts) and decide to go with fine tuning their way. They're using AlexanderDoria/novel17_test dataset, which when you inspect it, it's a [JSONL file](https://hackernoon.com/json-lines-format-76353b4e588d) (it wraps all content with an array and so you can keep all data in one line - saves disk space). Sample object has this structure

```
{"text":"### Human: Some human text going here### Assistant: Assistant's response here"},{"text":"... and so on
```

[Dataset I'm going to use](https://www.kaggle.com/datasets/andradaolteanu/rickmorty-scripts) is a csv with the following columns

```
index	season no.	episode no.	episode name	name	line
```

We can skip all columns except name and line, unless we want to do some fancy stuff like make Rick recognize in which episode he said a given line

My initial approach to data processing will be to iterate through the file and

GLOBAL INITIALIZATION

- once - at the beginning of processing - create a prompt template `{"text":"### Human: {other_lines} ### Assistant: {rick_lines}"}`
- create a temp empty strings `other_lines` and `rick_lines`
- we will have two pointers (one local and one global)
- global pointer initalized to -1
- global output initialized to empty string
- calculate total number of lines and store it in `total_lines_count` variable

NEXT GLOBAL LINE

- global pointer += 1
- if global pointer is equal to total_lines_count, stop the script and save global output to data.jsonl file
- local pointer initalized to 0
- set empty strings to `other_lines` and `rick_lines`

NEXT LOCAL LINE

- get row of index = global + local
- if current line is not Rick's line and `rick_lines` is not empty:
  - add template filled with `other_lines` and `rick_lines` to global output and append `', ` at the end of the prompt template
  - go to NEXT GLOBAL LINE
- preprocess row's line content and store it in local variable `preprocessed_line`
  - remove all double quotes from lines
- if current line is not Rick's line, then append it to `other_lines += preprocessed_line + ' '`
- if current line is Rick's line, then append it to `rick_lines` += preprocessed_line + ' '`
- local pointer += 1
- go to NEXT LOCAL LINE

We can also think of

- remove first row because it doesn't have other person text before Rick's line
- create a bunch of questions and answers to learn Rick who is he, who are people around him etc
- if we wanted to clear Rick's memory, we could remove all names from the dataset and replace them with "you" or something
- combining all names and lines before Rick's line into one prompt, like

```
{"text":"### Human: Jerry said 'Damn it!'. Beth said 'Jerry!'. Jerry said 'Beth!'. Summer said 'Oh my god, my parents are so loud, I want to die.' ### Assistant: Mm, there is no God, Summer. You gotta rip that band-aid off now. You'll thank me later."}
```

Not sure what kind of combining few lines into Human prompt would be the most effective one, so I guess it needs some experiments to be done

Let's start implementing script for preparing training data

First, create venv

```
python3 -m venv venv
```

I have Python installed as `python3`, you may need to replace it with `python` or whatever you named it. Same goes for `pip3 -> pip`

Activate it

```
source venv/bin/activate
```

I use pandas to manipulate csv data

```
pip install pandas
```

Since [dataset is on Kaggle](https://www.kaggle.com/datasets/andradaolteanu/rickmorty-scripts?resource=download) (you need to sign up first), let's install kaggle as well

```
pip install kaggle
```

Go to [settings in Kaggle web app](https://www.kaggle.com/settings) and click Create New Token. It generates and downloads an API token, which you need to copy into `~/.kaggle/kaggle.json` (you need to `mkdir ~/.kaggle` first)

Set correct rights on the file

```
chmod 600 ~/.kaggle/kaggle.json
```

Let's install langchain to help with some stuff, for now I just going to use it for prompt formatting

```
pip install langchain
```

Ok, the code to prepare data is done. [You can check it here](./src/prepare_training_data.py)

I pushed [the dataset on Hugging Face](https://huggingface.co/datasets/jmaczan/rick-and-morty-scripts-llama-2), so you can reuse it

I take [this great Google Colab](https://colab.research.google.com/drive/12dVqXZMIVxGI0uutU6HG9RWbWPXL3vts), adjust it a little to use my dataset and run it

And that's it actually. The finished model is [available here on Hugging Face](https://huggingface.co/jmaczan/llama-2-7b-qlora-rick-sanchez-c-137)

The model seems to be crappy. I'm not sure what I did wrong. I think the prompt format could be bad, actually. I learn that it's likely Vicuna 1 format

Time for second round of training

I do another [training based on this article](https://towardsdatascience.com/fine-tune-your-own-llama-2-model-in-a-colab-notebook-df9823a04a32)

In order to make it work, I need to adjust the data. [You can see a script here](./src/prepare_training_data_as_csv.py)

It produces dataset.csv, which needs some post processing, because it appends some excessive empty columns

[I publish the correct dataset on Hugging Face](https://huggingface.co/datasets/jmaczan/rick-and-morty-scripts-llama-2) and [update the old one to have proper naming](https://huggingface.co/datasets/jmaczan/rick-and-morty-scripts-vicuna-1)

Anyway, here it is! [Google Colab with fine tuning by PEFT, using Torch, Transformers, trl for SFT and QLoRA on 4 bits](https://colab.research.google.com/drive/1berHyEuuPtMaYMYD1EYK3J7dv3Qzv7cg). The foundation model is `meta-llama/Llama-2-7b-chat-hf`

To see how it performs, [you can click here to scroll to the example](https://colab.research.google.com/drive/1berHyEuuPtMaYMYD1EYK3J7dv3Qzv7cg#scrollTo=_zG_-Ew64bbF)

The model is [hosted on Hugging Face](https://huggingface.co/jmaczan/llama-2-7b-rick-c-137) as well

🤗 Thanks for reading

### Sources

https://replicate.com/blog/run-llama-locally

https://github.com/ggerganov/llama.cpp

https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGML

https://towardsdatascience.com/fine-tune-your-own-llama-2-model-in-a-colab-notebook-df9823a04a32

https://huggingface.co/datasets/Prarabdha/Rick_and_Morty_Transcript

https://www.kaggle.com/datasets/andradaolteanu/rickmorty-scripts

https://gist.github.com/younesbelkada/9f7f75c94bdc1981c8ca5cc937d4a4da

https://github.com/artidoro/qlora

https://colab.research.google.com/drive/12dVqXZMIVxGI0uutU6HG9RWbWPXL3vts

https://huggingface.co/datasets/jmaczan/rick-and-morty-scripts-llama-2

https://huggingface.co/jmaczan/llama-2-7b-qlora-rick-sanchez-c-137

Inspiration: https://towardsdatascience.com/make-your-own-rick-sanchez-bot-with-transformers-and-dialogpt-fine-tuning-f85e6d1f4e30

###### Author: [Jędrzej Paweł Maczan](https://maczan.pl)
