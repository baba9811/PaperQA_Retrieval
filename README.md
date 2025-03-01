# Korean PaperQA Retrieval
### 한국어 embeddig/retrieval evaluation
### 국내논문 QA 데이터셋 사용하여 평가하였습니다.
#### link: https://aida.kisti.re.kr/data/21b21974-6efd-4581-b9df-699f91f5bc98

기존 국내논문 QA 데이터셋에서 "context", "qas"를 사용하여 구축하였습니다.  
```
"context": "질의응답문장이 포함된 논문 풀텍스트",
"qas": [
    {
        "level": "난이도 (0:일반, 1:하, 2:상)",
        "question": "질의",
        "answer": {
            "answer_text": "응답에 해당하는 텍스트",
            "answer_start": "응답 시작 인덱스"
        }
    }
]
```
해당 데이터셋을 일부 수정하여 새로 구축하였습니다.  
"qas" 내부의 "question"을 "query"로 사용하였고,  
"context"와 "answer_start"를 이용하여 최대 길이 512의 "context"를 생성했습니다.  
이 때 검색되어야 하는 문서의 앞쪽에 관련 정보들이 모여있을 가능성이 높으므로,  
"answer_start" index 앞에 128 character를 포함하도록 context를 설정하였습니다.

### 총 데이터셋 개수
level: 0,1,2 각각에 해당하는 데이터셋 274,779개씩 포함하여,  
총 824,337개의 데이터셋을 구축하였습니다.

### Retrieval Evaluation

#### Random Sampled Dataset

Repository 내 data directory 안의 데이터셋은 총 824,337개의 전체 데이터 셋 중에서  
10,000개를 랜덤 샘플링하여 구성한 데이터셋입니다.  
데이터셋 예시는 아래와 같습니다.

```
    {
        "id": 33
        "level": 1,
        "question": "SNEP를 사용하기 위해 무엇이 필요한가?",
        "context": [
            "데이터 기밀성과 두 노드 사이에서의 데이터 인증, 무결성 및 데이터 freshness를 제공할 수 있는\
             SNEP과 데이터 브로드캐스트 시에 데이터 인증을 제공할 수 있는 μTESLA로 이루어져 있다.\nSNEP를\
             사용하기 위해서는 통신하기를 원하는 두 노드 간에 pairwise key를 공유하고 있어야 하는데 이를\
             위해서 네트워크의 각 노드들은 배치 이전에 베이스 스테이션과 각각 유일한 대칭키인 개인키를\
             공유하고 이를 바탕으로 베이스 스테이션이 pairwise key 설립에 참여를 함으로서 노드 간에 키를\
             설립한다. 이 대칭키에 단방향 함수(one-way function)을 취함으로 노드 사이에 전송될 메시지에\
             대한 인증키를 각 노드가 스스로 유도해낼 수 있으며 이 키들을 이용하여 노드 간에 각각의 카운터를\
             교환하여 사용함으로 약한 데이터 freshness를 이룰 수 있다.\n그리고 μTESLA를 이용하면 베이스\
             스테이션의 브로드캐스팅을 인증할 수 있는데 이는 시간의 지연을 두고 대칭키를 공표함으로\
             이루어진"
        ],
        "answer": "통신하기를 원하는 두 노드 간에 pairwise key를 공유", 
    }
```

고려대학교 nlpai-lab의 KURE-v1, BAAI의 bge-m3 모델과  
jinaai의 jina-embeddings-v3를 original retriever,  
BM25와 hybrid retrieval 방식으로 비교하였습니다.  
RTX4070 12GB GPU 1대로 FAISS-GPU를 사용하여 평가하였습니다.  
Hybrid retrieval은 BM25의 weight를 0.2로 설정하여 진행하였습니다.  
평가지표는 Recall, nDCG, MRR을 사용하였으며, 평가 batch_size는 8,  
Time은 1만개 데이터셋 총 평가에 걸린 시간입니다.

|Model Name|Recall@5(10)|nDCG@5(10)|MRR@5(10)|Time(Sec.)|
|---|---|---|---|:---:|
|nlpai-lab/KURE-v1|**0.977(0.987)**|**0.936(0.939)**|**0.922(0.923)**|21|
|nlpai-lab/KURE-v1 + BM25|0.838(0.894)|0.801(0.823)|0.789(0.800)|86|
|BAAI/bge-m3|0.950(0.970)|0.896(0.903)|0.878(0.881)|18|
|bge-m3 + BM25|0.837(0.892)|0.799(0.821)|0.786(0.798)|91|
|jinaai/jina-embeddings-v3|0.848(0.891)|0.765(0.779)|0.737(0.743)|**13**|
|jina-embeddings-v3 + BM25|0.831(0.883)|0.788(0.809)|0.774(0.786)|84|
|BM25|0.792(0.815)|0.736(0.744)|0.718(0.721)|64|

간단하게 평가를 진행했지만, 고려대학교에서 발표한 CachedGISTEmbedLoss와  
hard negative dataset을 이용해 contrastive learning으로로 학습한 KURE-v1 모델이 가장 성능이 높았습니다.  
또한, Hybrid retrieval을 했을 때 오히려 성능이 낮아지는 모습을 보였습니다.  

#### Total Dataset

다음은 데이터셋 824,337개 전체에 대해 실험한 결과입니다.  
실험은 RTX 3090 24GB GPU와 FAISS-GPU를 사용하여 평가하였습니다.  
실험 시간 관계 상 BM25와 hybrid search는 제외하고  
KURE-v1, bge-m3, jina-embeddings-v3 모델만 비교하였습니다.  
Time은 전체 데이터 평가에 걸린 총 시간의 합입니다.

|Model Name|Recall@5(10)|nDCG@5(10)|MRR@5(10)|Time(min.:sec.)|
|---|---|---|---|:---:|
|nlpai-lab/KURE-v1|**0.728(0.802)**|**0.605(0.629)**|**0.563(0.573)**|45:56|
|BAAI/bge-m3|0.657(0.735)|0.539(0.564)|0.500(0.510)|39:50|
|jinaai/jina-embeddings-v3|0.447(0.525)|0.352(0.378)|0.321(0.331)|**32:42**|

1만개를 랜덤샘플링했을 때는 비슷한 주제를 가진 documents가 데이터셋 내에 많지 않았기 때문에 임베딩 평가 시 성능이 더 높게 측정되었지만, 전체 데이터셋 내에는 비슷한 주제를 가진 documents가 다수 존재하기 때문에 더 낮은 성능을 보이는 것으로 예상됩니다.