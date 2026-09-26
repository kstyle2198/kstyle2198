좋습니다. 지금까지 진행한 내용을 **Google Colab에서 처음부터 끝까지 따라 할 수 있는 하나의 실습 과정**으로 다시 정리하겠습니다.

핵심 목표는 다음입니다.

> **한국어 굴착기 음성 명령 → MiniLM 임베딩 → Intent Decision Head → Logit → 확률 → Confidence → Brier/ECE → 실행/확인 판단**

---

# 1. 전체 실습 구조

```text
한국어 명령어
     │
     ▼
┌──────────────────────────┐
│ MiniLM Encoder           │
│ 384차원 Embedding        │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Decision Head            │
│ 384 → 128 → 12           │
└────────────┬─────────────┘
             │
             ▼
          Logits
             │
             ▼
         Softmax
             │
             ▼
       Intent Probability
             │
             ▼
        Confidence
             │
      ┌──────┴──────┐
      ▼             ▼
   높음             낮음
      │             │
      ▼             ▼
   EXECUTE        CONFIRM
```

이번 실습에서는 **MiniLM Encoder는 고정(freeze)**하고, 그 위의 Decision Head를 학습합니다.

---

# 2. 실습 데이터

현재 사용하는 Intent는 12개입니다.

```text
BOOM_UP
BOOM_DOWN
ARM_OUT
ARM_IN
BUCKET_OPEN
BUCKET_CLOSE
SWING_LEFT
SWING_RIGHT
TRAVEL_FORWARD
TRAVEL_BACKWARD
HYDRAULIC_TEMP_QUERY
ENGINE_TEMP_QUERY
```

예를 들어:

```text
"붐 올려"
"붐을 위로 올려줘"
"붐 상승"

→ BOOM_UP
```

```text
"붐 내려"
"붐을 아래로 내려"
"붐 하강"

→ BOOM_DOWN
```

현재 데이터는 **실습용 synthetic 데이터**입니다.

따라서 이번 실습에서 높은 Accuracy가 나오더라도 실제 굴착기 현장의 정확도를 의미하지는 않습니다.

---

# 3. Colab 환경 설치

### Cell 1

```python
!pip install -q sentence-transformers transformers scikit-learn pandas matplotlib seaborn
```

---

# 4. GPU 확인

### Cell 2

```python
import torch

print("PyTorch:", torch.__version__)
print("CUDA 사용 가능:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
```

예:

```text
PyTorch: 2.x.x
CUDA 사용 가능: True
GPU: Tesla T4
```

---

# 5. 데이터 업로드

### Cell 3

```python
from google.colab import files

uploaded = files.upload()
```

여기서

```text
excavator_intents_182.json
```

파일을 업로드합니다.

---

# 6. JSON 데이터 읽기

### Cell 4

```python
import json
import pandas as pd

FILE_NAME = "excavator_intents_182.json"

with open(FILE_NAME, "r", encoding="utf-8") as f:
    data = json.load(f)

df = pd.DataFrame(data)

print(df.head())
print()
print(df["intent"].value_counts())
```

데이터는 기본적으로 다음 구조입니다.

```text
text                    intent
------------------------------------
붐 올려                  BOOM_UP
붐을 위로 올려줘         BOOM_UP
붐 상승                  BOOM_UP
...
```

---

# 7. Intent를 숫자로 변환

신경망은 문자열보다 숫자 Label을 사용합니다.

### Cell 5

```python
labels = sorted(df["intent"].unique())

label2id = {
    label: i
    for i, label in enumerate(labels)
}

id2label = {
    i: label
    for label, i in label2id.items()
}

df["label"] = df["intent"].map(label2id)

print(label2id)
print(df[["text", "intent", "label"]].head())
```

예:

```text
BOOM_DOWN          → 0
BOOM_UP            → 1
...
```

실제 숫자 자체에는 의미가 없습니다.

단순히:

```text
문자열 Intent
      ↓
숫자 ID
```

로 바꾼 것입니다.

---

# 8. Train / Validation / Test 분리

### Cell 6

```python
from sklearn.model_selection import train_test_split

train_df, temp_df = train_test_split(
    df,
    test_size=0.2,
    stratify=df["label"],
    random_state=42
)

val_df, test_df = train_test_split(
    temp_df,
    test_size=0.5,
    stratify=temp_df["label"],
    random_state=42
)

print("Train:", len(train_df))
print("Validation:", len(val_df))
print("Test:", len(test_df))
```

개념은:

```text
전체 데이터
    │
    ├── Train       → 학습
    │
    ├── Validation  → 중간 검증
    │
    └── Test        → 최종 평가
```

입니다.

---

# 9. MiniLM Encoder 준비

### Cell 7

```python
from sentence_transformers import SentenceTransformer

MODEL_NAME = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"

encoder = SentenceTransformer(MODEL_NAME)

print("MiniLM loaded")
```

이 모델은 문장을 받아서 **384차원 벡터**로 변환합니다.

예:

```text
"붐을 올려줘"
        ↓
MiniLM
        ↓
[0.12, -0.34, 0.82, ...]
        ↓
384차원
```

---

# 10. Text와 Label 준비

### Cell 8

```python
train_texts = train_df["text"].tolist()
val_texts = val_df["text"].tolist()
test_texts = test_df["text"].tolist()

train_labels = train_df["label"].tolist()
val_labels = val_df["label"].tolist()
test_labels = test_df["label"].tolist()
```

중요합니다.

Label은 반드시:

```text
0, 1, 2, 3...
```

형태여야 합니다.

`"BOOM_UP"` 같은 문자열을 그대로 `torch.tensor()`에 넣으면 오류가 발생합니다.

---

# 11. MiniLM Embedding 생성

현재 실습에서는 MiniLM 자체는 학습하지 않습니다.

따라서 Embedding을 미리 만들어 놓습니다.

### Cell 9

```python
import torch

train_embeddings = torch.tensor(
    encoder.encode(
        train_texts,
        convert_to_numpy=True
    ),
    dtype=torch.float32
)

val_embeddings = torch.tensor(
    encoder.encode(
        val_texts,
        convert_to_numpy=True
    ),
    dtype=torch.float32
)

test_embeddings = torch.tensor(
    encoder.encode(
        test_texts,
        convert_to_numpy=True
    ),
    dtype=torch.float32
)

train_labels = torch.tensor(
    train_labels,
    dtype=torch.long
)

val_labels = torch.tensor(
    val_labels,
    dtype=torch.long
)

test_labels = torch.tensor(
    test_labels,
    dtype=torch.long
)

print("Train:", train_embeddings.shape)
print("Val:", val_embeddings.shape)
print("Test:", test_embeddings.shape)
```

예:

```text
Train: torch.Size([134, 384])
Val:   torch.Size([17, 384])
Test:  torch.Size([17, 384])
```

즉:

```text
134개 문장
×
384차원
```

입니다.

---

# 12. Decision Head 만들기

### Cell 10

```python
import torch.nn as nn

NUM_CLASSES = len(labels)

class IntentClassifier(nn.Module):

    def __init__(self, input_dim=384, num_classes=12):

        super().__init__()

        self.classifier = nn.Sequential(
            nn.Linear(input_dim, 128),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(128, num_classes)
        )

    def forward(self, x):
        return self.classifier(x)


device = torch.device(
    "cuda" if torch.cuda.is_available() else "cpu"
)

classifier = IntentClassifier(
    input_dim=384,
    num_classes=NUM_CLASSES
).to(device)

print(classifier)
```

구조는:

```text
384
 ↓
Linear
 ↓
128
 ↓
ReLU
 ↓
Dropout
 ↓
12
```

입니다.

마지막 12개 값이 바로 **Logit**입니다.

---

# 13. DataLoader 만들기

### Cell 11

```python
from torch.utils.data import TensorDataset, DataLoader

train_dataset = TensorDataset(
    train_embeddings,
    train_labels
)

val_dataset = TensorDataset(
    val_embeddings,
    val_labels
)

test_dataset = TensorDataset(
    test_embeddings,
    test_labels
)

BATCH_SIZE = 16

train_loader = DataLoader(
    train_dataset,
    batch_size=BATCH_SIZE,
    shuffle=True
)

val_loader = DataLoader(
    val_dataset,
    batch_size=BATCH_SIZE,
    shuffle=False
)

test_loader = DataLoader(
    test_dataset,
    batch_size=BATCH_SIZE,
    shuffle=False
)
```

---

# 14. Brier Loss 만들기

이번 실습의 중요한 부분입니다.

```python
def brier_loss(logits, labels):

    probabilities = torch.softmax(
        logits,
        dim=1
    )

    target = torch.zeros_like(probabilities)

    target.scatter_(
        1,
        labels.unsqueeze(1),
        1.0
    )

    loss = (probabilities - target) ** 2

    return loss.sum(dim=1).mean()
```

쉽게 말하면:

```text
모델 확률
     ↓
실제 정답
     ↓
둘의 차이를 계산
```

합니다.

---

# 15. Loss와 Optimizer

### Cell 12

```python
criterion = nn.CrossEntropyLoss()

optimizer = torch.optim.Adam(
    classifier.parameters(),
    lr=0.01
)

BRIER_WEIGHT = 0.3
EPOCHS = 50
```

전체 Loss는:

```text
Loss
 =
CrossEntropy
 +
0.3 × Brier
```

입니다.

즉,

```text
분류를 잘하는 것
+
확률을 너무 자신 있게 틀리지 않는 것
```

을 함께 학습합니다.

---

# 16. Decision Head 학습

### Cell 13

```python
for epoch in range(EPOCHS):

    classifier.train()

    total_loss = 0.0
    total_ce = 0.0
    total_brier = 0.0

    for batch_embeddings, batch_labels in train_loader:

        batch_embeddings = batch_embeddings.to(device)
        batch_labels = batch_labels.to(device)

        optimizer.zero_grad()

        logits = classifier(batch_embeddings)

        ce_loss = criterion(
            logits,
            batch_labels
        )

        brier = brier_loss(
            logits,
            batch_labels
        )

        loss = ce_loss + BRIER_WEIGHT * brier

        loss.backward()

        optimizer.step()

        total_loss += loss.item()
        total_ce += ce_loss.item()
        total_brier += brier.item()

    avg_loss = total_loss / len(train_loader)
    avg_ce = total_ce / len(train_loader)
    avg_brier = total_brier / len(train_loader)

    if (epoch + 1) % 5 == 0:

        print(
            f"Epoch {epoch+1:02d} | "
            f"Loss={avg_loss:.4f} | "
            f"CE={avg_ce:.4f} | "
            f"Brier={avg_brier:.4f}"
        )
```

여기까지 완료했다면 **학습 완료**입니다.

---

# 17. 모델 평가 함수

### Cell 14

```python
def evaluate_model(model, dataloader, device):

    model.eval()

    all_labels = []
    all_predictions = []
    all_probabilities = []
    all_logits = []

    with torch.no_grad():

        for embeddings, labels in dataloader:

            embeddings = embeddings.to(device)
            labels = labels.to(device)

            logits = model(embeddings)

            probabilities = torch.softmax(
                logits,
                dim=1
            )

            predictions = torch.argmax(
                probabilities,
                dim=1
            )

            all_labels.extend(
                labels.cpu().tolist()
            )

            all_predictions.extend(
                predictions.cpu().tolist()
            )

            all_probabilities.append(
                probabilities.cpu()
            )

            all_logits.append(
                logits.cpu()
            )

    all_probabilities = torch.cat(
        all_probabilities
    )

    all_logits = torch.cat(
        all_logits
    )

    return (
        all_labels,
        all_predictions,
        all_probabilities,
        all_logits
    )
```

---

# 18. Test Accuracy 확인

### Cell 15

```python
from sklearn.metrics import accuracy_score
from sklearn.metrics import classification_report

test_labels_list, test_predictions, test_probs, test_logits = evaluate_model(
    classifier,
    test_loader,
    device
)

test_accuracy = accuracy_score(
    test_labels_list,
    test_predictions
)

print(
    f"Test Accuracy: {test_accuracy:.4f}"
)
```

그리고 Intent별 결과:

```python
print(
    classification_report(
        test_labels_list,
        test_predictions,
        labels=list(range(NUM_CLASSES)),
        target_names=labels,
        zero_division=0
    )
)
```

여기서 확인할 것은:

```text
Precision
Recall
F1-score
```

입니다.

---

# 19. Test Brier Score

```python
test_brier = brier_loss(
    test_logits,
    torch.tensor(
        test_labels_list,
        dtype=torch.long
    )
)

print(
    f"Test Brier Score: {test_brier.item():.4f}"
)
```

**낮을수록 좋습니다.**

---

# 20. ECE 계산

```python
def expected_calibration_error(
    probabilities,
    labels,
    n_bins=10
):

    confidences, predictions = torch.max(
        probabilities,
        dim=1
    )

    accuracies = (
        predictions == labels
    )

    ece = torch.tensor(0.0)

    bin_boundaries = torch.linspace(
        0,
        1,
        n_bins + 1
    )

    for i in range(n_bins):

        lower = bin_boundaries[i]
        upper = bin_boundaries[i + 1]

        in_bin = (
            (confidences > lower) &
            (confidences <= upper)
        )

        count = in_bin.sum()

        if count > 0:

            accuracy_in_bin = (
                accuracies[in_bin]
                .float()
                .mean()
            )

            confidence_in_bin = (
                confidences[in_bin]
                .mean()
            )

            ece += (
                count.float()
                / len(labels)
            ) * torch.abs(
                confidence_in_bin
                - accuracy_in_bin
            )

    return ece.item()
```

실행:

```python
test_ece = expected_calibration_error(
    test_probs,
    torch.tensor(
        test_labels_list,
        dtype=torch.long
    )
)

print(
    f"Test ECE: {test_ece:.4f}"
)
```

---

# 21. 이제 실제 문장으로 추론

여기부터 재미있는 부분입니다.

```python
def predict_intent(text):

    classifier.eval()

    embedding = encoder.encode(
        [text],
        convert_to_tensor=True
    ).clone().detach().to(device)

    with torch.no_grad():

        logits = classifier(
            embedding
        )

        probabilities = torch.softmax(
            logits,
            dim=1
        )

        confidence, predicted_id = torch.max(
            probabilities,
            dim=1
        )

    intent = id2label[
        predicted_id.item()
    ]

    return (
        intent,
        confidence.item(),
        probabilities[0].cpu()
    )
```

테스트:

```python
test_sentences = [
    "붐을 위로 올려",
    "붐 내려",
    "암을 앞으로 밀어",
    "버킷 열어",
    "버킷 닫아",
    "왼쪽으로 선회해",
    "오른쪽으로 돌아",
    "앞으로 이동해",
    "엔진 온도가 몇 도야",
    "유압 온도 알려줘"
]

for text in test_sentences:

    intent, confidence, probabilities = predict_intent(text)

    print(
        f"{text:20s} → "
        f"{intent:25s} "
        f"confidence={confidence:.3f}"
    )
```

---

# 22. Logit → Probability → Confidence 관계

이번 실습의 핵심을 다시 정리하면:

```text
MiniLM
  ↓
384차원 Embedding
  ↓
Decision Head
  ↓
Logits
  ↓
Softmax
  ↓
Probability
  ↓
가장 높은 Probability
  ↓
Confidence
```

예를 들어:

```text
Logit

BOOM_UP       5.2
BOOM_DOWN     1.1
ARM_OUT       0.3
...
```

Softmax를 적용하면:

```text
BOOM_UP       0.96
BOOM_DOWN     0.01
ARM_OUT       0.01
...
```

따라서:

```text
Prediction = BOOM_UP
Confidence = 0.96
```

가 됩니다.

**Confidence는 별도의 신비한 값이 아니라, 기본적으로 가장 높은 softmax probability입니다.**

다만 이 값이 실제 정답 가능성과 잘 맞는지는 **Calibration(Brier/ECE)**을 확인해야 합니다.

---

# 23. Confidence Gate

이제 실제 장비 제어를 생각해봅니다.

```python
CONFIDENCE_THRESHOLD = 0.90

def decision(text):

    intent, confidence, probabilities = predict_intent(text)

    print(f"입력       : {text}")
    print(f"Intent     : {intent}")
    print(f"Confidence : {confidence:.3f}")

    if confidence >= CONFIDENCE_THRESHOLD:

        print(
            f"[EXECUTE] {intent}"
        )

        return {
            "action": "EXECUTE",
            "intent": intent,
            "confidence": confidence
        }

    else:

        print(
            "[CONFIRM] 확신도가 낮습니다."
        )

        return {
            "action": "CONFIRM",
            "intent": intent,
            "confidence": confidence
        }
```

실행:

```python
decision("붐을 올려줘")
```

예:

```text
입력       : 붐을 올려줘
Intent     : BOOM_UP
Confidence : 0.96

[EXECUTE] BOOM_UP
```

---

# 24. 최종적으로 만들고 싶은 구조

현재 실습이 끝나면 모델 부분은 다음과 같습니다.

```text
              "붐을 올려줘"
                    │
                    ▼
              ┌──────────┐
              │  MiniLM  │
              └────┬─────┘
                   384
                    │
                    ▼
             ┌────────────┐
             │Decision Head│
             │ 384→128→12 │
             └─────┬──────┘
                   │
                   ▼
                 Logit
                   │
                   ▼
                Softmax
                   │
                   ▼
          Probability Vector
                   │
                   ▼
          ┌────────────────┐
          │ BOOM_UP = 0.96 │
          └───────┬────────┘
                  │
                  ▼
              Confidence
                  │
           ┌──────┴──────┐
           │             │
        ≥ Threshold    < Threshold
           │             │
           ▼             ▼
       ECU 실행        확인 요청
```

---

# 25. 그리고 실제 굴착기 시스템으로 확장

현재:

```text
Text
 ↓
MiniLM
 ↓
Intent
```

다음 단계:

```text
Mic
 ↓
Silero VAD
 ↓
Streaming Whisper
 ↓
Korean Text
 ↓
MiniLM
 ↓
Decision Head
 ↓
Intent + Confidence
 ↓
Confidence Gate
 ↓
ECU API
```

그리고 나중에는 **파라미터 추출**까지 추가합니다.

예:

```text
"붐을 20도 올려"

        ↓

Intent
BOOM_UP

Parameter
angle = 20
unit = degree

        ↓

ECU Command

BOOM_UP(angle=20)
```

따라서 최종적으로는:

```text
             음성
              ↓
             STT
              ↓
       ┌──────┴───────┐
       ↓              ↓
 Intent Classifier  Parameter
       ↓              ↓
    BOOM_UP          20°
       └──────┬───────┘
              ↓
       Decision Model
              ↓
    Intent + Parameter
              ↓
       Confidence Gate
          ↓         ↓
       Execute    Confirm
          ↓
         ECU
```

이것이 지금 만들고 있는 **Jev-like 굴착기 음성 Decision Model의 전체 실습 로드맵**입니다.

### 현재 위치

```text
✅ 데이터 생성
✅ Train / Validation / Test 분리
✅ MiniLM 384-d Embedding
✅ Decision Head 384→128→12
✅ CrossEntropy
✅ Brier Loss
✅ 학습 완료

👉 지금 할 단계
   Test 평가
   ↓
   Brier / ECE
   ↓
   실제 문장 추론
   ↓
   Confidence Threshold 실험

👉 그 다음
   STT 연결
   ↓
   ECU Mock API 연결
   ↓
   음성 → ECU 전체 실습
```

특히 **다음 실습에서는 Test 결과를 표와 그래프로 시각화하고, `0.70 / 0.80 / 0.90 / 0.95` 등의 threshold별로 `EXECUTE`와 `CONFIRM`이 어떻게 달라지는지 확인**하면 좋습니다.
