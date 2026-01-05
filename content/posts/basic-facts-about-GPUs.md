---
title: "GPU에 관한 기본적인 사실들"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2025-07-04"
summary: "[basic facts about gpus](https://damek.github.io/random/basic-facts-about-gpus/)를 번역한 글입니다"
description: ""
toc: true
readTime: true
autonumber: true
math: true
tags: ["Graphics", "GPU", "AI"]
showTags: false
hideBackToTop: false
---

* 이 포스트는 [해당 글](https://damek.github.io/random/basic-facts-about-gpus/)을 번역해 옮겼습니다. 번역 과정에서 일부 의역을 포함했습니다.

저는 GPU의 동작에 대해 보다 나은 이해를 하기 위해 노력했습니다. 인터넷 상의 매우 많은 자료들을 읽었고, 아래의 포스트들이 특히 도움이 되었습니다.

1. [Making Deep Learning Go Brrrr From First Principles](https://horace.io/brrr_intro.html)
2. [What Shapes Do Matrix Multiplications Like?](https://www.thonking.ai/p/what-shapes-do-matrix-multiplications)
3. [How to Optimize a CUDA Matmul Kernel for cuBLAS-like Performance: a Worklog](https://siboehm.com/articles/22/CUDA-MMM)

이 포스트는 이러한 리소스들로부터 배운 다양한 사실들을 정리한 것입니다.

- [Hacker news discussion](https://news.ycombinator.com/item?id=44365320)
- [Twitter thread](https://x.com/damekdavis/status/1936496421743546488)

**Acknowledgements:** Thanks to [Alex McKinney](https://afmck.in/) for comments on [independent thread scheduling](https://docs.nvidia.com/cuda/ampere-compatibility-guide/#independent-thread-scheduling-compatibility).

---

## 연산 장치(Compute)와 메모리 계층

(역주. 여기서 "연산" 혹은 ''연산 장치'' 라고 번역한 Compute는 "Compute Shader"의 Compute와 동일한 의미입니다.)

GPU는 GPU의 메인 메모리에 접근하는 시간보다 연산 장치가 연산을 수행하는 시간이 훨씬 빠르게 설계되었기 때문에 둘 간의 불균형을 초래합니다. NVIDA A100을 예로 들면, 32비트 부동소수 연산을 초당 19.5 트릴리온만큼 수행(19.5 TFLOPs)할 수 있지만, 메모리 대역폭은 초당 겨우 1.5테라바이트(TB) 밖에 되지 않습니다. 따라서 GPU는 메모리에서 4Byte만큼의 정수를 하나 읽는 동안 50번 이상의 계산을 수행할 수 있습니다.

아래는 NVIDIA A100 GPU의 연산 장치와 메모리 구조를 나타내는 다이어그램입니다. 여기 인용된 수치들(flops/s, TB/s)는 오직 A100에만 유효하다는 것에 주의하세요.

```text
+---------------------------------------------------------------------------------+
|                               Global Memory (VRAM)                              |
|                            (~40 GB, ~1.5 TB/s on A100)                          |
+----------------------------------------+----------------------------------------+
                                         | (Slow off-chip bus)
+----------------------------------------v----------------------------------------+
|                            Streaming Multiprocessor (SM)                        |
|                     (1 of 108 SMs on an A100, each ~(19.5/108) TFLOPS)          |
|                           (2048 threads, 64 warps, 32 blocks)                   |
| +-----------------------------------------------------------------------------+ |
| |                        Shared Memory (SRAM) / L1 Cache                        |
| |                    (~192 KB on-chip workbench, 19.5 TB/s)                     |
| +-----------------------------------------------------------------------------+ |
| |                        Register File (~256 KB, ? TB/s)                        |
| +-----------------------------------------------------------------------------+ |
| |                                                                             | |
| |                //-- A "Block" of threads runs on one SM --//                | |
| | +--------------------------+ +------------------------+                     | |
| | |      Warp 0 (32 thr)     | |      Warp 1 (32 thr)   | ... (up to 32 warps)| |
| | | +----------------------+ | +----------------------+ |                     | |
| | | | Thread 0 Registers   | | | Thread 32 Registers  | |                     | |
| | | | [reg0: float]        | | | [reg0: float]        | |                     | |
| | | | [reg1: float] ...    | | | [reg1: float] ...    | |                     | |
| | | +----------------------+ | +----------------------+ |                     | |
| | +--------------------------+ +------------------------+                     | |
| |                                                                             | |
+---------------------------------------------------------------------------------+

```

이 다이어그램은 GPU성능 측면에서의 계층을 보여줍니다. **전역 메모리(VRAM)** 는 모든 데이터들이 기본적으로 존재하게 되는 크고 느린 오프칩(Off-chip) 메모리 풀입니다. **스트리밍 멀티프로세서(Streaming Multiprocessor, SM)** 는 GPU의 연산을 수행하는 기본 단위를 의미합니다. 이 SM은 느린 버스(bus)를 통해 메모리로부터 데이터를 가져와야만 연산 작업을 수행할 수 있습니다. 이를 완화하기 위해 각 SM은 19.5TB/s의 대역폭을 가지는 빠른 온칩(On-chip) 공유 메모리(Shared Memory, SRAM)를 보유합니다. 프로그래머는 이것을 수동적으로 관리하는 캐시처럼 활용할 수 있습니다.

**스레드(Thread)** 는 가장 작은 실행 단위입니다. 각각의 스레드는 즉각적인 계산을 위해 값들을 저장할 수 있는 자신만의 레지스터 세트를 가집니다. 스레드의 레지스터 접근 속도는 ??TB/s 입니다(역주. 엔비디아에서 해당 수치를 공식적으로 밝히지 않기에 원글에도 동일하게 물음표로 표현되어 있습니다). 하드웨어 관점에서 32개의 스레드는 1개의 **워프(Warps)** 로 구성됩니다. 한 워프에 존재하는 모든(32개) 스레드는 동일한 시점에 동일한 계산을 실행합니다. A100에서, 한 개의 SM은 최대 64개의 워프를 가지고 있습니다. 프로그래머는 스레드들을 한 블록(Block)단위로 그룹화할 수 있는데, 이 때 블록은 한 SM에서 동시에 동작하는 것이 보장되는 스레드의 그리드라 볼 수 있습니다. 블록은 1, 2, 혹은 3차원이 될 수 있습니다. 간단히 표현하기 위해, 이 포스트는 총 개수가 하드웨어의 한계인 1024개의 크기를 초과하지 않는 2차원 블록(BLOCK_DIM * BLOCK_DIM) 형태의 스레드에 집중할 것입니다. 블록 내의 모든 스레드들은 동일한 온칩 공유 메모리에 접근할 수 있습니다.

## 두 개의 성능 지표

우리는 다수의 GPU스레드를 병렬로 실행하기 위해 호스트(CPU)에서 호출하는 함수인 **커널(kernel)** 의 성능을 분석합니다.커널의 성능은 **메모리 대역폭(Bandwidth)** 혹은 **연산 처리율(Throughput)** 에 의해 제한됩니다. 이러한 두 제한은 성능 지표를 정의합니다.

작업이 전역 메모리에서 SM으로 데이터를 전송하는 속도에 따라 런타임(역주. 런타임*Run-time*은 보통 '실행 중인 상태'의 의미로 사용되지만 이 글에서는 말 그대로 '실행 시간'의 의미로 사용하겠습니다)이 결정되는 경우 이 작업은 **메모리 종속적(Memory-bound)** 인 작업입니다. 대표적으로 원소(element) 단위 덧셈(ex. $y = x + 1$) 연산과 같은 경우에 SM은 매우 적은 수의 플롭스를 각 요소의 값을 읽는데 사용합니다. 그리고 이 때 SM은 데이터 읽기가 완료되기를 기다리며 대부분의 시간을 유휴(idle)상태로 보내게 됩니다.

반면 작업이 SM들의 연산 속도에 따라 런타임이 결정되는 경우 이 작업은 **연산 종속적(Compute-Bound)** 인 작업입니다. 거대한 행렬 곱셈이 전형적인 예시입니다. 한번 데이터가 SM에 불러와지면(Load), 대량의 연산이 수행됩니다. 마찬가지로 이 때 메모리 버스는 SM이 바쁠 동안 유휴 상태가 됩니다.

상술한 두 가지 성능 지표를 측정하는 공식적인 방법이 바로 **산술 강도(Arithmetic Intensity)** 이며, **작업량과 메모리 트래픽의 비율**로 정의됩니다.

```
Arithmetic Intensity(AI) = Total FLOPs / Total Bytes Accessed
```

[루프라인 모델](https://en.wikipedia.org/wiki/Roofline_model) 의 경우, 총 메모리 접근량(Total Bytes Accessed)은 전역 메모리(HBM)과 온칩 SM사이에 전송된 데이터를 구체적으로 계산합니다. 이것은 모델이 주요 병목 현상인 느린 오프칩 메모리 버스에 따른 커널의 성능을 평가하기 때문입니다. SRAM에서 레지스터로의 데이터 전송과 같은 온칩 트래픽은 이 계산에 포함되지 않습니다.

루프라인 모델은 커널의 달성 가능한 성능을 AI에 대해 그래프로 표시합니다. 두 개의 "루프(지붕 형태)"는 GPU의 물리적 한계를 나타냅니다.

```
  ^ Performance (TFLOPS)
  |                                        
  | Memory-Bound Region ¦ Compute-Bound Region
  |                     ¦
  |                    /¦----------------------  <-- Peak Compute (~19.5 TFLOPS)
  |                   / ¦
  |                  /  ¦
  | Peak Global     /<--¦------ Inefficient Compute Roof (e.g., using scalar ops, transcendental functions)
  | Mem BW (~1.5   /    ¦
  | TB/s)         /     ¦
  |              /      ¦
  +---------------------¦---------------------------> Arithmetic Intensity (FLOPs/Byte)
                        ^
                        ¦
                  Hardware Ridge Point (~13)

```

커널의 성능은 다음과 같이 결정됩니다.

* 메모리 종속적인 작업의 경우, SM은 데이터를 기다리며 멈추게 됩니다. 따라서 런타임은 데이터를 이동시키는 데 걸리는 시간이 됩니다:

  ```
  Runtime = Bytes_Accessed / Memory_Bandwidth
  ```

  그러므로 커널의 성능은 다음과 같습니다.

  ```
  Performance = Total_FLOPs / Runtime = AI * Memory_Bandwidth
  ```

  로그-로그($log$) 그래프에서 이는 대각선으로 나타납니다.

* 연산 종속적인 작업의 경우 SM은 최대로 활용됩니다. 성능은 SM들의 최대 산술 연산 처리율에 의해 제한됩니다.

  ``` 
  Performance = Peak_Compute_FLOPs
  ```

  동일한 그래프에서 이는 수평선으로 나타납니다.

**커널의 실질적인 성능은 이러한 두 값들 중 최솟값입니다.** 두 성능의 한계점이 교차하는 능선(ridge point) 형태의 지점이 바로 산술 강도(AI)를 나타냅니다. A100의 경우, AI는 19.5 TFLOPs / 1.5TB/s = 13FLOPs/Byte입니다. 커널이 연산 종속적인 작업이 되려면 반드시 이 AI 수치를 초과해야 합니다. 만약 (A100기준) 13 미만의 AI인 커널은 그래프 상에서 메모리 종속 영역에서 작동하며, 13 이상의 AI인 커널은 반대로 연산 종속 영역에서 작동하게 됩니다. 커널 최적화의 목표는 커널의 동작 지점을 (루프라인 모델 그래프 상에서) 오른쪽으로 이동시켜 연산 한계에 도달하기 위해 AI 수치를 향상시키는 것입니다.

19.5 TFLOPs의 "연산 정점"은 이상적인 지점이며, 오직 텐서 코어의 행렬 곱셈과 같이 **극한으로 최적화된 연산들**과 **충분히 높은 전력 제한**이 함께할 때 도달할 수 있습니다. 임의의 동작은 연산 종속 영역에 위치할 수 있지만 여전히 이러한 정점보다는 훨씬 낮은 수준에서 동작합니다. 예를 들어,스칼라 산술 연산 혹은 복잡한 초월 함수($sin$, $cos$ 등)에 의해 지배되는 높은 AI수치의 커널은 특정 느린 명령어의 처리량에 의해 성능의 제한을 받게 됩니다. 이는 위의 그래프에서 보이는 것 처럼 커널에 덜 효율적인 "루프"를 만듭니다. AI의 증가는 최적화의 측면에서 필수적이지만, 충분하지는 않습니다. 이 말은 즉, FLOPS 역시 효율적이어야 한다는 것입니다.

AI를 증가시키기 위한 주요한 전략은 온칩 메모리에서 SM에 적재된 데이터를 최대한 재사용하는 것입니다. 후술한 내용은 스레드가 데이터를 전역 메모리로부터 그들의 개인 레지스터로 옮길 때를 나타내는 단순화된 모델입니다. 이 분석은 앞선 상황에서 최소한으로 요구되는 데이터 전송량을 측정합니다; 실제 메모리 트래픽은 우리가 추후 논의할 **접근 방식(Access pattern)** 에 따라 달려 있습니다.

모든 행렬이 NxN이고 4바이트의 부동 소수점을 사용하는 가상의 연산 C=A@B를 생각해 봅시다.

**전략 1: 각 스레드가 각 원소 `C[i, j]`를 계산**

* **FLOPs**: `C[i, j]`를 계산하기 위해, 스레드는 N번의 곱셈-덧셈 연산을 수행합니다. 이는 2\*N FLOPs입니다.
* **Bytes Accessed**: 스레드는 A의 i행과 B의 j열을 읽어야만 하며, 이는 2\*N 개의 float이므로 바이트로는 8\*N bytes입니다.
* **Arithmetic Intensity**: (2\*N FLOPs) / (8\*N Bytes) = 0.25 FLOPs/Byte

이 AI 수치는 낮습니다. 이 전략에서 커널은 메모리 종속 영역에 위치할 것입니다.

**전략 2: 한 스레드가 C의 2x2타일을 계산**

2x2 타일(`C[i, j]`, `C[i, j+1]`, `C[i+1, j]`, `C[i+1, j+1]`)을 계산하기 위해 스레드는 모든 네 개의 원소에 대해 연산을 수행해야 합니다.

* **FLOPs**: 4 elements * 2 * N Flops/element = 8*N FLOPs
* **Bytes Accessed**: 스레드는 A로부터 두 개의 행(`A[i, :]`, `A[i+1, :]`)을 읽어야 하며 B로부터 두 개의 열(`B[:, j]`, `B[:, j+1]`)을 읽어야 하므로 이는 2\*N + 2\*N = 4\*N 개의 float이며, 16\*N 바이트입니다.
* **Arithmetic Intensity**: (8\*N FLOPs) / (16\*N Bytes) = 0.5 FLOPs/Byte

두 전략에서 도출된 AI 수치들은 A100의 ridge point인 ~13FLOPs/Byte에 비하면 한참 낮습니다. 이 간단한 레지스터 전용 모델은 행렬 곱셈을 연산 종속적인 동작으로 만들기엔 부족합니다.

높은 AI를 달성하기 위한 키 포인트는 블록 안의 스레드들이 협력해 훨씬 많은A와 B의 타일을 SRAM으로 불러오는 것입니다. 이 공유 메모리에서 모두가 함께 동작함으로써, 블록 내의 1024개 스레드들은 AI를 13보다 더 높은 수치로 만들 수 있습니다. 우리는 이 메커니즘의 자세한 사항을 **공유 메모리** 섹션에서 다룰 것입니다.

## 세 번째 지표: 오버헤드

성능은 또한 **호스트 쪽의 오버헤드에**도 영향을 받습니다. 이것은 CPU가 GPU에 요청할 작업을 준비하기 위해 소비하는 시간이며, 예를 들자면 파이썬의 인터프리터 혹은 프레임워크의 디스패치 시스템 등이 있을 것입니다.

어플리케이션은 GPU 커널이 매우 적거나(작거나) 많을 때(클 때) 오버헤드가 발생합니다. GPU는 각각의 작은 일(Task)을 빠르게 실행한 후 CPU가 다음 명령을 요청할 때 까지 대기하거나 잠들게 됩니다. 이 때의 런타임은 얼마나 CPU가 GPU에 충분히 빠르게 데이터(와 명령)을 공급하지 못하는지에 따라 결정됩니다.

현대적인 프레임워크들은 이를 완화하기 위해 비동기 실행을 사용합니다. 호스트는 각 명령이 완료될 때 까지 기다리지 않고 GPU에 대한 명령 스트림을 큐에 넣을 수 있습니다. 만약 각각의 GPU 동작들이 충분히 크다면, 호스트는 "먼저 실행"할 수 있으며, 한 커널을 실행하는 오버헤드는 이전 커널을 실행함으로써 숨겨집니다(역주. 커널이 실행될 때 발생하는 호스트의 대기 등이 이전부터 실행 중이던 GPU 동작에 의해 감춰지기 때문에 사용자가 "인지"하지 못하게 된다는 의미입니다).

이 포스트의 나머지 부분에서 우리는 메모리와 연산에 집중하기 위해 오버헤드가 주요한 제한 요소가 아닐 만큼 커널이 충분히 크다고 가정할 것입니다.

## 성능 향상을 위한 두 가지 기본 전략: 융합(Fusion)과 타일링(Tiling)

상술했듯이 오버헤드가 무시될 정도로 충분히 큰 커널이 실행될 때 성능은 두 가지 물리적인 제한에 의해 지배됩니다. 바로 메모리 대역폭과 연산 처리율입니다. 그러므로 커널의 성능을 증가시킨다는 것은 루프라인 모델에서 이러한 동작들의 지점을 위로, 그리고 오른쪽으로 밀어낸다는 것을 의미합니다. 이를 달성하기 위한 기본적이 두 가지 전략이 존재하며, 다음과 같습니다.

* 개별적으로는 메모리 종속적인 일련의 작업들에 대한 전략은 이를 단일 커널로 **융합(Fuse)** 하여 **중간 메모리 트래픽을 제거**하는 것입니다.

* 높은 산술 강도를 수반하는 복잡한 단일 동작의 경우, **타일링(Tiling)** 을 통해 SM의 빠른 메모리 내에서 **데이터 재사용을 최소화**하는 것입니다.

각 전략을 차례대로 살펴보겠습니다.

### 연산자 융합(Operator Fusion)

`y = relu(x + 1)`과 같은 간단한 연산들이 결합(Chain)된 형태의 수식은 매우 흔하게 나타납니다. 각각의 연산(`add`, `relu`)은 매우 낮은 산술 강도를 가짐과 동시에 메모리 종속적입니다. 이러한 시퀀스를 개별적인, 동시에 연속적인 GPU 커널들로 수행하는 것은 비효율적입니다. 이를 최적화하는 주요 전략은 **연산자 융합(Fusion)** 입니다.

문제는 중간 메모리 트래픽입니다. 융합되지 않은 연산 `y = relu(x + 1)`을 고려해 봅시다:

1. **Kernel 1 (add):** 전역 메모리에서 텐서 x의 전체 데이터를 읽습니다. `tmp = x + 1`을 계산합니다. 중간 텐서인 `tmp`의 전체 데이터를 다시 전역 메모리에 씁니다(Write). 

2. **Kernel 2 (relu):** 텐서 `tmp`의 데이터 전체를 전역 메모리에서 읽습니다. `y = relu(tmp)`를 계산합니다. 최종 텐서 `y`를 전역 메모리에 씁니다.

이는 낭비가 매우 심한 접근입니다. 이는 두 개의 커널 실행 오버헤드와 중간 텐서 `tmp`의 **전역 메모리로의 왕복이 강제**됩니다.

융합된 커널에선:

1. 각 스레드가 x의 원소를 전역 메모리에서 자신의 레지스터로 읽어 가져옵니다.
2. 모든 연산을 수행합니다. 즉, `tmp = x + 1` 을 수행한 후 `y = relu(tmp)` 를 수행한다는 뜻입니다. 이들은 모두 빠른 레지스터에서 수행됩니다.

3. 최종 결과인 y를 전역 메모리에 씁니다.

```python
# Unfused (Conceptual)
def unfused_add_relu(x):
    tmp = torch.add(x, 1) # Reads x from HBM, writes tmp to HBM
    y = torch.relu(tmp)   # Reads tmp from HBM, writes y to HBM
    return y

# Fused (Conceptual)
@torch.compile
def fused_add_relu(x):
    # The compiler fuses these into one kernel.
    # The intermediate result of x+1 never touches HBM.
    return torch.relu(x + 1)
```

중간 텐서인 `tmp`는 임시 변수로 활용되며 절대 전역 메모리에 쓰이지 않습니다. 이는 메모리 트래픽을 반토막낼 뿐만 아니라(이전과 달리 x를 한 번 읽고, y를 한 번 쓸 뿐이므로) 두 번째 커널을 실행하며 발생할 오버헤드 역시 제거합니다.

### 타일링(Tiling): 연산 종속적인 커널을 위한 전략

우리의 레지스터 전용 모델인 `C=A@B` 는 0.25FLOPs/Byte의 산술 강도를 도출합니다. A100의 ridge point인 ~13에 비하면 한참 낮습니다. 왜냐하면 각 스레드가 $2N$ FLOPs만큼 일하기 위해 $2N$만큼 읽기 작업을 수행하기 때문입니다. 이 데이터는 한 번 사용되고 나면 버려집니다. 데이터 재사용을 증가시키고 작업을 연산 종속 영역에 위치시키기 위해, 한 블록 안의 스레드들은 거대한 인풋 행렬을 SM의 빠른 온칩 공유 메모리로 옮기기 위해 협력해야 합니다.

이러한 협력 로직은 행렬 곱을 분해(Decomposing)하는 것에 기반합니다. 개별 원소 `C[i, j]`에 대한 계산은 차원 K에 대한 다음과 같은 합연산과 동일합니다:

```
C[i, j] = sum_k A[i, k] B[k. j]
```

이 결과는 타일에 대한 부분합의 총합으로 분할될 수 있습니다. 정방형 타일의 경우, 내부의 k차원은 외부 차원과의 매칭을 위해 `BLOCK_DIM`크기의 타일들로 분해됩니다. 이 공식은 다음과 같습니다:
{{<rawhtml>}}
\[
\begin{aligned}
C[i,j] = \sum_{t=0}^{\text{NUM\_K\_TILES}-1} \left( \sum_{k=t \cdot \text{BLOCK\_DIM}}^{(t+1) \cdot \text{BLOCK\_DIM} - 1} A[i,k] B[k,j] \right)
\end{aligned}
\]
{{</rawhtml>}}
이 타일링 알고리즘은 매 반복마다 바깥쪽 합(부분곱 하나)에서 한 개의 항을 계산합니다. 블록의 스레드들은 각각 A와 B의 타일을 불러오고, 칩 안에서 그들의 곱을 계산한 후에 그 결과를 누적하는 것을 k차원에 대해 반복해 최종적으로 하나의 결과인 `C_tile`을 얻어냅니다. 이는 **불러오기(Load)**, **동기화(Synchronize)**, 그리고 **연산(Compute)** 이라는 3단계의 패턴으로 이루어집니다.

```python
# Conceptual algorithm for one thread block computing one output tile, C_tile.
# C_tile corresponds to, e.g., C[block_row_start:end, block_col_start:end].

# Each thread in the block holds a piece of C_tile in its registers. Initialize to zero.
thread_private_C_accumulator = zeros(...)

# Loop over tiles of A and B along the k-dimension.
# Each iteration computes one partial product from the sum above.
for k_tile_idx in range(NUM_K_TILES):
    # Phase 1: Load
    # All threads in the block cooperate to load one tile of A and one tile of B
    # from slow Global Memory into fast Shared Memory.
    A_tile = load_A_tile_from_global_mem(k_tile_idx)
    B_tile = load_B_tile_from_global_mem(k_tile_idx)

    # Phase 2: Synchronize
    # Wait for all threads to finish loading before any thread starts computing.
    # This ensures A_tile and B_tile are fully populated.
    __syncthreads()

    # Phase 3: Compute
    # Each thread computes its piece of the on-chip matmul.
    # The data in A_tile and B_tile is reused extensively from Shared Memory.
    thread_private_C_accumulator += on_chip_matmul_piece(A_tile, B_tile)

    # Wait for all threads to finish computing before loading the next tile.
    __syncthreads()

# After the loop, write the final accumulated result to Global Memory.
write_C_tile_to_global_mem(thread_private_C_accumulator)
```

이제 불러오기, 동기화, 연산이라는 3단계 계산 패턴의 동작 원리를 살펴봅시다.

### 집약된 불러오기: HBM에서 SRAM으로

첫 단계는 A와 B의 타일을 느린 전역 메모리(HBM)에서 빠른 온칩 공유 메모리(SRAM)으로 불러오는 것입니다. 이 단계의 목표는 이러한 데이터 전송을 가능한 한 최대의 메모리 대역폭을 이용해 수행하는 것입니다. 이는 **집약된 메모리 접근(Coalesced memory access)** 을 요구합니다. 메모리 접근은 워프 내의 32개 스레드가 한 번의 트랜잭션으로 HBM에서 연속적인 128바이트 단일 블록에 접근할 때 통합됩니다.

이를 달성하기 위해, 커널은 스레드 인덱스들을 메모리 주소로 사상합니다. `BLOCK_DIM` * `BLOCK_DIM` 크기의 블록의 스레드들이 동일한 크기의 데이터 타일을 불러오기 위해, 일반적으로 스레드 `(tx, ty)`에 대해 `A[global_row + ty, global_k + tx]` 를 `A_tile[ty, tx]` 로 공유 메모리에 불러오도록 사상합니다. 이 예시에서, `BLOCK_DIM`은 32입니다.

한 워프의 스레드들이 `ty`가 고정되고 `tx`의 범위가 0에서 31로 정의되었다고 가정해 봅시다.

* 스레드 `(0, ty)`는 `A[global_row + ty, global_k + 0]`을 읽습니다.
* 스레드 `(1, ty)`는 `A[global_row + ty, global_k + 1]`을 읽습니다.

* ...
* 스레드 `(31, ty)`는 `A[global_row + ty, global_k + 31]`을 읽습니다.

Row-major(역주. Row-major는 NxN행렬이 저장될 때 메모리에 *m_00, m_01, ..., m_0x...m_10, m_11, ...* 처럼 열의 방향을 따라 저장하는 방식입니다) 저장이라고 가정하면, 이러한 스레드들은 32개의 연속적인 4바이트 float들, 즉, 연속적인 128바이트의 세그먼트에 접근하게 됩니다. 이는 완벽히 집약된 읽기를 수행합니다. 전체 32x32타일이 블록 내의 각 워프에 대해 하나씩, 총 32번의 이와 같은 집약된 읽기에 의해 불러와집니다.

```
Thread Block (32x32)          Global Memory (HBM)
                              (One row of A's tile)
+--------------------+
| Warp 0 (ty=0)      | ----> [A_ij, A_i,j+1, ..., A_i,j+31]  (128 bytes)
| (tx = 0..31)       |       (One coalesced memory transaction)
+--------------------+
| Warp 1 (ty=1)      | ----> [A_i+1,j, ..., A_i+1,j+31] (128 bytes)
+--------------------+
| ...                |
+--------------------+
| Warp 31 (ty=31)    | ----> [A_i+31,j, ..., A_i+31,j+31] (128 bytes)
+--------------------+
```

이러한 불러오기는 **벡터화된(Vectorized) 접근**을 이용하면 더욱 효율적이게 됩니다. 집약된 읽기를 위한 물리 메모리 트랜잭션은 HBM에서 128바이트 데이터 전체를 가져옵니다. 차이는 어떻게 SM이 이 데이터를 요청하느냐입니다.

스칼라 불러오기의 경우, 워프는 반드시 32개의 분할된 32비트 불러오기 명령어를 실행해야 합니다. 반면 벡터화된 불러오기의 경우, 워프는 128비트 불러오기 명령어를 8번 실행하면 됩니다. SM은 클록 사이클마다 제한된 수의 명령 요청 슬롯을 가지고 있기 때문에 후자의 경우가 훨씬 효율적입니다. 8번의 넓은(역주. 연산 대상이 되는 메모리의 너비를 의미합니다) 명령어는 32번의 좁은 명령어보다 더 적은 하드웨어 자원을 소모합니다. 이는 메모리 컨트롤러가 최대 너비 요청의 연속적인 스트림을 위해 바쁜 상태를 유지하게 되고, SM측의 병목 현상이 줄어들어 활용되는 메모리 대역폭이 늘어나는 결과를 제공합니다.

벡터화된 접근은 메모리가 벡터 사이즈에 맞게 정렬되도록 약속했다는 가정하에 코드 내에서 포인터 형변환을 통해 가능합니다(예를 들면, float* 를 float4*로).

이러한 벡터화된 불러오기의 효율은 **메모리 정렬(Memory Alignment)** 에 달려 있습니다. 단일의 float4 연산은 16바이트 벡터를 불러옵니다. 4바이트 float으로 구성된 행렬의 경우, 이 벡터는 4개의 원소를 포함할 수 있습니다. 하드웨어는 오직 해당 벡터의 메모리 주소가 16의 배수인 경우에만 이 명령어를 효율적으로 수행합니다. 이는 행렬의 내부 차원 K(열의 크기)가 반드시 4의 배수가 되어야 한다는 의미입니다. 만약 K가 4의 배수가 아니라면, 행은 16바이트 메모리 세그먼트에 맞게 정렬되지 않게 됩니다.

4바이트 float으로 구성된 행렬과 16바이트 세그먼트의 메모리 시스템을 가정해봅시다.

* **정렬된 경우(K=8, 즉 4의 배수)**

  ```
  Memory: |<--- 16B --->|<--- 16B --->|
          [Seg 0       ][Seg 1       ]
  Row 0:  [e0 e1 e2 e3 | e4 e5 e6 e7]  (A float4 load for e0-e3 is aligned)
  Row 1:  [e0 e1 e2 e3 | e4 e5 e6 e7]  (A float4 load for e0-e3 is aligned)
  ```

* **정렬되지 않은 경우(K=7)**

  ```
  Memory: |<--- 16B --->|<--- 16B --->|<--- 16B --->|
          [Seg 0       ][Seg 1       ][Seg 2       ]
  Row 0:  [e0 e1 e2 e3 e4 e5 e6]
  Row 1:                      [e0 e1 e2 e3 e4 e5 e6] (A float4 load for Row 1's e0-e3 spans Seg 0 and Seg 1)
  ```

이러한 오정렬(Misalignment)은 하드웨어가 더 복잡하고 느린 불러오기 연산을 수행하도록 강제하고, 메모리 대역폭의 감소를 일으킵니다.

여기서 **주의**할 점은 이러한 row-wise전략은 행렬 A에 대해선 통합된 접근을 제공하지만, 행렬 B에 대해선 필요한 접근 패턴이 반대가 된다는 것입니다.

1. **HBM 요구사항**: 통합된 접근을 유지하기 위해 타일 B는 반드시 HBM으로부터 row-by-row로 읽어져야 합니다.
2. **연산 요구사항**: 행렬 곱은 그 자체로 타일 B의 열 접근을 요구합니다.

row-major 행렬에서 열을 그대로 불러오는 것은 HBM 트랜잭션을 직렬화(Serialize)하는 집약되지 않고 띄어진(strided) 접근입니다. 따라서 이에 대한 해결책은 타일 B를 열 단위의 집약된 읽기로 불러오되, SRAM에 모두 불러오고 나면 **원소들을 재배열**하는 것입니다. 이러한 재배열의 구조는 SRAM의 물리적인 설계, 혹은 저장 방식의 설계에 의해 결정됩니다.

### 동기화

`__syncthreads()` 호출은 배리어의 역할을 합니다. 블록 내의 어떠한 스레드도 다른 모든 스레드가 이 지점에 도달할 때 까지 더 이상의 연산을 진행할 수 없습니다. 이는 `A_tile`과 `B_tile`이 연산 단계로 진입하기 전에 온전히 공유 메모리에 불러와졌음을 보장합니다.

### 온칩 하드웨어: Bank와 Warp

공유 메모리는 SM에 위치한 물리적인 리소스입니다. 스레드 블록이 한 SM에서 실행되도록 스케줄링되었을 때, 공유 메모리의 일부가 해당 블록의 독점적인 사용을 위해 할당됩니다.

공유 메모리는 물리적으로 동일한 크기의 32개의 독립적인 메모리 모듈로 분할되어 있으며, 이를 **뱅크(Banks)** 라고 부릅니다. 이러한 뱅크들은 병렬적으로 요구된 메모리를 제공할 수 있습니다. 눈치채셨겠지만 32개라라는 수는 절대 임의로 정해진 게 아닙니다. 바로 워프의 크기이죠. 워프는 잠금 단계에서 명령을 실행하는 32개의 스레드로 구성되며, 메모리 접근의 기본 단위임을 기억하세요. 32개의 뱅크는 한 클록 사이클 안에서 병렬적으로 단일 워프로부터 32개의 메모리 요구에 대응할 수 있도록 설계되어 있으며, 이 때 각 요청들은 서로 다른 뱅크를 대상으로 이루어집니다.

4바이트의 워드(word)로 표현되는 주소는 뱅크 전체에 걸쳐 삽입됩니다.

```
bank 0:  [word 0, word 32, word 64, ...]
bank 1:  [word 1, word 33, word 65, ...]
...
bank 31: [word 31, word 63, word 95, ...]
```

주어진 워드 크기 주소에 어떤 뱅크가 대응되는지는 (여느 주소 사상 방식들과 비슷하게) `address % 32`로 결정됩니다.

### 뱅크 충돌 문제

공유 메모리의 최대 대역폭을 사용하기 위해, 워프 내 32개 스레드들은 32개의 서로 다른 뱅크와 지정될 수 있는 워드 주소에 접근해야 합니다. **뱅크 충돌(Bank Conflict)** 은 다수의 스레드가 서로 다른 주소에 접근하지만 그 주소들이 같은 뱅크에 사상되어 있을 때 발생합니다. 하드웨어는 이를 요청을 직렬화함으로써 해결할 수 있지만 이 때 대역폭이 감소됩니다. 모든 스레드가 동일한 주소를 읽는 **브로드캐스트(Broadcast)** 는 빠르고 충돌이 없는 작업입니다.

이는 행렬 곱에서 문제를 발생시킵니다. `BLOCK_DIM` * `BLOCK_DIM` 크기 타일이 공유 메모리에 row-major 레이아웃으로 저장되어 있다고 가정해봅시다. `BLOCK_DIM`의 크기가 32일 때, `tile[row, col]`의 주소는 `row * 32 + col`이 됩니다.

* **행 접근 (Row Access, A_tile):** 워프는 각 `t = 0 ... 31 `에 대해 `A_tile[fixed_row, t]` 에 접근합니다. 이 때 주소는 `fixed_row * 32 + t`가 됩니다. 각 스레드의 뱅크는 `(fixed_row * 32 + t) % 32 = t % 32`입니다. 각 스레드마다 `t`가 모두 다르기 때문에, 스레드는 32개의 서로 다른 뱅크에 접근합니다. 이는 충돌이 없고 최대 대역폭을 사용하는 접근입니다.
* **열 접근 (Column Access, B_tile):** 워프는 각 `t = 0 ... 31`에 대해 `B_tile[t, fixed_col]`에 접근합니다. 이때 주소는 `t * 32 + fixed_col`이 됩니다. 각 스레드의 뱅크는 `(t * 32 + fixed_col) % 32 = fixed_col % 32`입니다. **모든 32개의 스레드가 같은 뱅크를 목표로 합니다.** 이는 **32-way 뱅크 충돌**을 야기하며, 메모리 접근이 직렬화됩니다.

해결책은 `B_tile`을 전치(transposed)시켜 공유 메모리에 저장하는 것입니다.

```
# Action for thread (tx, ty) during the load phase
# A is loaded directly, B is loaded and transposed on-the-fly
A_tile[ty, tx] = A_global[global_row + ty, global_k + tx]
B_tile[tx, ty] = B_global[global_k + ty, global_j + tx] # Indices are swapped
```

이러한 "불러와서 전치하기(load-and-transpose)" 행동은 칩 내부 계산을 변경합니다. 각 원소에 대한 부분곱 계산은 더이상 `A_tile`의 행과 `B_tile`의 열의 내적으로 표현되지 못하며, 대신에 전치된 칩 안의 `B_tile`에 대해 다음과 같은 공식이 성립합니다.
{{<rawhtml>}}
\[
\begin{aligned}
C_{\text{partial}}[i,j] = \sum_{k} A_{\text{tile}}[i,k] \cdot B_{\text{tile}}[j,k]
\end{aligned}
\]
{{</rawhtml>}}
이 공식에서, 고정된 $i$에 대한 서로 다른 $j$값을 계산하는 워프 내의 스레드들은 `A_tile`의 행과 칩 안의 `B_tile`의 행에 접근합니다. 둘 모두 충돌이 없는 접근 패턴입니다. 이러한 단일 전략은 HBM 집약 요구와 SRAM 뱅크 충돌 문제 모두를 해결합니다.

```
Load-and-Transpose Operation (Thread tx, ty)
Reads row-wise from HBM, writes column-wise to SRAM

Global Memory (HBM)                Shared Memory (SRAM)
+-------------------------+        +-----------------------+
| B[k_base+ty, j_base+tx] | -----> |      B_tile[tx, ty]   |
+-------------------------+        +-----------------------+

Result: HBM reads are coalesced, SRAM reads are conflict-free.
```

### 칩 안의 연산 단계: 산술 강도의 증가

공유 메모리에 올라간 데이터로 블록은 계산을 수행합니다. 이 때의 목표는 온칩 메모리에서의 데이터 재사용을 최대화하는 것입니다. 우리는 칩 안 연산을 구성하는 두 가지 전략에 대해 분석할 것입니다.

#### 전략 1: 한 스레드가 하나의 출력을 계산하기

이 가장 간단한 접근은 하나의 출력 원소를 한 스레드와 사상하는 것입니다. `BLOCK_DIM`과 `TILE_DIM`이 같은 크기일 때  `BLOCK_DIM x BLOCK_DIM` 크기의 스레드 블록은 `TILE_DIM x TILE_DIM` 크기의 데이터 타일을 연산합니다. 이 전략은 개념적으로 공유 메모리 캐싱을 소개한 [Boehm의 포스트](https://siboehm.com/articles/22/CUDA-MMM)의 **Kernel 3**과 유사합니다. 블록당 1024개 스레드라는 하드웨어 제한은  `BLOCK_DIM`의 최대 크기가 32(역주. $N^2$가 1024를 넘을 수 없으므로 $N$은 32가 최대입니다)라는 제약을 만듭니다. 스레드 `(tx, ty)`는 단일 출력 원소 `C_patial[tx, ty]`를 책임지게 됩니다.

```
# Action for a single thread (tx, ty) where BLOCK_DIM = TILE_DIM
c_accumulator = 0.0
for k in range(TILE_DIM):
    c_accumulator += A_tile[ty, k] * B_tile[tx, k]
```

이 전략에서 산술 강도는 `TILE_DIM / 4`가 됩니다.

* **Total FLOPs:** 블록은 `2 \* TILE_DIM^3` FLOPs를 수행합니다.
* **Total Bytes Accessed (HBM):** 블록은 두 개의 데이터 타일을 불러오며, 이는 총 `8 \* TILE_DIM^2`바이트입니다.
* **Arithmetic Intensity (AI):** `(2 \* TILE_DIM^3) / (8 \* TILE_DIM^2)` = `TILE_DIM / 4`FLOPs/Byte입니다.

`TILE_DIM`이 32로 제한되면서, 최대 AI는 32 / 4 = 8이 됩니다. 이는 A100의 ridge point인 ~13에 비하면 충분하지 않습니다. 커널은 여전히 메모리 종속 영역에 위치합니다.

#### 전략 2: 한 스레드가 여러 출력을 계산하기

AI를 증가시키기 위해 우리는 스레드 개수의 증가 없이 `TILE_DIM`을 증가시켜야 합니다. 이는 데이터 타일의 크기와 스레드 블록의 크기가 서로 디커플링될 것을 요구합니다. 우리는 더 많은 작업을 각 스레드에 할당할 것이며, 이 전략은 마찬가지로 [Boehm의 포스트](https://siboehm.com/articles/22/CUDA-MMM)의 **Kernel 5**의 목표와 대응됩니다.

한 `16x16`스레드 블록(`BLOCK_DIM = 16`, 256 스레드)는 `64x64`크기의 데이터 타일(`TILE_DIM = 64`)을 연산할 수 있습니다. 각 스레드는 이제 출력의 `4x4` 서브 타일을 계산합니다. 이는 `TILE_DIM=64`가 공유 메모리의 허용 용량을 초과하지 않을 것을 요구합니다.

```
# A thread computes a 4x4 output sub-tile
# TILE_DIM = 64, BLOCK_DIM = 16
c_regs = [[0.0] * 4 for _ in range(4)]
a_regs = [0.0] * 4
b_regs = [0.0] * 4

for k in range(TILE_DIM):
    # Load a sliver of A_tile and B_tile into registers
    for i in range(4): a_regs[i] = A_tile[thread_row*4 + i, k]
    for j in range(4): b_regs[j] = B_tile[thread_col*4 + j, k]

    # Compute outer product in registers, accumulating into c_regs
    for i in range(4):
        for j in range(4):
            c_regs[i][j] += a_regs[i] * b_regs[j]
```

여전히 AI 계산은 `TIME_DIM / 4` 이며, `TILE_DIM = 64`일 때 AI는 `64 / 4 = 16`FLOPs/Byte입니다. 이는 A100의 ridge point를 초과합니다. **드디어 커널은 연산 종속 영역에 위치합니다.**

연산 종속된 커널의 런타임은 SM의 산술 연산 처리율에 의해 제한됩니다. 이는 절대적으로 높은 성능을 보장하지 않습니다. 초반에 논의했듯 FLOPs가 비효율적이거나 GPU 연산이 전력 제한에 의해 최대 클록 속도보다 낮은 속도로 동작할 경우 커널은 연산-종속 영역에 위치하면서도 여전히 느릴 수 있습니다(예시로, 텐서 코어와 같은 특수 하드웨어를 두고 FP32 스칼라 연산 레지스터를 사용한다고 생각해 보세요).

위 코드의 내부 루프는 더욱 최적화될 수 있습니다. 한 스레드는 네 개의 분리된 `float`값을 `A_tile`에서 `a_regs`로 불러옵니다. 이는 한 번의 16바이트의 `float4`벡터 불러오기 연산으로 대체할 수 있습니다. 공유 메모리로부터의 벡터화된 불러오기는 칩 안의 데이터 이동을 위한 명령어의 수를 감소시키고, 이는 연산 단계의 효율을 높입니다. 이는 [Boehm의 포스트](https://siboehm.com/articles/22/CUDA-MMM)의 Kernel 6 에서 사용된 온칩 벡터화 개선과 대응됩니다.

#### 마지막 고려 사항: 타일 양자화

만약 행렬의 차원들이 타일 사이즈의 배수가 아니라면, 커널은 낭비되는 계산을 수행하는 추가 블록을 실행합니다.

`M x N` 행렬을 TILE_`TILE_M x TILE_N` 타일로 덮기 위해 GPU는 `ceil(M/TILE_M) x ceil(N/TILE_N)` 스레드 블록의 그리드를 실행합니다. 따라서 65x65 행렬을 32x32타일로 타일링하는 것은 `ceil(65/32) x ceil(65/32)` = 3x3 블록 그리드를 요구합니다. 각각의 블록은 전체 32x32타일에 대해 산술 연산을 수행하도록 커널의 로직은 고정되어 있습니다.

[NVIDIA에 의하면](https://docs.nvidia.com/deeplearning/performance/dl-performance-matrix-multiplication/index.html#tile-quant), "라이브러리는 어떤 타일도 잘못된 메모리 접근을 수행하지 못하도록 보장하지만, 모든 타일은 동일한 양의 수학 연산을 수행합니다". 이러한 현상이 발생하는 원인에 대한 제 생각(잘못되었다면 기꺼이 수정하겠습니다)은 **커널이 명시적으로 데이터를 패딩하기 때문에 경계에 위치한 블록이 블록이 낭비되는 작업을 수행한다**는 것입니다(역주. 이 가정은 사실일 가능성이 매우 높습니다. 텍스쳐링에서 Pitch개념이 존재함을 생각해 본다면요). 행렬 경계의 외부에서 원소를 불러오도록 할당된 스레드는 방어 조건에 의해 해당 불러오기가 금지됩니다. 대신에, 온칩 공유 메모리 타일의 해당 위치에 0을 씁니다. 이 때 산술 루프는 단축되지 않습니다. 커널의 로직은 타일 전체에 걸쳐서 동일합니다. 한 워프 내의 모든 스레드들은 동일한 내적(곱-합) 명령어를 수행합니다. 0으로 패딩된 데이터에 대응되는 스레드는 여전히 이 명령을 수행할 것이며, 이는 단지 불필요한 연산을 수행하는 것입니다. $C += A * 0$과 같은 연산이죠. 하드웨어 자원은 사용되었지만, 결과는 버려집니다.

## 추가적인 성능 고려 사항

지금까지 저희는 저희의 커널을 연산-종속 영역에 위치하도록 만들었습니다. 이제 커널의 연산은 칩 내부 산술 연산의 속도에 의해 제한됩니다. 그러나, 커널은 하드웨어의 **추가적인 사항들을 고려**해 관리함으로써 여전히 더 빨라질 수 있습니다. 후술할 내용에는 그러한 세 가지의 고려 사항이 존재합니다. 다른 고려 사항들도 있지만, 아직 자세히 설명할 만큼 제 지식이 충분히 성숙되지 못했습니다. 다른 고려 사항은 [Boehm의 포스트](https://siboehm.com/articles/22/CUDA-MMM)를 참조하세요.

### 점유율 및 지연 숨기기

워프는 전역 메모리 읽기와 같이 긴 **지연(Latency)** 시간을 가진 명령어를 실행할 때 **멈추게 됩니다.** 이 경우 수백 사이클에 걸려 해당 데이터가 도착하기 전까지 다음 명령어를 실행하지 못하게 됩니다. 이 시간 동안, SM의 연산 장치는 멈춘 워프가 다시 일할 수 있게 될 때까지 잠들게 될 수 있습니다.

SM은 다른 작업을 함으로써 이러한 지연을 숨길 수 있습니다. SM은 상주 워프 풀(A pool of regident warps)을 만듦으로써 여러 스레드 블록을 동시적으로 맡을 수 있습니다. 한 워프가 멈추게 되면, SM의 하드웨어는 즉시 풀에서 실행 가능한 다른 워프를 꺼내 교체합니다. 이러한 동작 방식을 **지연 숨기기(latency hiding)** 이라 합니다.

```
+-------------------------------------------------------------------+
| Streaming Multiprocessor (SM)                                     |
|                                                                   |
|  [Block A]              [Block B]                                 |
|   - Warp A1 (Ready)      - Warp B1 (Ready)                        |
|   - Warp A2 (Stalled -> waiting on HBM)                           |
|        |                  |                                       |
|        +------------------v------------------+                    |
|           [ Pool of Ready-to-Run Warps ]                          |
|           [ A1, B1 ]                                              |
|                           |                                       |
|                   +-------v-------+                               |
|                   | SM Scheduler  | --> [Execute instructions]    |
|                   +---------------+                               |
|                                                                   |
+-------------------------------------------------------------------+
```

**점유율(Occupancy)** 은 SM에서 **활성화된 워프 수와 SM이 지원할 수 있는 최대 워프 수의 비율**입니다. 점유율이 높으면 스케줄러가 선택할 수 있는 워프의 풀이 더 넓어집니다. 이를 통해 주어진 사이클에 실행할 준비가 된 워프를 찾을 가능성이 높아지고 이는 곧 연산 장치가 활성 상태를 유지함을 의미합니다.

이로 인해 블록당 사용되는 리소스와 SM에 상주할 수 있는 블록 수 사이에 트레이드오프가 발생합니다. 두 가지 극단적인 상황을 시각화하면 아래와 같습니다.

```
+------------------------------------+ +----------------------------------------------+
| SM with High AI, Low Occupancy     | | SM with Low AI, High Occupancy               |
|                                    | |                                              |
| +--------------------------------+ | | +----------+ +-----------+     +-----------+ |
| | Block 0 (uses 64KB SMEM)       | | | | Block 0  | | Block 1   | ... | Block N   | |
| | TILE_DIM=128 -> High AI        | | | | (8KB SMEM) | (8KB SMEM)|     | (8KB SMEM)| |
| +--------------------------------+ | | +----------+ +-----------+     +-----------+ |
|                                    | |                                              |
| --> Low # of resident blocks.      | | --> High # of resident blocks.               |
| --> Small pool of warps for        | | --> Large pool of warps for                  |
|     latency hiding.                | |     latency hiding.                          |
+------------------------------------+ +----------------------------------------------+
```

우리는 높은 AI의 이점과 충분한 점유율의 필요성 사이의 균형을 맞추기 위해 커널의 리소스 사용량을 조절합니다. 이 때 조절의 주요 기준은 **스레드 블록의 차원(`BLOCK_DIM`), 블록 당 할당된 공유 메모리의 양(`TILE_DIM`에 의해 결정됩니다), 그리고 한 스레드 당 사용하는 레지스터의 개수**입니다.

### 스레드 분기 피하기

워프의 모든 스레드가 동일한 결과를 갖지 않은 조건 분기(`if-else`)는 **스레드 분기를 야기합니다.** 스레드 분기가 발생하면 하드웨어는 다른 코드 경로를 직렬적으로 실행함으로써 분기를 해소합니다. 첫 번째로, `if` 분기를 타는 스레드들을 실행시키고 그 동안 다른 스레드들(`else` 분기)을 **정지**시킵니다. 그 다음 반대로 `if`분기를 타는 스레드들을 **정지**시키고 `else`분기를 타는 스레드들을 실행합니다.

```
# A warp of 32 threads encounters an `if` statement:
if (thread_id < 16) 
    # Path A
else 
    # Path B

Execution Timeline:

Time ->
+------------------------------------------------------------------+
| Warp Execution                                                   |
|                                                                  |
|  Cycle 1: Path A is executed.                                    |
|   - Threads 0-15:  Active, execute Path A code.                  |
|   - Threads 16-31: Inactive, masked off.                         |
|                                                                  |
|  Cycle 2: Path B is executed.                                    |
|   - Threads 0-15:  Inactive, masked off.                         |
|   - Threads 16-31: Active, execute Path B code.                  |
|                                                                  |
| Result: Two cycles are required instead of one.                  |
|         Effective throughput is halved.                          |
+------------------------------------------------------------------+
```

이러한 직렬화는 분기 코드의 실행 시간을 두 배로 늘리며 동시에 워프의 처리율을 반토막냅니다. 우리는 이 비용을 `if-else` 대신 `min`과 `max`를 사용하는 등의 성능에 민감한 영역에 분기가 없는 코드를 작성함으로써 피해야 합니다(역주. 이러한 이유 때문에 그래픽스 프로그래밍에서는 `#define`을 통한 쉐이더 배리언트의 개념을 사용합니다).

### 양자화(Quantization)

양자화는 텐서 원소의 정밀도를 `FP32`에서 `FP16`또는 `BFP16`으로 감소시킵니다. 이는 두 가지 효과를 가져옵니다. 첫 번째로, 각 요소를 저장하는 데 필요한 메모리를 예를 들자면 2만큼 줄인다고 해 봅시다. 이를 통해 초당 두 배 더 많은 요소를 전역 메모리에서 공유 메모리로 전송할 수 있게 됩니다. 이렇게 하면 AI가 2만큼 증가합니다.

두 번째로, A100과 같은 GPU들은 **더 낮은 정밀도의 원소일수록 더 빠르게 연산을 수행**할 수 있습니다. 예를 들어, A100에서 특정한 FP16연산은 312TFLOPs를 달성할 수 있는 반면, FP32연산은 19.5TFLOPs로 제한됩니다. 따라서 이론적으로 연산 속도를 16배 향상시킬 수 있습니다.

그러므로 양자화를 통해 루프라인 모델 그래프에서 커널을 우상단으로 이동시킬 수 있습니다.