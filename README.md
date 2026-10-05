물론. README에 바로 넣을 수 있게 한국어로 정리하면 아래처럼 쓰면 돼.

```markdown
## Pointwise Relevance Gate Ablation

SQuAD와 HotpotQA 각각 100개 샘플을 대상으로 두 가지 retrieval 전략을 비교했다.

- **no_gate**: BM25 + Dense Retrieval → RRF → 상위 evidence를 그대로 사용
- **pointwise_gate**: BM25 + Dense Retrieval → RRF → pointwise relevance filtering → relevant evidence만 사용
- 최대 retrieval round: 3
- 주요 평가 지표: gold evidence recall

### 결과

| Dataset | Method | N | Final Recall | All-Gold Recall | Step 1 Recall | Step 2 Recall | Step 3 Recall | Avg. Steps | Avg. Evidence |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| HotpotQA | no_gate | 100 | **0.880** | **0.760** | **0.605** | **0.873** | **0.806** | 1.77 | 3.50 |
| HotpotQA | pointwise_gate | 100 | 0.785 | 0.620 | 0.600 | 0.717 | 0.595 | **1.67** | **1.98** |
| SQuAD | no_gate | 100 | **0.760** | **0.760** | 0.710 | **0.429** | 0.250 | 1.33 | 2.65 |
| SQuAD | pointwise_gate | 100 | **0.760** | **0.760** | **0.750** | 0.350 | **0.455** | **1.31** | **1.96** |

### 주요 결과

#### 1. Pointwise gate는 context 크기를 크게 줄였다

Pointwise relevance gate를 적용하면 ReAct loop에 전달되는 evidence 수가 일관되게 감소했다.

- **HotpotQA:** 3.50 → **1.98** documents, 약 **43.4% 감소**
- **SQuAD:** 2.65 → **1.96** documents, 약 **26.0% 감소**

평균 retrieval step 수도 소폭 감소했다.

- HotpotQA: 1.77 → **1.67**
- SQuAD: 1.33 → **1.31**

즉, pointwise gate는 불필요한 context를 줄이는 **context filtering mechanism**으로는 효과적이었다.

#### 2. SQuAD에서는 recall을 유지하면서 context를 줄였다

SQuAD의 최종 recall은 두 방식이 동일했다.

- no_gate: **0.760**
- pointwise_gate: **0.760**

하지만 pointwise gate는 평균 evidence 수를 2.65에서 1.96으로 줄였다.

따라서 single-hop 성격이 강한 SQuAD에서는 pointwise filtering이 retrieval recall을 유지하면서 불필요한 context를 줄이는 데 효과적이었다.

Step 1 recall도 pointwise gate가 더 높았다.

- no_gate: 0.710
- pointwise_gate: **0.750**

#### 3. HotpotQA에서는 pointwise gate가 multi-hop recall을 감소시켰다

HotpotQA에서는 차이가 크게 나타났다.

- Final Recall
  - no_gate: **0.880**
  - pointwise_gate: **0.785**

- All-Gold Recall
  - no_gate: **0.760**
  - pointwise_gate: **0.620**

Pointwise gate가 context 크기는 줄였지만, 이후 retrieval hop에 필요한 intermediate evidence까지 제거하면서 multi-hop retrieval 성능이 감소한 것으로 보인다.

HotpotQA에서는 어떤 문서가 최종 질문에 직접 답하지 않더라도, 다음 검색 query를 만들기 위한 entity나 intermediate fact를 포함할 수 있다.

따라서 intermediate step에서 binary relevance filtering을 강하게 적용하면 유용한 bridge evidence까지 제거될 수 있다.

#### 4. Intermediate filtering에는 recall-efficiency trade-off가 존재한다

HotpotQA 결과는 다음과 같은 trade-off를 보여준다.

```text
Pointwise Gate
    ↓
Irrelevant document 감소
    ↓
Context 크기 감소
    ↓
Retrieval step 감소
    ↓
하지만 bridge evidence 제거 가능성 증가
    ↓
Multi-hop recall 감소
```

반대로 `no_gate`는 더 많은 context를 유지하기 때문에 일부 irrelevant document도 포함되지만, 이후 ReAct step에서 필요한 intermediate evidence를 보존할 가능성이 높다.

### 결론

Pointwise relevance gate는 **context efficiency를 개선하는 데 효과적**이지만, task 특성에 따라 성능 차이가 크게 나타났다.

SQuAD와 같은 single-hop QA에서는 평균 evidence 수를 약 **26% 감소**시키면서 동일한 최종 recall을 유지했다.

반면 HotpotQA와 같은 multi-hop QA에서는 final recall이 **0.880 → 0.785**, all-gold recall이 **0.760 → 0.620**으로 감소했다.

따라서 multi-hop retrieval에서는 intermediate document를 너무 엄격하게 binary filtering하는 방식보다, partially relevant한 bridge evidence를 보존하는 high-recall gate가 더 적합할 가능성이 높다.

추천 구조는 다음과 같다.

```text
Retrieve
→ RRF
→ High-Recall / Soft Relevance Gate
→ Partially Relevant Bridge Evidence 유지
→ ReAct Follow-up Retrieval
→ Final Sufficiency Check
```

### Takeaway

> Pointwise relevance gate는 context 크기를 줄이는 데 효과적이지만, multi-hop retrieval에서는 strict filtering이 bridge evidence를 제거해 recall을 낮출 수 있다.  
> ReAct 기반 multi-hop search에서는 relevance gate의 precision보다 recall을 우선하고, partially relevant evidence를 최대한 보존하는 것이 중요하다.
```

한 가지 주의점은 `recall_step_2`, `recall_step_3`가 해당 step까지 실제로 진행한 샘플만 대상으로 평균낸 값이라면, step 간 숫자를 단순한 누적 향상으로 해석하면 안 돼. README에서는 `Final Recall`과 `All-Gold Recall`을 메인 지표로 보는 게 가장 안전해.
