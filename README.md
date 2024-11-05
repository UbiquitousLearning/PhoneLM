## Data Preparation

### Stable Training Stage

We use the following dataset in stable training stage.
| type     | dataset                         | token  |
|----------|---------------------------------|--------|
| web      | [DCLM-baseline](https://huggingface.co/datasets/mlfoundations/dclm-baseline-1.0-parquet) | 1.35T  |
| code     | [StarCoderData](https://huggingface.co/datasets/bigcode/starcoderdata) | 112.75B|
| math     | [OpenWebMath](https://huggingface.co/datasets/open-web-math/open-web-math) | 13.25B |
| academic | [Dolma-algebraic](https://huggingface.co/datasets/allenai/dolma) | 12.75B |
| academic | [Dolma-arxiv](https://huggingface.co/datasets/allenai/dolma) | 29B   |
| **total**|                                 | **1.5T** |

**Download The Original Data**

You can download the dataset from the links provided in the table above using any method.As an example, we use `huggingface-cli` to download DCLM-baseline. Here is an example command:
```bash
huggingface-cli download --repo-type dataset --local-dir ./dclm-baseline --local-dir-use-symlinks False --resume-download mlfoundations/dclm-baseline-1.0-parquet
```
You can decide how to download the dataset through the links in the table above.

**Preprocess the dataset**

Before pretraining, it is necessary to perform tokenization on the dataset in advance. Before tokenization, you should first know the format of the dataset and the field in the dataset used to pretrain. Take [`dclm-baseline`](https://huggingface.co/datasets/mlfoundations/dclm-baseline-1.0-parquet) as an example, the data files format is parquet. And in its Dataset Card, it can be seen that the `text` field of each data entry is used for pretraining. After knowing the format type, we can use the following command to tokenize the data in advance
```bash
python path/to/dataset path/to/output_dir\
  --prefix prefix_of_output_file\ 
  --handler file_format\
  --field field_used_to_pretrain\
  --num_workers  workers_to_process\
  --tokenizer_path path/to/tokenizer\
  --max_size max_tokens_of_each_output_file
```
For example, to tokenize dclm-baseline, use following command in `PhoneLM`
```bash
python pretokenize.py path/to/dclm-baseline ./train_datasets/dclm-baseline 
  --prefix dclm-baseline 
  --handler parquet 
  --field text
  --tokenizer_path tokenizer
```
The output will look like:
```
train_datasets/
└── dclm-baseline
    ├── dclm-baseline-000-00000.data
    ├── dclm-baseline-001-00000.data
    ├── dclm-baseline-002-00000.data
    ├── dclm-baseline-003-00000.data
    ...
```

**Pretrain**

After performing the same operation on all datasets, the tokenized datasets are stored in `train_datasets`. Subsequently, you can start pretraining with the following command:
```bash
deepspeed train.py --config config_phonelm_1.5b.yaml
```

### Decay Stage

In the decay stage, the data contains some dataset from stable training stage, including DCLM-baseline, StarCoderData, and Dolma. And it also
contains some high-quality fine-tuning data, which is used in fine-tuning stage. Following table shows the data
| Type      | Dataset                     | Token |
|-----------|-----------------------------|-------|
| web       | [DCLM-baseline](https://huggingface.co/datasets/mlfoundations/dclm-baseline-1.0-parquet) | 10B   |
| code       | [StarCoderData](https://huggingface.co/datasets/bigcode/starcoderdata) | 1.575B |
| code      | [The Stack Smol](https://huggingface.co/datasets/bigcode/the-stack-smol)              | 0.95B |
| acadamic  | [Dolma-arxiv](https://huggingface.co/datasets/allenai/dolma) | 2.325B |
| acadamic  | [Dolma-pes2o](https://huggingface.co/datasets/allenai/dolma) | 2.35B |
| math instruct | [MathInstruct](https://huggingface.co/datasets/TIGER-Lab/MathInstruct) | 65.25M |
| chat instruct | [UltraChat](https://huggingface.co/datasets/stingning/ultrachat) | 1.775B |
| chat instruct | [OpenAssistant 2](https://huggingface.co/datasets/OpenAssistant/oasst2) | 42.25M |
| chat instruct | [OpenHermes](https://huggingface.co/datasets/teknium/openhermes) | 77.25M |
| code instruct | [Magicoder Evol Instruct](https://huggingface.co/datasets/ise-uiuc/Magicoder-OSS-Instruct-75K) | 30.25M |
| code instruct | [CommitPackFT](https://huggingface.co/datasets/bigcode/commitpackft) | 0.35B |
| code instruct | [Magicoder OSS Instruct](https://huggingface.co/datasets/ise-uiuc/Magicoder-OSS-Instruct-75K) | 43.5M |
| function calling | [SlimOrca](https://huggingface.co/datasets/Open-Orca/SlimOrca) | 209.75M |
| function calling | [APIGen](https://huggingface.co/datasets/Salesforce/xlam-function-calling-60k) | 48.25M |
| function calling | [Glaive Function Calling](https://huggingface.co/datasets/glaiveai/glaive-function-calling-v2) | 57.5M |
| **total**      |                            | **20B**   |

Unfortunately, the datasets in the table above, excluding those used for pretraining, each have their own format. To standardize the datasets in this phase, we have processed all SFT data into a chat format and formatted them as text using a unified template.

We will show you an example. First download the dataset as shown above.Then use the following command to process:
```bash
python prepare_chat.py path/to/MathInstruct chat/MathInstruct --dataset_name MathInstruct # process MathInstruct

python prepare_chat.py ../datasets/Magicoder-OSS-Instruct-75K/ chat/Magicoder --dataset_name Magicoder # process Magicoder
```
After processing the dataset, the `chat` directory will looks like
```bash
chat/
├── Magicoder
│   └── 000_Magicoder_00000.parquet
└── MathInstruct
    └── 000_MathInstruct_00000.parquet
```
Format of processed data is as following:
```json
{
  "text": "pretrain data",
  "chat": [
    {"role": "...", "content": "..."},
    ...
  ]
}
```

Then you can tokenize the `text` field to get the Decay Stage pretrain data using `pretokenize.py`.


## how to run
```python
from transformers import AutoTokenizer, AutoModelForCausalLM

model_name = 'mllmTeam/PhoneLM-1.5B-Instruct'
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name, trust_remote_code=True)

model = model.to('cuda')

question = "Hello, who are you?"
chat = [
    {"role": "user", "content": question},
]

prompt = tokenizer.apply_chat_template(chat, tokenize=False, add_generation_prompt=True)

inp = tokenizer(prompt, return_tensors="pt")
inp = {k: v.to('cuda') for k, v in inp.items()}
out = model.generate(**inp, 
                     max_length=256,
                     do_sample=False
                     )
text = tokenizer.decode(out[0], skip_special_tokens=True)

```

## how to sft
Initial dataset structure
```
- train_datasets_instruct
|
- - - dataset_00
    |
    - - -data_001.parquet
    |
    - - -data_002.parquet
    |
    - - - ...
```

Launch train command
```shell
deepspeed train_instruct.py --config config_phonelm_1.5b_instruct.yaml
```
If it is the first time loading train_datasets_instruct, two directories train_dataset_test and val_dataset_test will be generated in the train_datasets_instruct directory. Subsequently, data will be read directly from these two directories.
