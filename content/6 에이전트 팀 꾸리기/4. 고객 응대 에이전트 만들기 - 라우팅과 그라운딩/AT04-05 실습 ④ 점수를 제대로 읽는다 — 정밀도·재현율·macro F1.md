# [에이전트 팀 꾸리기] - 5. 실습 ④ 점수를 제대로 읽는다 — 정밀도·재현율·macro F1

---

## 정확도 하나로는 부족하다

앞 섹션에서 규칙 라우터의 정확도를 구했습니다. 그런데 정확도(accuracy)는 "전체 중 몇 개를 맞혔나"만 알려 줍니다. **어디가 틀렸는지**는 전혀 알려 주지 않아요. 개선을 하려면 어디가 틀렸는지를 알아야 하므로 지표를 더 세분해서 봐야 합니다.

세 가지를 봅니다. 특정 라우트 R에 대해 **정밀도(precision)** 는 "모델이 R이라고 예측한 것들 중 실제로 R이었던 비율"입니다. 낮으면 R을 남발하고 있다는 뜻이고, 실무적으로는 **엉뚱한 담당자에게 문의가 잘못 배정되는 빈도**에 대응하죠. **재현율(recall)** 은 "실제로 R인 것들 중 모델이 R이라고 맞힌 비율"이고, 낮으면 R인 문의를 놓치고 있다는 뜻입니다. **F1**은 둘의 조화평균이라 한쪽만 높은 상태를 좋은 점수로 인정하지 않습니다. 정밀도 1.0에 재현율 0.1이면 F1은 약 0.18이에요.

클래스가 여러 개일 때 종합하는 방식이 두 가지 있습니다. **macro 평균**은 클래스별 F1을 단순 평균해 클래스마다 동등한 무게를 주고, **weighted 평균**은 데이터 수로 가중해 큰 클래스가 점수를 지배합니다. 우리는 소수 라우트의 실패도 그대로 보고 싶으므로 **macro F1**을 주 지표로 삼습니다.

공식을 직접 구현하지는 않겠습니다. `scikit-learn`의 `classification_report` 와 `confusion_matrix` 가 표준이고, 실무에서 이것을 두고 손으로 계산할 일은 없거든요. 대신 **출력을 읽는 데 시간을 씁니다.** 지표를 만드는 것보다 읽는 것이 훨씬 어렵고, 오늘 배울 것도 그쪽입니다.

➡️**실습: scikit-learn 으로 지표를 읽고 기준선을 확정한다**
```python
from concurrent.futures import ThreadPoolExecutor

from sklearn.metrics import classification_report, confusion_matrix, f1_score, accuracy_score

LABELS4 = ["ORDER_PLACE", "PRODUCT_INFO", "SHIPPING", "RETURN_REFUND"]


def pmap(fn, items, workers=12):
    """여러 건을 동시에 호출한다. 결과 순서는 입력 순서와 같다.

    LLM 호출은 대부분 응답을 기다리는 시간이라 동시에 보내면 거의 그 배수만큼 빨라진다.
    workers 를 너무 올리면 요청 한도(rate limit)에 걸린다. 12 정도가 무난하다.
    """
    with ThreadPoolExecutor(max_workers=workers) as ex:
        return list(ex.map(fn, items))


def evaluate_graph(graph, df_eval=None, limit=None, name="router", report=False):
    """그래프를 평가셋에 통과시켜 macro F1과 정확도를 계산한다."""
    df_eval = eval_set if df_eval is None else df_eval
    d = df_eval if limit is None else df_eval.head(limit)

    states = pmap(lambda q: graph.invoke({"question": q}), d["question"].tolist())
    preds = [s["route"] for s in states]
    y = d["route"].tolist()

    macro_f1 = f1_score(y, preds, labels=LABELS4, average="macro", zero_division=0)
    acc = accuracy_score(y, preds)
    print(f"[{name}] n={len(d)}  정확도 {acc:.3f}  macro F1 {macro_f1:.3f}")
    if report:
        print(classification_report(y, preds, labels=LABELS4, digits=3, zero_division=0))
    # states 를 함께 돌려준다 — 같은 평가셋을 다시 부르지 않기 위해서다.
    # 호출 한 번이 곧 비용이고 수업 중 대기 시간이다.
    return {"name": name, "pred": preds, "conf": [s.get("confidence") for s in states],
            "states": states, "macro_f1": macro_f1, "acc": acc, "n": len(d), "df": d}


# 섹션 4의 규칙 라우터가 오늘의 기준선이다. 앞으로 모든 비교가 이 숫자를 기준으로 한다.
base_result = evaluate_graph(build(rule_classify), name="규칙 라우터", report=True)

cm = confusion_matrix(eval_set["route"], eval_set["pred_rule"], labels=LABELS4)
print("[혼동 행렬] 행=정답, 열=예측")
print(pd.DataFrame(cm, index=LABELS4, columns=LABELS4).to_string())
```

## 혼동 행렬을 읽는 법

혼동 행렬(confusion matrix)은 행이 정답, 열이 예측인 표입니다. **대각선 위의 숫자가 맞힌 것**이고, **대각선을 벗어난 숫자가 틀린 것**입니다. 중요한 것은 어디로 벗어났는지입니다.

예를 들어 `ORDER_PLACE` 행의 `PRODUCT_INFO` 열에 큰 숫자가 있다면, "정답은 주문·구매인데 상품 문의로 보냈다"는 뜻입니다. 이것은 앞에서 예상한 혼동이 실제로 일어났다는 증거입니다. 반대로 `SHIPPING` 행의 `RETURN_REFUND` 열에 숫자가 있다면 "반품 배송비" 같은 복합 문의에서 우선순위 규칙이 작동한 결과입니다.

혼동 행렬이 개선 작업에서 갖는 가치는 **어디를 고쳐야 할지 지목해 준다**는 점입니다. 정확도가 60%라는 사실만으로는 무엇을 해야 할지 알 수 없지만, "ORDER_PLACE의 절반이 PRODUCT_INFO로 흘러간다"는 사실을 알면 프롬프트에 그 둘의 구분 기준을 명시하면 된다는 결론이 바로 나옵니다. 오늘 오후에 정확히 그렇게 하죠.

또 하나 습관을 만듭시다. **오분류를 숫자로만 보지 말고 실제 문장을 열어 보는 것**입니다. 숫자는 어디가 틀렸는지 알려 주지만, 왜 틀렸는지는 문장을 읽어야 압니다.

➡️**실습: 오분류 사례를 직접 읽기**
```python
# 규칙 라우터가 틀린 사례를 실제 문장으로 확인한다
miss = eval_set[eval_set["pred_rule"] != eval_set["route"]]
print(f"오분류 {len(miss)}건 중 앞 8건\n")
for _, r in miss.head(8).iterrows():
    print(f"  정답 {r['route']:14s} → 예측 {r['pred_rule']:14s}")
    print(f"    Q: {r['question'][:70]}")
```

> [!question]+ 객관식 퀴즈
> **Q. 어떤 라우터가 `SHIPPING`에 대해 정밀도 0.95, 재현율 0.40을 기록했습니다. 이 라우터의 실제 동작을 가장 정확하게 묘사한 것은 무엇인가요?**
>
> 1. SHIPPING이라고 예측한 건은 거의 다 맞지만, 실제 SHIPPING 문의의 60%를 다른 라우트로 보내고 있습니다
> 2. SHIPPING 문의를 거의 다 잡아내지만, SHIPPING이 아닌 것도 많이 SHIPPING으로 보냅니다
> 3. SHIPPING의 F1은 정밀도와 재현율의 평균인 0.675가 됩니다
> 4. 정밀도가 0.9를 넘으므로 SHIPPING 라우팅은 개선할 필요가 없습니다
>
> ![힌트](https://learn.modulabs.co.kr/_next/static/media/Thick.72cfead3.svg)힌트
> >정답은 1번입니다. 정밀도가 높고 재현율이 낮은 상태는 '확실할 때만 그 라우트를 고르는' 보수적 동작이며, 그 대가로 많은 건을 놓칩니다. 2번은 정밀도와 재현율이 반대인 경우의 설명입니다. 3번은 산술평균이고 F1은 조화평균이므로 2×0.95×0.40/(0.95+0.40)≈0.563 입니다. 조화평균은 낮은 쪽에 더 끌립니다. 4번은 위험한 판단으로, 재현율 0.40은 배송 담당이 받아야 할 문의의 60%가 엉뚱한 곳으로 갔다는 뜻입니다.

#### 📚 이 섹션의 참고 자료

- [scikit-learn — Precision, recall and F-measures 설명](https://scikit-learn.org/stable/modules/model_evaluation.html) — 정밀도·재현율·F1과 macro/weighted 평균의 수식이 정리된 공식 문서다. 오늘 직접 구현한 공식과 대조해 읽으면 확실히 남는다
- [scikit-learn — confusion_matrix](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.confusion_matrix.html) — 혼동 행렬의 축 순서(행=정답, 열=예측)와 labels 인자 사용법을 확인할 수 있다. 축을 헷갈리면 해석이 완전히 뒤집히므로 한 번 확인해 둘 가치가 있다

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AT04-01 들어가며 — 분류하고, 조회하고, 근거로 답한다]]           |
|  **2강**  | [[AT04-02 실습 ① 데이터를 열고 라우트 체계를 가져온다]]           |
|  **3강**  | [[AT04-03 실습 ② 평가셋을 먼저 만든다 — 만들기 전에 재는 법부터]]           |
|  **4강**  | [[AT04-04 실습 ③ 라우터 그래프를 세우고 규칙 노드를 꽂는다]]           |
|  **5강**  | **5강. 실습 ④ 점수를 제대로 읽는다 — 정밀도·재현율·macro F1 (현재)**           |
|  **6강**  | [[AT04-06 실습 ⑤ LLM 라우터로 갈아 끼우고, 확신 없으면 넘긴다]]           |
|  **7강**  | [[AT04-07 실습 ⑥ 매뉴얼을 쪼개고 조회 경계를 도출한다]]           |
|  **8강**  | [[AT04-08 실습 ⑦ 조회 도구를 만든다]]           |
|  **9강**  | [[AT04-09 실습 ⑧ 근거만으로 답하게 만들고, 모델이 스스로 조회하게 한다]]           |
|  **10강**  | [[AT04-10 실습 ⑨ 가드레일 — 출처 없는 숫자를 기계적으로 잡는다]]           |
|  **11강**  | [[AT04-11 실습 ⑩ 정답셋으로 채점하고 전체를 한 그래프로 잇는다]]           |
|  **12강**  | [[AT04-12 정리와 회고 — 하루로 무엇을 만들었나]]           |
