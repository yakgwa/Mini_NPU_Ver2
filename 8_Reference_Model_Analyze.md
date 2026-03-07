## [Reference Model 문제점 파악/개선 모델 최초 제시] - MNIST 손글씨 데이터 인식 목적의 추론형 NPU

### 🔴 Reference Model, ZyNet의 문제점 파악 🔴
- Performance1(Cycle & Latency/Throughput) 관점
  - Layer 1 Cycle
    - 784 cycle : 최초 Layer1 입력 시, in[783:0] (X0~X783)이 직렬로 stream
    - 2 cycle : myinputValid → weight_valid → mult_valid(neuron.v)로 valid가 2번 밀림
    - 2 cycle : muxValid_f → sigValid → outvalid(neuron.v)로 valid가 2번 밀림
    - Layer1 출력 valid : 784 + 2 + 2 cycle 

  - 직렬화 FSM1 + Layer 2 Cycle
    - 1 cycle : o*_valid[0] → holdData<=x*_out으로 SEND 상태로 바꿈
    - 30 cycle : Layer1의 출력 30개를 30클록에 걸쳐 Layer2로 흘림
    - 2 cycle + 2cycle
    - 직렬화 FSM + Layer2 출력 valid : 1 + 30 + 2 + 2 cycle

  - 직렬화 FSM2 + Layer 3 Cycle
    - 1 cycle + 20 cycle
    - 2 cycle + 2cycle
    - 직렬화 FSM + Layer2 출력 valid : 1 + 20 + 2 + 2 cycle

  - maxFinder Cycle
     - 10cycle

    👉 이론적인 사이클 분석 결과, 단순 연산 흐름만 고려할 경우 입력 스트림이 시작된 시점으로부터 약 858 cycle 이후에 최종 결과(out_valid)가 출력된다.

    👉 이때 Latency는 단순히 MAC 연산량에 의해서만 결정되는 것이 아니라, 레이어 간 결과를 전달하기 위해 수행되는 데이터 재포맷 및 직렬화 과정에 의해 크게 증가한다.

    👉 입력이 1 cycle당 1개의 stream 형태로 유입되며, 해당 입력은 내부에서 모든 뉴런으로 브로드캐스트되어 각 뉴런이 동일 입력에 대해 병렬로 누산 연산을 수행한다.

    👉 그러나 각 레이어의 출력은 벡터 형태로 생성된 이후, 직렬화 FSM을 통해 다시 스칼라 스트림으로 변환되어 다음 레이어로 전달된다. 이 과정에서 레이어마다 추가적인 사이클 오버헤드가 발생한다.

    👉 Throughput은 단위 시간당 처리 가능한 입력 샘플 수로 정의되며, 본 설계에서는 MNIST 이미지 한 장(in[783:0])의 처리율에 해당한다.

    👉 ZyNet 구조상 Layer2와 Layer3는 Layer1이 전체 입력(784 cycle)을 모두 처리하여 출력을 생성하기 전까지는 어떠한 연산도 수행할 수 없다.

    👉 결과적으로, 하나의 샘플에 대해 Layer1이 784 cycle 동안 연산을 수행한 이후, Layer2와 Layer3는 각각 30 cycle과 20 cycle 동안만 활성화된다.

    👉 이 동안 다음 입력 샘플은 다시 Layer1에서 784 cycle 동안 처리되며, 이 중 약 734 cycle(≈ 92.6%) 동안 Layer2와 Layer3는 유휴 상태(idle)로 남게 된다.

    👉 따라서 ZyNet의 steady-state throughput은 가장 연산량이 큰 Layer1에 의해 사실상 결정되며, Layer2와 Layer3의 하드웨어 자원은 throughput 관점에서 충분히 활용되지 못한다. 

- Performance2(TOPS, Memory) 관점

<table style="text-align: center;">
  <tr>
    <td width="150px">레이어</td>
    <td width="100px">입력 수</td>
    <td width="100px">뉴런 수</td>    
    <td width="150px">MAC 수</td>   
  </tr>
  <tr>
    <td><strong>Layer1</strong></td>
    <td><strong>784</strong></td>
    <td><strong>30</strong></td>    
    <td><strong>23,520</strong></td>         
  </tr>
  <tr>
    <td><strong>Layer2</strong></td>
    <td><strong>30</strong></td>
    <td><strong>20</strong></td>    
    <td><strong>600</strong></td>         
  </tr>
  <tr>
    <td><strong>Layer3</strong></td>
    <td><strong>20</strong></td>
    <td><strong>10</strong></td>    
    <td><strong>200</strong></td>         
  </tr>  
  <tr>
    <td colspan="3"><strong>합계 (mul+add를 op 2개로 보면 48,640 ops)</strong></td>
    <td><strong>24,320 MAC/inf</strong></td>  
  </tr>
  <tr>
    <td colspan="4"><strong>Layer2, Layer3는 784 클록 주기 대비 상당히 짧게만 동작하여, 자원 활용이 충분히 이루어지지 않음.<br>(Layer2 : 30 cycle / 784 cycle = 약 3.8%, Layer3 : 20 cycle / 784 cycle = 약 2.6%)</strong></td>
  </tr>  
</table>

    👉 ZyNet은 가중치를 (784×30, 30×20, 20×10 …) 형태로 한 번 로딩해 각 뉴런의 로컬 Weight Memory에 저장해 둔다. 
  
    👉 이후 입력은 스트리밍으로 들어오며, 각 뉴런은 매 사이클 1개의 weight를 로컬 메모리에서 읽어 1 MAC을 수행한다. 
  
    👉 이때 필요 대역폭 자체는 1 read/cycle로 낮지만, 가중치 재사용이 거의 없어 Arithmetic Intensity(ops/byte)가 매우 낮다. 
  
    👉 따라서 재사용성 없는 데이터플로우와 레이어 간 직렬화로 인한 낮은 연산 활용도를 개선할 필요가 있다.

- Power/Area 관점

  - “뉴런 복제”로 인해 면적이 뉴런 수에 선형으로 폭증 : 각 뉴런마다 weight memory + mul + add + acc + activation 등이 들어 있는데, 이는 뉴런 수와 레이어 수에 면적이 선형적으로 증가하기 떄문에 확장성 측면에서 최악이다.
  - Layer 2/3 면적이 "성능에 기여하지 않는 면적"이 됨 : 동작 듀티란, 전체 시간 중 실제로 유효 연산을 하고 있는 비율을 의미하는데, Layer 1은 매 cycle마다 30 MAC을 동작하지만, 한 샘플 처리 기준 Layer2, Layer3는 대부분의 시간을 놀고 있기 떄문에 동작 듀티가 매우 낮다. 실질적으로 Layer2, Layer3을 합친 면적은 Layer1과 거의 비슷하지만, 성능 대비 면적 효율(area efficiency)는 매우 나쁘다. Performance 측면에서도 Throughput은 Layer2, Layer3가 아닌 오로지 Layer1이 결정하며, Power 측면에서도 Layer2, Layer3는 대부분 idle 상태이지만 지속적인 전력이 발생할 수 밖에 없다. (clock gating이 거의 없는 구조이기 때문에 계속 전력을 소모할 수 밖에 없음.)
  - 뉴런마다 가지고 있는 weight_memory : 뉴런마다 하나씩 가지고 있는 메모리는 면적 뿐 아니라 전력 문제에서도 상당한 타격을 입는다. 예컨대, Layer1에서 30개 weight memory를 매 클록 read하는 구조이기 때문에 전력에 불리할 수 밖에 없다.

### 🟢개선 방향 제시🟢

최종적으로는 현재의 Reference Model을 NPU 모델, 특히 Systolic Array 기반으로 개선해보고자 한다. 우리가 개선할 모델은 단순 'MLP' 연산 및 'Inference' 만을 지원하는 Mini-NPU 모델로 생각하면 된다. 이때 성능을 좌우하는 핵심 설계 포인트는 다음과 같이 두 가지라고 생각된다. 

- 타일링 전략 및 어레이 크기 (2×2, 3×3, 4×4, 5×5 …)
- 데이터플로우(Dataflow) 선택 (OS / WS / IS 중 무엇을 쓸 것인지)

특히 Systolic Array의 데이터플로우 선정은 하드웨어 설계에서 가장 중요한 의사결정 중 하나이다. 데이터플로우 선택의 본질은 결국 "무엇을 PE에 Stationary시켜 데이터 이동 에너지를 줄일 것인가?" 이다.

✅ 데이터플로우 : WS/IS/OS 개념과 장단점 요약
- WS : WS는 Weight를 PE에 고정하고, Input을 흘려보내며, psum을 누적해 출력값을 만든다. 이는 weight를 한 번 로딩하면 오랫동안 재사용 가능하여 특히 CNN처럼 동일 Kernel로 공간 위치를 계속 찍어내는 연산에 유리하다. 또한 Batch Size가 클 경우, Weight 1세트를 로드해 다수 샘플에 반복 사용하므로 효과가 커질 수 있다. 하지만, Batch가 작고 MLP 구조라면 weight 재사용 이득이 크지 않을 수 있다. 구현에 따라 psum이 PE 간 이동하면서 누산되면, array 내부 데이터 이동 비용이 커질 수 있다.
- IS : IS는 Input을 PE에 고정하고, weight를 흘려보내며, psum을 누적하는 방식이다. Input을 극한으로 재사용할 때 강점이 있어, 일부 특수 필터 연산이나 특정 상황에서 Input reuse가 클 때 유리할 수 있지만, GEMM 연산 관점에서는 WS/OS가 더 자주 채택되는 편이며, 적용 난이도와 복잡성 때문에 덜 쓰이는 경향이 있다고 생각한다.
- OS : OS는 Output(=psum)을 PE에 고정해두고, Input과 weight가 PE로 흘러 들어와 곱해지며 누산된다. 연산이 끝나면 완성된 결과만 밖으로 내보낸다는 관점으로 설명할 수 있다. 이는 긴 누산 구산에서 psum을 외부로 내보내거나 다시 불러오는 동작을 최소화할 수 있다. 특히 MLP처럼 reduction dimension(weight 784개)이 매우 길 때 누산이 오래 지속되므로 OS가 매력적일 수 있다. 단, psum을 PE에 고정하는 대신, 두 값을 계속 공급해야 하므로 Input/weight 경우 대역폭 및 데이터 공급 구조가 중요할 수 있다.

✅ 현재 시스템에는 OS가 가장 유리하다고 판단

- [이유 1] Reduction Dimension(784)이 매우 큼
  - 현재 연산은 최초 레이어1에서 출력 1개를 만들기 위해 784번의 MAC 누산이 필요하다.
  - WS를 쓸 경우: 구현에 따라 psum이 누적되면서 내부 데이터 이동(레지스터/배선 토글)이 커질 수 있다.
  - OS를 쓸 경우: psum을 PE 내부에 고정하고 784번 동안 가만히 앉아서 더하기만 하면 되므로, psum의 이동/왕복 비용을 구조적으로 최소화할 수 있다. 따라서 reduction이 긴 FC 계열에서는 OS가 유리

- [이유 2] Batch Size가 작음
  - 현실적으로 학부/스터디 수준에서 혼자 구현 가능한 규모는 타일링이 “수×수” 정도(예: 4×4 등)로 제한되며, 결국 한 번에 처리할 수 있는 샘플 수(batch)는 크기 어렵다. 
  - WS의 경우, Weight를 고정했을 때 그 weight를 최대한 많이 재사용해야 한다. 예컨데, batch=100이면 weight 로딩 1번으로 100번 연산하므로 이득이 커진다.  하지만 현재는 batch를 크게 키우기 어렵고, 따라서 weight를 로딩해도 몇 번 연산하면 또 다른 weight로 갈아 끼워야 한다. 예컨대 4×4라고 하면, weight preload 비용이 발생할 수 있고, batch가 작으면 그 비용이 상대적으로 더 크게 느껴질 수 있다.

   - 1️⃣ 타일링 전략
     - OS 방식으로 선정했다고 끝이 아니다. 타일링은 결국 성능(처리량/지연시간)뿐 아니라 면적/전력/클록과도 직결된다. 우선 전체적인 타일링 전략은 예컨대, Layer1에서 총 30개의 Output Feature(Y1~Y30)를 계산하기 위해 Weight Matrix를 출력 차원 기준으로 분할한다. 즉, 출력 뉴런을 4개씩 묶으면, 총 8회의 타일링(Tiling) : (뉴런 1~4), (5~8), … (29~30)을 통해 순차적으로 연산을 완료하도록 가져갈 수 있다. 

   - 2️⃣ 어레이 크기 선택
     - 어레이 크기에 대해 결론을 내리기 전에 반드시 고려해야 할 점이 있다. (1) Systolic Array를 작게 만들면 면적이 크게 줄지만, 그만큼 연산 병렬도가 낮아져 레이턴시가 길어질 수 밖에 없다. 즉, 면적은 어마어마하게 줄지만 1 sample에 대한 latency가 느려질 수 있다는 문제가 생긴다. 그래서 단일 샘플만 빠르게 끝내기보다는, 여러 샘플을 Systolic Array에 동시에 넣어(배치/스트리밍) latency는 길어도 처리량을 확보하는 방향이 설득력 있다. 또 하나 중요한 현실 제약은 (2) 데이터를 한 번에 가져올 수 있는 양이 한정적이라는 점이다. 이때 AMBA BUS Protocol(예: AXI 등)을 고려해야 하는데, 아직 AMBA/AXI를 본격적으로 배우진 않았더라도, “버스를 통해 한 번에 전송 가능한 폭/대역폭이 제한되어 결과적으로 아키텍처 선택이 제약된다”는 점은 언급하는 것이 타당하다. 또한 (3) Array를 키우면 처리량은 올라갈 수 있지만, 동시에 PE 간 배선 증가, clock skew, fmax 하락등으로 이어질 수 있어 면적/전력/타이밍의 trade-off를 함께 고려해야 한다. 

​

성능 비교를 위한 Notation

$n번째\ sample\ :\ x_n^{\left(1\right)}\ \sim \ x_n^{\left(784\right)}\ \left(MNIST\ Dataset\right)$n번째 sample : x
(1)
n​ ~ x
(784)
n​ (MNIST Dataset)​
$$​
$Layer1\ Neuron"s\ Weight,\ Neuron\ m\ weight\ :\ w_m^{\left(1\right)}\ \sim \ w_m^{\left(784\right)}$Layer1 Neuron′s Weight, Neuron m weight : w
(1)
m​ ~ w
(784)
m​​
$Layer2\ Neuron"s\ Weight,\ Neuron\ m\ weight\ :\ w_m^{\left(1\right)}\ \sim \ w_m^{\left(30\right)}$Layer2 Neuron′s Weight, Neuron m weight : w
(1)
m​ ~ w
(30)
m​​
$Layer3\ Neuron"s\ Weight,\ Neuron\ m\ weight\ :\ w_m^{\left(1\right)}\ \sim \ w_m^{\left(20\right)}$Layer3 Neuron′s Weight, Neuron m weight : w
(1)
m​ ~ w
(20)
m​​
$$​
$Layer1\ output\ :\ Y_1^{\left(1\right)}\ \sim \ Y_1^{\left(30\right)}$Layer1 output : Y
(1)
1​ ~ Y
(30)
1​​
$Layer2\ output\ :\ Y_1^{\left(1\right)}\ \sim \ Y_1^{\left(20\right)}$Layer2 output : Y
(1)
1​ ~ Y
(20)
1​​
$Layer3\ output\ :\ Y_1^{\left(1\right)}\ \sim \ Y_1^{\left(10\right)}$Layer3 output : Y
(1)
1​ ~ Y
(10)
1​​
아래는 “면적을 줄이기 위해 PE를 줄이고(1D Chain), 또는 2D Systolic Array로 확장했을 때”의 대략적인 latency/throughput 추정이다. 

​

1D (1-dimension) 1×2 PE Chain

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_11.png" width="400"/>

Cycle 0

<div align="left">
 

Cycle 0, Cycle1


Cycle 2

....


Cycle 784

그림을 통해 알 수 있듯이 Layer1의 출력은 총 30개인데, 1x2 Chain은 그 중 2개의 출력만 만든다. 따라서 Layer1을 모두 수행하려면 이를 15번 반복해야 한다. Layer2, 3도 같은 방식으로 진행한다.

대략적인 1 sample latency: 784×15+30×10+20×5=12,160 cycles

기존 ZyNet은 (아주 대략적으로) 784+30+20=834 cycles 수준으로 1 sample이 나온다고 보면, 면적은 어마어마하게 줄일 수 있으나, 1 sample latency는 현저히 느려진다.

​

1D (1-dimension) 1×4 PE Chain

PE를 4개로 늘리면 병렬도가 증가해 반복 횟수가 줄어든다.

대략적인 1 sample latency: 784×8+30×5+20×3=6,482 cycles

기존 ZyNet(834 cycles) 대비, 면적은 줄일 수 있으나 여전히 1 sample latency는 느리다.

​

2D (2-dimension) 4×4 Systolic Array (Batch=4)

따라서 최초 설계는 다음과 같이 4x4 Systolic Array 수준으로 설계하고자 한다. 물론 정확한 수치적인 측면은 컨트롤/파이프라인/메모리 구조에 따라 달라질 수 있으나, 설계 방향성을 잡기 위한 1차 모델로 제시한다. ‘최소한의 복잡도로 OS dataflow의 Systolic Array의 모든 핵심 개념을 드러내는 교육·검증용 구성’이다. 레이어1 병목을 이기기 위한 최적 설계라기보다는, 기존 ZyNet과 구조적 차이를 비교하기 위한 anchor로 최초 설계를 진행하고자 한다.

여기서 기존 1D PE Chain과의 차이점은 한 샘플에 대한 레이턴시가 느려져도, 여러 샘플을 동시에 넣어 throughput에서 이득을 보자는 관점으로 Batch 4개를 한 번에 처리하고자 한다. 대략적인 Layer1 사이클 흐름은 다음과 같다.

 

Cycle 0, 1

 

Cycle 2, 3

 

Cycle 4, 5

....


Cycle 784, 그 이후 ~

대략적인 4 sample latency(보수적으로) : 784×8+30×5+20×3=6,482 cycles

이때 샘플 처리량(throughput)을 단순 계산하면,

4/6482≈0.0006171 sample/cycle

그리고 기존 ZyNet이 1 sample 당 834 cycle이라면:

1834≈0.001198 sample/cycle

이므로, 이 단순히 Systolic Array만 생각한 모델에서는 기존 ZyNet이 샘플당 처리량도 더 빠를 수 있다는 결과가 나온다. 물론 이러한 비교는 정말 단순히 Systolic Array만 놓고 봤기 때문에 double-buffering을 통한 메모리 접근 숨김, 그리고 데이터 공급 구조와 같은 요소로 최적화가 이루어진다면 더 개선될 여지도, 외부 메모리나 데이터 인터페이스를 추가로 고려한다면 오히려 더 안 좋아질 여지도 충분히 보인다.

​

또 하나 중요한 점은 4x4가 반드시 최적의 설계 지점은 역시 아닐 수 있다는 점이다. 예컨대, array 크기를 5×5, 6×6으로 확장하면 면적과 전력은 분명히 증가하지만, Systolic Array의 본질적인 장점은 기존에 복제된 뉴런 연산을 하나의 규칙적인 MAC 구조로 흡수할 수 있다는 데 있다. 즉,  더 큰 어레이로 연산을 “퉁쳐서” 처리할 수 있다면, 일정 수준 면적·전력 증가를 감안하는 선에서 성능 향상을 끌어올릴 여지도 보인다.

​

아주 러프하게 6x6 Systolic Array '자체'의 계산량만 고려하면, 

대략적인 6 sample latency(보수적으로) : 784×5+30×4+20×2=4,080 cycles

수준까지 레이턴시를 줄일 수 있고, 이를 6×6 기준으로 처리량으로 환산하면,

6/4080≈0.00147 sample/cycle

에 도달할 수 있다. 사이클 자체로만 처리량을 놓고보면, 6x6는 기존 ZyNet보다 오히려 더 높은 처리량을  '이론적으로' 가질 수 있는 분기점이 될 수 있다.

​

다만, 이 지점에서 가장 큰 관심사는 결국 데이터 버스 전송 폭과 공급 구조다. 어레이 크기가 커질수록 PE 내부의 연산 자원은 늘어나지만, 외부 메모리나 AMBA 인터페이스가 충분한 대역폭으로 activation과 weight를 공급하지 못하면 계산 유닛은 쉽게 놀게 된다. 따라서 실제 성능의 상한은 “MAC 개수”가 아니라, 버스 폭, 전송 스케줄링, 그리고 double-buffering이 얼마나 잘 맞물리는가에 의해 결정될 가능성이 크다. 따라서 NPU 설계의 핵심 과제는 “주어진 데이터 플로우에서 어레이를 얼마나 키울 수 있는가”가 아니라, “주어진 데이터 버스/플로우 조건에서 어레이를 얼마나 효율적으로 먹여 살릴 수 있는가”라고 생각된다.

​

최종적인 4x4 Systolic Array 기반 NPU 모델 제시

(논문 : Google TPU, Gemmini 참고)
