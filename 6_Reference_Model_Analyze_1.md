## [Reference Model 분석] - CNN-Handwritten-Digit-MNIST
앞서 두 가지 논문을 살펴보며, NPU를 하드웨어 아키텍처 관점에서 분석해왔다. 이 과정에서 GEMM을 중심으로 한 연산 구조, 데이터플로우 설계, 그리고 Systolic array가 갖는 구조적 장점에 대해 이해할 수 있었다. 이제는 실제로 뉴럴넷넷 연산이 RTL 레벨에서 어떻게 구현되는지를 살펴볼 차례다. 위 Github에서 제시된 바와 같이, MNIST 손글씨 숫자 인식을 목적으로 한 Neural Network를 FPGA 상에 직접 구현한 reference model을 분석한다. 디렉토리 구조는 학습을 담당하는 SW 영역(Train), NN 연산을 수행하는 HW 영역(RTL), 그리고 전체 동작을 검증하기 위한 시뮬레이션 영역(Simulation)으로 분리되어 있다. 

이 Reference model은 NN 연산을 우리가 직관적으로 떠올리는 방식, 즉, 뉴런 단위의 MAC 연산과 Activation 처리를 중심으로 설계되어 있다. 이러한 구조는 RTL 관점에서 뉴럴 네트워크를 이해하기에는 적합하지만, 성능과 효율 측면에서는 최적이라고 보기 어렵다. 따라서 궁극적인 목표는, 이와 같은 직관적인 뉴럴넷 RTL 구조를 출발점으로 삼아, GEMM 기반 연산으로 재해석하고, Systolic Array 구조를 적용하여 하드웨어적으로 최적화된 NPU 아키텍처로 확장하는 것이다. 그 첫 단계로, 해당 reference model의 전체 구조와 설계 의도를 파악하고, NN 연산이 RTL로 어떻게 풀어져 있는지를 살펴본다. 이를 통해 앞서 살펴본 논문에서 사용된 구조적 아이디어를 실제 RTL로 연결해 나고자 한다.

● GitHub에서 사용할 코드 (대략적인 디렉토리 구조)

    CNN-Handwritten-Digit-MNIST/
    ├── Network/Vivado
        ├── Python/
            ├── genTestData.py
        ├── src/fpga/
            ├── rtl/
                 └─ *.v
                 └─ *.mif
            └── tb/
                 └─ top_sim.v
