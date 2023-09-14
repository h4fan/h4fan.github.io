---
layout: post
title:  Beginner Level - Backdoor llm through from_pretrained
tags: [llm,backdoor]
---

You cloned the code from github. And you downloaded the model file from some website. You run the demo code. Then you got calced.
This is the demo code taken from `***`. Is it safe to run the code?  

```python
tokenizer = AutoTokenizer.from_pretrained("./model-int4/", trust_remote_code=True)
```
It's safe, is it?
# How to backdoor through from_pretrained function?

if you add this code in the file `tokenization_demogpt.py` in the model directory, it will execute.  
```python
import os
os.system('/System/Applications/Calculator.app/Contents/MacOS/Calculator')
```
Let's examine the function.
[from_pretrained](https://github.com/huggingface/transformers/blob/05de038f3d249ce96740885f85fd8d0aa00c29bc/src/transformers/models/auto/tokenization_auto.py#L568)

```
trust_remote_code (`bool`, *optional*, defaults to `False`):
                Whether or not to allow for custom models defined on the Hub in their own modeling files. This option
                should only be set to `True` for repositories you trust and in which you have read the code, as it will
                execute code present on the Hub on your local machine.
```
If trust_remote_code is set to True, it will execute code on your local machine.  

If you read code, you can see that the code will load the `tokenizer_config.json`. (TOKENIZER_CONFIG_FILE = "tokenizer_config.json")[https://github.com/huggingface/transformers/blob/05de038f3d249ce96740885f85fd8d0aa00c29bc/src/transformers/tokenization_utils.py#L49]
```json
{
  "name_or_path": "******",
  "remove_space": false,
  "do_lower_case": false,
  "tokenizer_class": "DemoTokenizer",
  "auto_map": {
    "AutoTokenizer": [
      "tokenization_demogpt.DemoTokenizer",
      null
      ]
  }
}
```
Finally, it will read `auto_map` key. In the end, it will load `tokenization_demogpt`. Then our code will execute.


# What to do
Always check the model files. Load from trusted websites. Will you? Do you have the energy to do it?


## code
[register_for_auto_class](https://github.com/huggingface/transformers/blob/05de038f3d249ce96740885f85fd8d0aa00c29bc/src/transformers/modeling_utils.py#L3751)
[get_class_from_dynamic_module](https://github.com/huggingface/transformers/blob/05de038f3d249ce96740885f85fd8d0aa00c29bc/src/transformers/dynamic_module_utils.py#L378C15-L378C15)

