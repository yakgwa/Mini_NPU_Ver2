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

GitHub에서도 확인할 수 있듯이, top model인 ZyNet은 논문에서 제시된 바와 같이, Python으로 학습한 뉴럴넷을 FPGA에서 실행 가능한 RTL 구조로 자동 변환하는 것을 목표로 한 플랫폼이다. ZyNet은 Python으로 정의된 DNN 모델로부터 가중치를 가져와, 해당 구조에 맞는 합성 가능한 Verilog RTL 코드를 생성하고, 이를 FPGA에 연결함으로써 별도의 RTL 설계 없이도 즉시 Inference를 수행할 수 있는 환경을 제공한다. 논문에서 제시하는 ZyNet의 등장 배경은 IoT 기반 애플리케이션 확산에 따라 데이터가 중앙 서버가 아닌, 데이터 생성 지점과 가까운 엣지 디바이스에서 처리될 필요성이 커지고 있다. 엣지 컴퓨팅은 데이터 latency를 줄이고, 네트워크 대역폭 사용을 최소화하며, 실시간 처리가 가능하다는 장점을 가진다. 예컨대, 수집된 센서 데이터를 외부 클라우드 서버로 전송하지 않고, 공장 내부의 소형 시스템에서 즉시 분석·판단하는 방식이 대표적인 엣지 컴퓨팅의 사례다.

이러한 엣지 환경에서는 저전력·저비용 조건을 만족하면서도 신경망 연산을 수행할 수 있는 하드웨어 가속기가 요구된다. FPGA는 병렬 연산 구조로 인해 신경망 연산에 적합한 플랫폼이지만, 기존 FPGA 기반 DNN 구현 방식은 복잡한 개발 흐름과 고가 디바이스 중심의 툴 체인으로 인해 접근성이 낮았다. ZyNet은 이러한 문제를 해결하기 위해, TensorFlow와 유사한 Python 기반 모델 정의 방식으로부터 합성 가능한 DNN RTL과 주변 시스템을 자동으로 생성함으로써, FPGA 기반 신경망 하드웨어 구현을 보다 현실적으로 가능하게 하도록 제안되었다. ZyNet이 수행하는 작업은 다음과 같다.

- Python 환경에서 TensorFlow와 유사한 방식으로 DNN 구조 정의
- 사전 학습된(pre-trained) 가중치를 기반으로 합성 가능한 Verilog RTL 자동 생성
- AXI 인터페이스, DMA, 인터럽트 등을 포함한 SoC 수준의 DNN 시스템 구성

<br>

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_1.png" width="400"/>

<div align="left">

<br>

ZyNet에서 사용되는 single artificial neuron은 Fig.2.(a)에 제시되어 있다. 이는 우리가 직관적으로 알 수 있는 수준의 하드웨어 구조로 구현한 것이다. ZyNet의 각 뉴런은 전용 Weight Memory를 가지고 있다. 이 에모리는 해당 뉴런으로 들어오는 입력 개수에 따라 크기가 결정되며, 각 입력에 대응하는 가중치 값이 순서대로 저장된다. (뉴럴넷 입력 수 = 가중치 수) 네트워크 설정에 따라 이 메모리는 두 가지 방식으로 구현된다.

- Pre-trained 모드
- 가중치가 미리 학습된 경우, .mif 파일로 초기화된 ROM 형태로 동작
- On-board training 모드
- ARM 프로세서가 AXI-Lite 인터페이스를 통해 가중치를 갱신할 수 있도록 RAM 형태로 동작

즉, ZyNet은 가중치를 “계산”하지는 않지만, 가중치를 저장하고 접근하는 구조는 하드웨어 안에 존재한다. ZyNet 뉴런의 또 다른 특징 중 하나는 입력이 한 번에 하나씩 순차적으로(streaming) 들어온다는 점이다. 다시 말해, 입력이 여러 개 있더라도 병렬로 들어오지 않고, 클럭마다 하나의 입력 값만 전달된다. 이 구조 덕분에 뉴런 내부 하드웨어는 단순해지고, 대신 전체 연산이 끝나기까지의 latency는 늘어난다. 뉴런 내부의 Control Logic은 지금 들어온 입력이 몇 번째 입력인지 관리, 해당 입력에 대응하는 가중치 주소 생성, MAC 연산이 언제 시작되고 언제 끝나는지를 제어한다. 입력이 들어오면 뉴런은 weight memory에서 가중치 하나를 읽고, 입력 값과 가중치를 곱한 뒤, accumulator에 더한다. 이 과정이 바로 MAC 연산이며, 모든 입력에 대해 이 과정을 반복한 후 하나의 누적 결과를 만든다. 모든 입력에 대한 MAC 연산이 끝나면, 누적된 결과에 bias 값이 더해진다. Bias 값 역시 .mif 파일 또는 AXI-Lite를 통해 설정되며, 뉴런 내부 레지스터에 저장된다. Bias까지 더해진 결과는 마지막으로 activation function를 통과한다. ZyNet에서 ReLU는 간단한 비교 로직, Sigmoid는 LUT(ROM) 형태로 구현되어 있으며, 이를 통해 뉴런의 최종 출력 값이 결정된다.

다만, 본 글에서 ZyNet을 바라보는 관점은 논문에서 제시하는 활용 방향과는 다소 다르다. ZyNet을 엣지 디바이스용 완성형 개발 도구로 사용하기보다는, FPGA 기반 DNN의 기본적인 RTL 구조를 빠르게 확보하고, 이를 출발점으로 NPU 아키텍처의 데이터플로우, 병렬성, 연산 구조를 실험하고 분석하기 위한 연구용 레퍼런스로 활용하고자 한다. 대략적인 활용방법은 다음과 같다.

- ZyNet : 뉴런을 많이 복제해서 병렬성 확보
- 개선방안 : 4X4 Systolic Array를 구현하여 MAC 16개(PE)로 고정 및 타일링 (GEMM 연산)

1️⃣ 기존 레퍼런스 모델 장단점 : ZyNet

✅ 장점 : 
- 구현이 매우 직관적/쉬움
- 뉴런 = MAC + weight RAM + activation이라 구조가 단순함.
- 한 번에 많은 출력 뉴런을 동시에 계산
- 뉴런 단위로 결과를 찍고, weight/bias 로딩 검증하기 편함.
- 입력 스트림만 잘 흘려주면 각 뉴런이 알아서 누산하고 끝남.

❌ 단점 : 
- 면적/자원 소모
- 출력 뉴런 수만큼 MAC 유닛 / weight memory 수가 비례함.
- 뉴런마다 하나의 weight memory가 있기 때문에 memory가 분산되어 있음.
- 레이어 규모가 커짐에 따라 자원 낭비가 심해짐.
- 레이어마다 존재하는 뉴런은 병렬적으로 실행되지만, 입력은 직렬적으로 스트리밍하기 때문에 Throughput 관점에서 최악임.

2️⃣ 제안된 모델의 장단점 : 4×4 Systolic Array

✅ 장점 :
- 면적 효율이 좋음
- 데이터 재사용/대역폭 최적화/파이프라이닝으로 성능 향상.
- FC는 물론, Conv도 im2col 타일으로 연결 가능.

❌ 단점 : 
- batch=1(MNIST 추론)에서는 활용률이 낮을 수 있음. (latency 성능은 안 좋을 수 있음.)
- Scratchpad, Weight FIFO 등 구현 필요

||Reference Model|Proposed NPU|
|------|---|---|
|단일 샘플 latency|👍 빠름|👎 느림|
|배치 throughput|👍|👍👍👍|
|면적|👎|👍👍|
|확장성|👎|👍👍|
|데이터 재사용|👎|👍👍|

### Reference Model Analysis : top_sim.v
본격적으로 Reference Model을 분석해보자. 우선 RTL 코드를 분석할 때 가장 먼저 확인해야 할 대상은 DUT가 아니라 Testbench이다. DUT만으로는 입력과 출력 신호가 어떤 규칙으로 사용되는지, 동작이 언제 시작되고 종료되는지를 파악하기 어렵다. 특히 내부에 FSM이나 파이프라인 구조가 포함된 경우, 실제 시간 흐름을 이해하기에는 한계가 있다. 반면 Testbench에는 설계자의 사용 의도에 따른 stimulus 형태로 비교적 명확하게 드러난다. 따라서 Testbench는 DUT에 적절한 Input/Control Data를 stimulus 함으로써, 단순한 검증용 코드가 아닌, 사용 가정과 동작 규칙을 보여주는 일종의 사용 설명서에 가깝다. (AMBA AXI는 SoC 내부의 여러 블록을 연결/관리하는 표준 On-Chip BUS 프로토콜로서, 추후 다룰 예정)

​

0️⃣ 전처리/컴파일 지시어

        `timescale 1ns / 1ps                  //시뮬레이션 시간 단위(1ns), 지연 정밀도(1ps) 정의
        `include "..\rtl\include.v"           //네트워크 파라미터 매크로 include
        `define MaxTestSamples 100            // 테스트할 MNIST 샘플 개수

        //include.v에서 정의한 파라미터 
        `define pretrained              // pre-trained 모드 : 합성 시 weight·bias를 ROM에 내장
        `define numLayers 4             // 전체 신경망 레이어 개수 (Hidden + Output Layer)
        `define dataWidth 8             // 입력 데이터 비트 폭 (8-bit 정수 / fixed-point)
        `define numNeuronLayer1 30      // Layer 1의 뉴런 개수
        `define numWeightLayer1 784     // Layer 1의 뉴런 하나 당 weight 수
                                        // MNIST 28×28 = 784 Pixel Input)
        `define Layer1ActType "sigmoid" // Layer 1에서 사용하는 활성화 함수 타입
        `define numNeuronLayer2 20      // Layer 2의 뉴런 개수
        `define numWeightLayer2 30      // Layer 2의 뉴런 하나 당 weight 수(=Layer1 출력 뉴런 수)
        `define Layer2ActType "sigmoid" // Layer 2에서 사용하는 활성화 함수 타입
        `define numNeuronLayer3 10      // Layer 3의 뉴런 개수
        `define numWeightLayer3 20      // Layer 3의 뉴런 하나 당 weight 수(=Layer2 출력 뉴런 수)
        `define Layer3ActType "sigmoid" // Layer 3에서 사용하는 활성화 함수 타입
        `define numNeuronLayer4 10      // Layer 4의 뉴런 개수 (출력층, 0~9 숫자 클래스)
        `define numWeightLayer4 10      // Layer 4의 뉴런 하나 당 weight 수(=Layer3 출력 뉴런 수)
        `define Layer4ActType "hardmax" // 출력층 활성화 함수 (=최댓값 인덱스를 출력시키는 함수)
        `define sigmoidSize 10          // sigmoid LUT Entry (입력 범위 분할 개수)
                                        // fixed-point 연산에 영향
                                        // 실제 sigmoid 연산을 하는 것이 아닌 sigmoid 함수를 모방
        `define weightIntWidth 4        // weight의 정수부 비트 수(전체 weight 비트 폭=정수부+소수부)
                                        // 즉 Q-format을 Q4.4로 정의
                                        //값 범위 : −2^3 ~ 2^3-2^-4 = -8 ~ 7.9375
                                        //해상도(Step Size) : 2^-4 = 0.0625

1️⃣ Module 선언 & 주요 레지스터/와이어 선언

        module top_sim(); 
        //포트가 없는 top-level simulation wrapper로, 내부에서 DUT Instantiation + Stimulus
        
            reg reset;
            reg clock;
            reg [`dataWidth-1:0] in;
            reg in_valid;
            reg [`dataWidth-1:0] in_mem [784:0];
            reg [25*7:0] fileName;
            reg s_axi_awvalid;
            reg [4:0] s_axi_awaddr;
            wire s_axi_awready;
            reg [31:0] s_axi_wdata;
            reg s_axi_wvalid;
            wire s_axi_wready;
            wire s_axi_bvalid;
            reg s_axi_bready;
            wire intr;
            reg [31:0] axiRdData;
            reg [4:0] s_axi_araddr;
            wire [31:0] s_axi_rdata;
            reg s_axi_arvalid;
            wire s_axi_arready;
            wire s_axi_rvalid;
            reg s_axi_rready;
            reg [`dataWidth-1:0] expected;
        
            wire [31:0] numNeurons[31:1];
            wire [31:0] numWeights[31:1];
            
            assign numNeurons[1] = 30;
            assign numNeurons[2] = 20;
            assign numNeurons[3] = 10;
            assign numNeurons[4] = 10;
            
            assign numWeights[1] = 784;
            assign numWeights[2] = 30;
            assign numWeights[3] = 20;
            assign numWeights[4] = 10;
            
            integer right=0;
            integer wrong=0;

- reset/clock → DUT에 들어갈 클럭과 리셋(여기서는 s_axi_aresetn에 reset을 그대로 넣음)을 생성한다. 
- in, in_valid (AXI-Stream 입력) → 픽셀셀을 axis_in_data, axis_in_data_valid로 한 사이클에 한 샘플씩 흘려보내는 스트리밍 입력이다. 
- in_mem[784:0], expected → 앞의 784개는 입력, 마지막 1개는 정답 라벨(expected)로 사용한다.
- fileName → test_data_0000.txt 같은 파일명을 동적으로 만들어 $readmemb에 넘기는 용도다. 
- AXI-Lite Write 채널(s_axi_aw, s_axi_w) → 레이어/뉴런 번호 세팅, weight/bias 쓰기 같은 설정을 AXI-Lite로 DUT에 주입한다. 
- AXI-Lite Read 채널(s_axi_ar, s_axi_r) + axiRdData → 최종 결과나 디버그 출력을 읽어오기 위한 읽기 인터페이스다. 
- intr → 결과 valid 되었음을 알려주는 인터럽트(여기서는 intr posedge를 기다림)다. 
- numNeurons[], numWeights[] 배열 → 레이어별 뉴런 수/가중치 수를 TB가 알고 있도록 하드코딩해서, 설정 루프(for) 범위를 결정한다. 
- right/wrong 카운터 → 테스트 샘플마다 정답 여부를 집계해서 Accuracy를 계산한다.             

2️⃣ DUT(ZyNet) Instantiation

            zyNet dut(
            ....
            );
        //→ AXI-Lite(설정/읽기) + AXI-Stream(입력 스트림) + intr(완료 신호)를 top_sim이 구동/관찰한다.

3️⃣ Initial Condition & Clock Setting

            initial
            begin
                clock = 1'b0;
                s_axi_awvalid = 1'b0;
                s_axi_bready = 1'b0;
                s_axi_wvalid = 1'b0;
                s_axi_arvalid = 1'b0;
            end
        //시뮬레이션 시작 전, AXI valid/ready 관련 신호를 0으로 초기화한다.
           
            always #5 clock = ~clock;
        //10ns period(=100MHz) 클럭 생성

4️⃣ 유틸 함수/핸드셰이크 자동화(https://wikidocs.net/280125)

            function [7:0] to_ascii;
              input integer a;
              begin
                to_ascii = a+48;
              end
            endfunction
        //숫자(0~9)를 문자 '0'~'9'(ASCII)로 바꿔서 파일명 문자열을 조립한다.
            always @(posedge clock)
            begin
                s_axi_bready <= s_axi_bvalid;
                s_axi_rready <= s_axi_rvalid;
            end
        //bvalid가 오면 bready를, rvalid가 오면 rready를 따라가게 해서 
        //AXI-Lite 응답을 “바로바로 받아먹는” 마스터처럼 동작시킨다.

5️⃣ AXI-Lite Transaction Task

            task writeAxi(
            input [31:0] address,
            input [31:0] data
            );
            begin
                @(posedge clock);
                s_axi_awvalid <= 1'b1;
                s_axi_awaddr <= address;
                s_axi_wdata <= data;
                s_axi_wvalid <= 1'b1;
                wait(s_axi_wready);
                @(posedge clock);
                s_axi_awvalid <= 1'b0;
                s_axi_wvalid <= 1'b0;
                @(posedge clock);
            end
            endtask
        //AWVALID/WVALID을 세우고 WREADY 기다린 뒤 내리는 방식으로 “레지스터 쓰기 1회”를 캡슐화한다.
            
            task readAxi(
            input [31:0] address
            );
            begin
                @(posedge clock);
                s_axi_arvalid <= 1'b1;
                s_axi_araddr <= address;
                wait(s_axi_arready);
                @(posedge clock);
                s_axi_arvalid <= 1'b0;
                wait(s_axi_rvalid);
                @(posedge clock);
                axiRdData <= s_axi_rdata;
                @(posedge clock);
            end
            endtask
        //RVALID 세우고 ARREADY 기다린 뒤 RVALID까지 기다려 rdata를 axiRdData에 샘플링하는 
        //“레지스터 읽기 1회” 캡슐화다.

6️⃣ weight/bias setting task

            task configWeights();
            ...
            endtask
            
            task configBias();
            ...
            endtask

<table border="1" bordercolor="#4A90E2" bgcolor="#f0f7ff">
  <tr>
    <td style="padding: 10px;">
      <font size="2">
        <p align="center"><b>[pretrained = ON]</b></p>
        <p align="center">ROM(weight) —▶ MAC —▶ activation</p>
        <br>
        <p align="center"><b>[pretrained = OFF]</b></p>
        <p align="center">AXI —▶ RAM(weight) —▶ MAC —▶ activation</p>
      </font>
    </td>
  </tr>
</table>
