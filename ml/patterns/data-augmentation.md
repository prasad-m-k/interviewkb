# Data Augmentation

**Topic:** [[ml/topics/feature-engineering]], [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/regularization]], [[ml/concepts/overfitting]]

## What it solves
Increasing the effective size and diversity of training data by creating modified versions of existing examples. Reduces overfitting, teaches invariances, and improves generalization — especially when labeled data is scarce.

## Computer Vision Augmentations

### Basic (always use)
```python
from torchvision import transforms
train_transform = transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomCrop(224, padding=4),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])
```

### Advanced
- **CutOut**: randomly zero a square patch of the image → forces model to use whole image
- **MixUp**: blend two images and their labels: `x̃ = λxᵢ + (1-λ)xⱼ, ỹ = λyᵢ + (1-λ)yⱼ`
- **CutMix**: paste a patch from one image into another; mix labels proportionally to area → outperforms MixUp on ImageNet
- **RandAugment**: randomly apply N transforms from a fixed set, each with magnitude M; reduces augmentation search space
- **AugMix**: apply mixed sequences of augmentations; robust to distribution shift

### What NOT to augment
- Medical images: rotation may change diagnosis (orientation matters for X-rays)
- Text images: flipping changes words
- Satellite imagery with directional features

## NLP Augmentations

### Lexical Substitution
```python
# Synonym replacement (Easy Data Augmentation)
from nlpaug.augmenter.word import SynonymAug
aug = SynonymAug()
augmented = aug.augment("The quick brown fox")
# → "The fast brown fox"
```

### Back-Translation
Translate to another language and back:
```
English: "The model failed to converge"
→ French: "Le modèle n'a pas convergé"
→ English: "The model did not converge"
```
Creates paraphrases that preserve semantics. Effective for text classification.

### Random Perturbations
- **Random deletion**: remove each word with probability p
- **Random swap**: swap two random words in the sentence
- **Random insertion**: insert a synonym of a random word at a random position

### BERT-based Augmentation
- MLM augmentation: mask tokens → fill with BERT predictions → diverse paraphrases
- Contextual word embeddings similarity for synonym replacement

## Tabular Augmentations

### SMOTE (for class imbalance)
See [[ml/concepts/class-imbalance]].

### Gaussian Noise Injection
```python
noise = np.random.normal(0, 0.01, X_train.shape)
X_augmented = X_train + noise
```
Trains robustness to small perturbations; implicit regularization.

### Feature Dropout (Cutout for tabular)
Randomly set some feature values to their mean/median during training — forces the model to make predictions without every feature → more robust to missing values at inference.

### MixUp for Tabular
```python
λ = np.random.beta(0.2, 0.2)
x_mix = λ * X[i] + (1 - λ) * X[j]
y_mix = λ * y[i] + (1 - λ) * y[j]
```
Works well for regression and some classification problems.

## When Augmentation Helps vs. Hurts

| Helps | Hurts |
|---|---|
| Limited labeled data | Augmentation breaks task-relevant structure |
| Model is overfitting | When distribution is well-sampled |
| Domain invariances are known | When label changes with the transformation |
| Pre-training for self-supervised learning | High-quality large dataset (diminishing returns) |

## Common interview angles
- Why does MixUp improve generalization? (trains on convex hull of training examples; smooths decision boundaries; discourages overconfident predictions)
- When is back-translation better than synonym replacement? (when semantic meaning must be preserved; back-translation produces more natural paraphrases; synonym replacement can break context)
- What is test-time augmentation (TTA)? (apply augmentations at inference and average predictions; improves accuracy ~0.5-1%; slows inference by N× number of augmentations)

## Sources
- [[ML overview]]
