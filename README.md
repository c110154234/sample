# -*- coding: utf-8 -*-
"""「4. AICUP_Pythia fine tune_STUDENT VERSION」的副本

Automatically generated by Colaboratory.

Original file is located at
    https://colab.research.google.com/drive/1CwpWRTnG5EULyqto6F7JgMWJh5DVtH7n
"""

!pip install transformers
!pip install datasets

from transformers import AutoTokenizer, AutoModelForCausalLM
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer

plm = "EleutherAI/pythia-70m-deduped"

bos = '<|endoftext|>'
eos = '<|END|>'
pad = '<|pad|>'
sep ='\n\n####\n\n'

special_tokens_dict = {'eos_token': eos, 'bos_token': bos, 'pad_token': pad, 'sep_token': sep}

tokenizer = AutoTokenizer.from_pretrained(plm, revision="step3000")
num_added_toks = tokenizer.add_special_tokens(special_tokens_dict)
tokenizer.padding_side = 'left'

from datasets import load_dataset, Features, Value
dataset = load_dataset("csv", data_files="opendid_set1.tsv", delimiter='\t',
                       features = Features({
                              'fid': Value('string'), 'idx': Value('int64'),
                              'content': Value('string'), 'label': Value('string')}),
                       column_names=['fid', 'idx', 'content', 'label'], keep_default_na=False)

import torch
sub_datasets = torch.utils.data.random_split(dataset['train'], [85736,0])
print(len(sub_datasets[0]))
for i in range(4): print(sub_datasets[0][i])

PAD_IDX = tokenizer.convert_tokens_to_ids(tokenizer.pad_token)
IGNORED_PAD_IDX = -100
PAD_IDX

from torch.utils.data import DataLoader
import torch

train_data = list(sub_datasets[0])

def collate_batch(batch):
    texts = [f"{bos} {data['content']} {sep}"+ data['label'].replace('\\n','\n')+f" {eos}" for data in list(batch)] # 範例 prompt
    encoded_seq = tokenizer(texts, padding=True)

    indexed_tks = torch.tensor(encoded_seq['input_ids'])
    attention_mask = torch.tensor(encoded_seq['attention_mask'])
    encoded_label = torch.tensor(encoded_seq['input_ids'])
    encoded_label[encoded_label == tokenizer.pad_token_id] = IGNORED_PAD_IDX

    return indexed_tks, encoded_label, attention_mask

train_dataloader = DataLoader(train_data, batch_size=2, shuffle=False, collate_fn=collate_batch)
titer = iter(train_dataloader)
tks, labels, masks= next(titer)
print(tks.shape)
next(iter(titer))

import random
BATCH_SIZE = 8 # 自行決定大小

class BatchSampler():
    def __init__(self, data, batch_size):
        self.pooled_indices = []
        self.data = data
        self.batch_size = batch_size
        self.len = len(list(data))
    def __iter__(self):
        self.pooled_indices = []
        indices = [(index, len(data["content"])) for index, data in enumerate(self.data)]
        random.shuffle(indices)
        for i in range(0, len(indices), BATCH_SIZE * 100):
            self.pooled_indices.extend(sorted(indices[i:i + BATCH_SIZE * 100], key=lambda x: x[1], reverse=True))
        self.pooled_indices = [x[0] for x in self.pooled_indices]

        for i in range(0, len(self.pooled_indices), BATCH_SIZE):
            yield self.pooled_indices[i:i + BATCH_SIZE]
    def __len__(self):
        return (self.len + self.batch_size - 1) // self.batch_size

bucket_train_dataloader = DataLoader(train_data, batch_sampler=BatchSampler(train_data, BATCH_SIZE),
                                     collate_fn=collate_batch, pin_memory=True)

from transformers import AutoConfig
config = AutoConfig.from_pretrained(plm,
                                    bos_token_id=tokenizer.bos_token_id,
                                    eos_token_id=tokenizer.eos_token_id,
                                    pad_token_id=tokenizer.pad_token_id,
                                    sep_token_id=tokenizer.sep_token_id,
                                    output_hidden_states=False)

model = AutoModelForCausalLM.from_pretrained(plm, revision="step3000", config=config)
model

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model.to(device)

import torch
from tqdm import tqdm#, tqdm_notebook
from torch.nn import functional as F

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

def sample_text(model, tokenizer, seed, n_words=20):
    model = model.to(device)
    model.eval()
    text = tokenizer.encode(seed)
    inputs, past_key_values = torch.tensor([text]), None
    with torch.no_grad():
        for _ in tqdm(range(n_words)):
            out = model(inputs.to(device), past_key_values=past_key_values)
            logits = out.logits
            past_key_values = out.past_key_values
            log_probs = F.softmax(logits[:, -1], dim=-1)
            inputs = torch.multinomial(log_probs, 1)
            text.append(inputs.item())
            if tokenizer.decode(inputs.item()) == eos:
                break


    return tokenizer.decode(text)

sample_text(model, tokenizer, seed=f"{bos} DR AADLAND ABRAHAM {sep}")

from transformers import get_linear_schedule_with_warmup
from torch.optim import AdamW

EPOCHS = 5 # 設定你的訓練次數
optimizer = AdamW(model.parameters(),lr=5e-5)

steps = len(bucket_train_dataloader)
total_steps = steps * EPOCHS
print(steps, total_steps)

scheduler = get_linear_schedule_with_warmup(
    optimizer,
    num_warmup_steps=total_steps*0.1,
    num_training_steps=total_steps
)

model.resize_token_embeddings(len(tokenizer))
model.to(device)
print(f'Total numbers of steps: {total_steps}')
model

from tqdm import tqdm,trange

global_step = 0
total_loss = 0

model.train()
for _ in trange(EPOCHS, desc="Epoch"):
    model.train()
    total_loss = 0

    predictions , true_labels = [], []

    for step, (seqs, labels, masks) in enumerate(bucket_train_dataloader):
        seqs = seqs.to(device)
        labels = labels.to(device)
        masks = masks.to(device)
        model.zero_grad()
        outputs = model(seqs, labels=labels)#, attention_mask=masks)
        logits = outputs.logits
        loss = outputs.loss
        loss = loss.mean()
        total_loss += loss.item()
        loss.backward()
        optimizer.step()
        scheduler.step()  # Update learning rate schedule
    avg_train_loss = total_loss / len(bucket_train_dataloader)
    print("Average train loss: {}".format(avg_train_loss))

from datasets import load_dataset, Features, Value
valid_data = load_dataset("csv", data_files="/content/opendid_test.tsv", delimiter='\t',
                          features = Features({
                              'fid': Value('string'), 'idx': Value('int64'),
                              'content': Value('string'), 'label': Value('string')}),
                              column_names=['fid', 'idx', 'content', 'label'])
valid_list= list(valid_data['train'])
valid_list

from tqdm import tqdm#, tqdm_notebook

tokenizer.padding_side = 'left'
def sample_batch(model, tokenizer, input):
    """Generate text from a trained model."""
    model.eval()
    seeds = [f"{bos} {text['content']} {sep}" for text in input]
    texts = tokenizer(seeds, return_tensors = 'pt', padding=True).to(device)
    outputs = []
    #return
    with torch.cuda.amp.autocast():
        output_tokens = model.generate(**texts, max_new_tokens=400, pad_token_id = PAD_IDX,
                                        eos_token_id=tokenizer.convert_tokens_to_ids(eos))
        preds = tokenizer.batch_decode(output_tokens)
        for idx , pred in enumerate(preds):
            pred = pred[pred.index(sep)+len(sep):].replace(pad, "").replace(eos, "").strip()
            if pred == "PHI: NULL":
                continue
            phis = pred.split('\n')
            lidxs = {}
            for p in phis:
                tid = p.find(':')
                if tid > 0:
                    text = p[tid+1:].strip()
                    nv = text.find('=>')
                    normalizedV = None
                    # 處理時間正規化
                    if nv>=0:
                        normalizedV = text[nv+2:].strip()
                        normalizedV = normalizedV.strip()
                        #print(normalizedV)

                        text = text[:nv].strip()
                    # YOU IMPLEMENTATION
                    #
                    #
                    #
                    lidx = 0
                    if text in lidxs:
                        lidx = lidxs[text]
                    lidx = input[idx]['content'].find(text, lidx)
                    eidx = lidx+len(text)
                    lidxs[text] = eidx
                    sidx=int(input[idx]['idx'])
                    if lidx+sidx!=eidx+sidx:
                        if normalizedV is None:
                            outputs.append(f'{input[idx]["fid"]}\t{p[:tid]}\t{lidx+sidx}\t{eidx+sidx}\t{text}')
                        else:
                            outputs.append(f'{input[idx]["fid"]}\t{p[:tid]}\t{lidx+sidx}\t{eidx+sidx}\t{text}\t{normalizedV}')


    return outputs

f = open("answer.txt", "w", encoding="utf-8")
BATCH_SIZE = 100
for i in tqdm(range(0, len(valid_list), BATCH_SIZE)):
    with torch.no_grad():
        seeds = valid_list[i:i+BATCH_SIZE]
        outputs = sample_batch(model, tokenizer, input=seeds)
        for o in outputs:
            f.write(o)
            f.write('\n')
f.close()
