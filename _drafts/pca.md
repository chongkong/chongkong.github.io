---
layout: post
title: Principal Component Analysis
categories: machine-learning
tags: math, linear-algebra, pca
use_mathjax: true
---
> 최근 PCA를 공부하고 있었는데 저처럼 고등학생 수준의 수학 지식만을 가진 사람에게 수식을 처음부터 끝까지 잘 설명해주는
> 글은 생각보다 찾기 어려워서, 10/22 하루동안 공부한 내용들을 정리해서 한 번 써 보게 되었습니다. 

PCA는 $n$차원 공간의 데이터를 $k$개의 basis로 재구성하는 방법입니다. 이때 적은 차원으로도 최대한 데이터의 분포를 잘 표현할 수 있어야 하는데,
PCA는 데이터의 분산이 가장 커지는 벡터들을 순서대로 하나씩 고르는 방법입니다. 정확히는 다음 과정을 반복하는 알고리즘입니다.

1. 길이가 1인 벡터들 중 데이터들의 분산이 최대화되는 방향인 벡터 $u_{1}$을 고릅니다.
2. $u_{1}$과 수직하면서 길이가 1인 벡터들 중에서 데이터들의 분산이 최대화되는 방향의 벡터 $u_{2}$를 고릅니다.
3. $u_{1}, u_{2}$와 수직하면서 길이가 1인 벡터들 중에서 데이터들의 분산이 최대화되는 방향의 벡터 $u_{3}$을 고릅니다.
4. ...

위 과정을 계속해 나가면 서로 수직이면서 길이가 1인 $k$개의 벡터 $u_{1}, u_{2}, ..., u_{k}$를 구할 수 있습니다. 이렇게 고른
$k$개의 basis를 principal component라고 합니다. 결국 PCA는 데이터의 분산을 최대화시키는 greedy한 방법입니다.
이를 몇몇 선형대수의 개념을 이용하면 알고리즘이 아닌 단순 계산으로도 PCA의 결과와 같은 $k$개의 basis를 구할 수 있습니다.

$$
\frac{1}{m}X^{T}X = U\Lambda V^{-1}
$$

지금부터 위 식이 의미하는 바가 무엇인지 하나씩 살펴보도록 하겠습니다.

## Definitions

먼저 변수들을 정의해 보도록 합시다. 먼저 주어진 데이터의 개수는 $m$개이고 각각의 데이터를 $x^{(1)}, x^{(2)}, ..., x^{(m)}$ 라고 하겠습니다.
각 데이터는 $n$차원 벡터이고 이들을 하나의 $m \times n$ 행렬 $X$로 다음과 같이 표시하도록 합시다.

$$
X =
\begin{bmatrix}
    x^{(1)} \\
    x^{(2)} \\
    \vdots  \\
    x^{(m)} 
\end{bmatrix}
=
\begin{bmatrix}
    x^{(1)}_1 & x^{(1)}_2 & \ldots & x^{(1)}_n \\
    x^{(2)}_1 & x^{(2)}_2 & \ldots & x^{(2)}_n \\
    \vdots    &           & \ddots & \vdots    \\
    x^{(m)}_1 & x^{(m)}_2 & \ldots & x^{(m)}_n

\end{bmatrix}
$$

마지막으로 이 데이터들을 이동시킬 $k$개의 basis들을 $u_{1}, u_{2}, ..., u_{k}$ 라고 합시다.

## 2D to 1D

문제를 상상하기 쉽도록 2차원 데이터를 1차원 데이터로 이동시키는 경우를 생각합시다. 즉 $n=2, k=1$인 경우입니다.


## Covariance Matrix

그 전에 먼저 Covariance Matrix에 대해 알아봅시다. 

## Eigenvalue and Eigenvectors

## Eigendecomposition

## Singular Value Decomposition

## SVD of Covariance Matrix

## Rayleigh Quotient

------
참고 자료:


- _Jon Shlens_, [A Tutorial on Principal Component Analysis](https://www.cs.princeton.edu/picasso/mats/PCA-Tutorial-Intuition_jp.pdf), 2003
- _다크 프로그래머_, [주성분분석(PCA)의 이해와 활용](http://darkpgmr.tistory.com/110), 2013
- _Vincent Spruyt_, [Geometric Interpretation of Covariance Matrix](http://www.visiondummy.com/2014/04/geometric-interpretation-covariance-matrix/), 2014
