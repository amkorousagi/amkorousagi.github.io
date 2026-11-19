---
layout: post
tags: ["Computer Vision", "3D 기하", "선형대수", "시선 추적"]
title: "두 눈의 시선은 어디서 만날까 — 꼬인 위치의 두 직선에서 가장 가까운 점 구하기"
description: 왼쪽·오른쪽 눈의 시선 벡터로 응시점을 추정할 때, 두 직선이 정확히 만나지 않는 경우까지 포함해 최근접점을 구하는 식과 구현 시 주의점
---

양쪽 눈의 위치와 시선 방향을 3D로 추정하면, 두 시선이 만나는 점이 곧 **응시점**(그리고 그 점까지의 거리, 즉 수렴 거리)이 됩니다. 문제는 추정값에 잡음이 있어 두 직선이 **거의 항상 정확히 만나지 않는다**는 것입니다. 3D에서 평행하지도 만나지도 않는 두 직선, 즉 꼬인 위치의 두 직선에서 서로 가장 가까운 두 점을 구하고 그 중점을 응시점으로 씁니다.

## 설정

- 왼쪽 눈 위치 $P_a$, 시선 방향 $V_a$ / 오른쪽 눈 위치 $P_b$, 시선 방향 $V_b$
- 각 시선 위의 점: $C_a = P_a + tV_a$, $C_b = P_b + sV_b$
- $D = P_a - P_b$

## 최근접 조건

두 점을 잇는 벡터 $C_a - C_b$가 **두 방향 벡터 모두와 수직**일 때 거리가 가장 짧습니다.

$$
(C_a - C_b)\cdot V_a = 0,\qquad (C_a - C_b)\cdot V_b = 0
$$

대입해 정리하면 $t, s$에 대한 2×2 연립방정식이 됩니다.

$$
a = V_a\cdot V_a,\quad b = V_a\cdot V_b,\quad c = V_b\cdot V_b,\quad d_a = D\cdot V_a,\quad d_b = D\cdot V_b
$$

$$
\begin{bmatrix} a & -b \\ b & -c \end{bmatrix}
\begin{bmatrix} t \\ s \end{bmatrix}
=
\begin{bmatrix} -d_a \\ -d_b \end{bmatrix}
$$

행렬식 $\Delta = b^2 - ac$이고,

$$
t = \frac{b\,d_b - c\,d_a}{ac - b^2},\qquad s = \frac{a\,d_b - b\,d_a}{ac - b^2}
$$

응시점은 $\frac{C_a + C_b}{2}$, 두 점 사이 거리 $\lVert C_a - C_b\rVert$는 **두 시선이 얼마나 어긋났는지**를 나타내는 품질 지표로 쓸 수 있습니다. 두 직선이 실제로 만나면 이 거리가 0이 되고, 같은 식이 교점을 줍니다.

```python
import numpy as np

def gaze_closest_points(Pa, Va, Pb, Vb, eps=1e-9):
    D = Pa - Pb
    a, b, c = Va @ Va, Va @ Vb, Vb @ Vb
    da, db = D @ Va, D @ Vb
    den = a * c - b * b
    if den < eps:                      # 거의 평행: 응시점이 매우 멀다
        return None
    t = (b * db - c * da) / den
    s = (a * db - b * da) / den
    Ca, Cb = Pa + t * Va, Pb + s * Vb
    return (Ca + Cb) / 2, np.linalg.norm(Ca - Cb), t, s
```

## 구현 시 주의점

- **거의 평행하면 불안정합니다.** 먼 곳을 볼수록 두 시선은 평행에 가까워지고 $ac - b^2 \to 0$이 됩니다. 이때는 응시점 대신 "먼 곳을 본다"로 처리하거나 방향만 씁니다
- **$t, s$의 부호를 확인합니다.** 음수면 최근접점이 눈 뒤쪽에 있다는 뜻으로, 시선 추정이 잘못됐거나 두 시선이 발산하는 경우입니다
- **단위 벡터로 정규화**하면 $a = c = 1$이 되어 식이 단순해지고 수치도 안정적입니다
- **좌표계를 통일합니다.** 눈 위치와 시선 방향이 같은 좌표계(카메라 기준 등)에 있어야 하며, 화면 위의 점이 필요하면 응시점 대신 시선 직선과 화면 평면의 교점을 구하는 편이 낫습니다
- 잡음이 크면 프레임마다 구한 응시점을 그대로 쓰지 말고, 어긋남 거리로 가중하거나 시간 필터를 거칩니다([실시간 포인터 스무딩](/blog/smoothing-realtime-tracking-pointer/))

## 정리

- 두 시선은 잡음 때문에 거의 만나지 않으므로 교점이 아니라 **최근접점의 중점**을 쓴다
- 최근접 조건 = 연결 벡터가 두 방향과 모두 수직 → 2×2 연립방정식
- 평행에 가까운 경우와 음수 파라미터를 예외로 처리한다
