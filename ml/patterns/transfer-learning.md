# Transfer Learning

**Topic:** [[ml/topics/deep-learning]], [[ml/topics/nlp]]
**Related:** [[ml/concepts/transformers]], [[ml/concepts/regularization]]

## What it solves
Leveraging a model pre-trained on a large dataset to solve a different (but related) task with limited labeled data. Critical in modern ML — almost no NLP or CV model is trained from scratch.

## Template: Fine-tuning a Pre-trained Model

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer, Trainer

# 1. Load pre-trained model + tokenizer
model = AutoModelForSequenceClassification.from_pretrained('bert-base-uncased', num_labels=2)
tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')

# 2. Freeze encoder layers (optional for small datasets)
for param in model.bert.encoder.layer[:8].parameters():
    param.requires_grad = False

# 3. Fine-tune with small learning rate
from transformers import TrainingArguments
args = TrainingArguments(
    output_dir='./results',
    learning_rate=2e-5,        # much smaller than pre-training (1e-4)
    num_train_epochs=3,
    warmup_ratio=0.1,
    weight_decay=0.01,
)
```

## Strategies by Dataset Size

| Labeled data | Strategy |
|---|---|
| < 100 examples | Zero-shot or few-shot prompting; or use embeddings + simple classifier |
| 100–1K | Freeze encoder; fine-tune classification head only |
| 1K–10K | Fine-tune top N layers + head |
| > 10K | Full fine-tune (all layers); use small LR and warmup |
| Large (100K+) | Full fine-tune; consider domain-adaptive pre-training first |

## Parameter-Efficient Fine-tuning (PEFT)

### LoRA (Low-Rank Adaptation)
```
# Instead of updating W (d × d), decompose update:
ΔW = A · B    where A is (d × r), B is (r × d), r << d

# Forward pass:
h = Wx + ΔWx = Wx + ABx

# Only A and B are trainable; W is frozen
```
- r = 8-16 typical; reduces trainable params by 10-100×
- Works remarkably well: nearly matches full fine-tune performance
- Multiple tasks can share the same base model; swap LoRA adapters per task

### QLoRA
- LoRA applied to a quantized (4-bit) base model
- Fine-tune a 70B model on a single 48GB GPU
- Slight quality degradation vs. full fine-tune; acceptable for many tasks

### Prefix Tuning / Prompt Tuning
- Prepend learnable soft tokens to the input
- Frozen model; only soft prompts are trained
- Lower quality than LoRA for most tasks; useful when model can't be modified

## Transfer Learning in Computer Vision

### ImageNet pre-training
```python
import torchvision.models as models
model = models.resnet50(pretrained=True)

# Replace final classification head
model.fc = nn.Linear(2048, num_classes)

# Fine-tune: lower LR for pre-trained layers, higher for new head
optimizer = torch.optim.SGD([
    {'params': model.fc.parameters(), 'lr': 1e-3},
    {'params': [p for n, p in model.named_parameters() if 'fc' not in n], 'lr': 1e-4}
])
```

### Domain Gap
The larger the gap between pre-training and target domain, the less transfer helps:
- ImageNet → medical X-rays: moderate gap; fine-tune all layers
- ImageNet → satellite imagery: large gap; consider domain-specific pre-training (BiT, SatMAE)
- Wikipedia BERT → legal documents: moderate; fine-tune or use legal BERT

## Progressive Unfreezing (Discriminative Fine-tuning)
1. Freeze all pre-trained layers; train classification head for 1 epoch
2. Unfreeze top 2-3 Transformer layers; train with 1/10 LR for those layers
3. Gradually unfreeze more layers with progressively lower LR
4. Final: full fine-tune with very small LR

Prevents catastrophic forgetting while allowing task-specific adaptation.

## Common interview angles
- What is catastrophic forgetting and how do transfer learning strategies prevent it? (fine-tuning on new task can overwrite pre-trained weights; fix with small LR, progressive unfreezing, LoRA)
- Why use pre-trained embeddings rather than training from scratch? (pre-training encodes billions of examples of linguistic/visual knowledge; scratch training needs massive data and compute to replicate)
- When would you NOT use transfer learning? (target domain is very different from pre-training; pre-trained model contains inappropriate biases; inference latency constraint; data is huge and task is well-defined)
- What is LoRA and why does it work? (low-rank approximation of weight updates; most fine-tuning information lives in a low-dimensional subspace of parameter space)

## Sources
- [[ML overview]]
