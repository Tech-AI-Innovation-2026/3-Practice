# 중간고사 (테스트용 5문항)

## Q1. 다음 중 Transformer의 self-attention 시간복잡도로 올바른 것은? (단일선택)

1. O(n)
2. O(n log n)
3. O(n²)
4. O(n³)

## Q2. 교차엔트로피 손실 계산에서 log의 기본 밑은? (단일선택, PyTorch/TF 기준)

1. 2
2. e
3. 10
4. 임의 선택

## Q3. 다음 중 regularization 기법에 해당하는 것을 모두 고르시오. (복수선택)

1. L2 weight decay
2. Adam optimizer
3. Dropout
4. Batch Normalization 기본 학습 모드 사용
5. 학습률 상수 유지

## Q4. Batch Normalization은 inference 시 mini-batch 통계 대신 running statistics를 사용한다. (T/F)

## Q5. 총 파라미터 수가 1.3B인 모델을 fp16으로 로드할 때 필요한 최소 VRAM(GB)은? 소수점 둘째자리까지 기재. (숫자)

- 힌트: 파라미터당 2바이트
- 1GB = 10⁹ 바이트로 계산
- 오차 ±0.01 허용
