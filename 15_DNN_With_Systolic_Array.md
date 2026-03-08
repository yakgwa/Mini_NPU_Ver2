## [DNN with Systolic Array] PPA 분석 

### Leference Paper : ZyNet : Automating Deep Neural Network Implementation on Low-cost Reconfigurable Edge Computing Platforms

- VIVADO SETTING
    - 본 논문에서는 “All designs are simulated and implemented with Xilinx Vivado 2018.3 version and hardware validated on ZedBoard having an xc7z020clg484-1 SoC and 512MB external DDR3 memory. All implementations use Vivado default settings and do not apply any optimization (timing, area or power).”라고 명시하고 있다. 이는 논문에 제시된 자원 사용량 및 성능 수치가 단순히 설계된 ZyNet DUT(RTL 로직)만을 의미하는 것이 아니라, 실제로는 해당 설계가 탑재된 Zynq-7000(xc7z020clg484-1) SoC 환경 전체를 기준으로 도출된 결과임을 의미한다.
    - Zynq-7000 계열 SoC는 FPGA의 PL(Programmable Logic) 영역과 함께 ARM Cortex-A9 기반의 PS(Processing System)가 하드웨어적으로 통합된 구조를 갖는다. PS 영역에는 ARM CPU, DDR 컨트롤러, USB, Ethernet 등의 주변 회로가 이미 hardening되어 있으며, 이는 사용자가 Verilog로 직접 수정하거나 재구성할 수 없는 고정된 블록이다. 따라서 실제 동작 시에는 PL에서 구현한 사용자 RTL 로직뿐만 아니라, PS 영역이 소모하는 기본 전력(static + system-level power) 역시 함께 존재하게 된다.
    - 반면, PL 영역은 LUT, Flip-Flop, BRAM, DSP 등의 자원을 기반으로 사용자가 설계한 RTL 로직을 프로그래머블하게 구현하는 영역이다. 일반적으로 “순수 RTL 로직의 자원 사용량”이라고 할 때는 이 PL 영역에서 소비되는 자원 및 전력을 의미한다. 그러나 본 논문과 같이 ZedBoard 상에서 실제 SoC 환경으로 검증한 경우, 보고된 수치는 PL 단독이 아닌 PS를 포함한 전체 시스템 레벨 관점의 결과로 해석해야 한다.
    - 따라서 reference model과의 비교를 수행할 때에도 단순히 RTL logic(ZyNet IP)만을 독립적으로 구현한 상태에서의 PL 자원 및 전력만을 기준으로 비교하는 것은 적절하지 않다. 동일한 SoC 환경(PS 설정 포함)을 프로젝트에 포함시켜 시스템 레벨 조건을 맞춘 뒤, 그 위에서 PL 영역의 사용량 및 전체 전력 특성을 비교하는 방식이 보다 타당한 분석 접근이라 할 수 있다.

- 1️⃣ Block Design 생성: Vivado 좌측 메뉴에서 IP Integrator -> Create Block Design 클릭

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_52.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_53.png" width="400"/>

<div align="left">

- 2️⃣  IP 추가: + 버튼을 누르고 ZYNQ7 Processing System을 검색해서 추가

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_54.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_55.png" width="400"/>

<div align="left">

- 3️⃣ Preset 적용: 상단에 뜨는 "Run Block Automation"을 클릭하고, Board Preset 확인 후 OK (이렇게 해야 PS의 DDR 메모리 컨트롤러 등 주변장치 전력이 포함됨)

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_56.png" width="400"/>

<div align="left">

- 4️⃣ Module 추가하기

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_57.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_58.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_59.png" width="400"/>

 Diagram 빈 공간에 우클릭 후 Add Module(TOP DUT 불러오기)

<div align="left">

- 5️⃣ Clock Frequency : 100MHz Setting

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_60.png" width="400"/>

우측 Zynq 더블클릭 이후 왼쪽 메뉴에서 세팅

<div align="left">

- 6️⃣ Run Connection Automation (자동 연결이 안 뜨는 핀들은 수동으로 연결)

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_61.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_62.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_63.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_64.png" width="400"/>

아래 핀들을 선택 후 마우스 우클릭하여 Make External

<div align="left">

    - axis_in_data[7:0]
    - axis_in_data_valid
    - axis_in_data_ready
    - intr

- 본 논문의 시스템은 PS 영역에서 ARM CPU가 처리할 데이터를 DDR Memory에 올려놓고, PS(CPU)가 DMA Controller에게 명령하여 FPGA에게 메모리를 전달하라는 명령을 한다.(편의상 DMA는 구현하지 않고 CPU 블록 자체만 켜져 있는 상태로 PPA 분석 예정)
- 명령을 받은 DMA는 메모리에서 데이터를 읽어서 AXI Stream을 통해 FPGA에게 전달해준다. 
- 이렇게 받은 데이터를 PL(FPGA)에서 실시간 처리한다.
- 처리한 결과는 다시 AXI Stream을 통해 DMA로 보낸다.
- DMA가 결과값을 다시 DDR Memory에 쓰고, CPU에게 Interrupt 신호를 보낸다.

- 7️⃣ Validate Design : 'Successful' Check

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_65.png" width="400"/>

<div align="left">

- 8️⃣ Create HDL Wrapper

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_66.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_67.png" width="400"/>

OK 클릭 후 생성된 .bd 파일을 'set as TOP'으로 지정해야 함

<div align="left">

- 9️⃣ .xdc 설정 (gemini) 후 Run Implementation

        ## =========================================================
        ## 1. I/O Standard 설정 (모든 외부 핀 LVCMOS33)
        ## =========================================================
        
        # 제어 신호 입력
        set_property IOSTANDARD LVCMOS33 [get_ports i_start_inference_0]
        set_property IOSTANDARD LVCMOS33 [get_ports i_input_valid_0]
        
        # 데이터 입력 (32-bit)
        set_property IOSTANDARD LVCMOS33 [get_ports {i_input_pixels_0[*]}]
        
        # 출력 신호 (인터럽트 및 결과 데이터)
        set_property IOSTANDARD LVCMOS33 [get_ports o_done_interrupt_0]
        set_property IOSTANDARD LVCMOS33 [get_ports {o_result_class_0_0[*]}]
        set_property IOSTANDARD LVCMOS33 [get_ports {o_result_class_1_0[*]}]
        set_property IOSTANDARD LVCMOS33 [get_ports {o_result_class_2_0[*]}]
        set_property IOSTANDARD LVCMOS33 [get_ports {o_result_class_3_0[*]}]
        
        ## =========================================================
        ## 2. Pin Location (입력 핀 고정)
        ## ZedBoard의 JA, JB, JC, JD 헤더 핀을 입력 데이터에 할당
        ## =========================================================
        
        # JA Header (Lower 8 bits)
        set_property PACKAGE_PIN Y9  [get_ports {i_input_pixels_0[0]}]
        set_property PACKAGE_PIN AA9 [get_ports {i_input_pixels_0[1]}]
        set_property PACKAGE_PIN Y10 [get_ports {i_input_pixels_0[2]}]
        set_property PACKAGE_PIN AA11 [get_ports {i_input_pixels_0[3]}]
        set_property PACKAGE_PIN AB11 [get_ports {i_input_pixels_0[4]}]
        set_property PACKAGE_PIN AB10 [get_ports {i_input_pixels_0[5]}]
        set_property PACKAGE_PIN AB9 [get_ports {i_input_pixels_0[6]}]
        set_property PACKAGE_PIN AA8 [get_ports {i_input_pixels_0[7]}]
        
        # JB Header (Next 8 bits)
        set_property PACKAGE_PIN W12 [get_ports {i_input_pixels_0[8]}]
        set_property PACKAGE_PIN W11 [get_ports {i_input_pixels_0[9]}]
        set_property PACKAGE_PIN V10 [get_ports {i_input_pixels_0[10]}]
        set_property PACKAGE_PIN W8  [get_ports {i_input_pixels_0[11]}]
        set_property PACKAGE_PIN V12 [get_ports {i_input_pixels_0[12]}]
        set_property PACKAGE_PIN W10 [get_ports {i_input_pixels_0[13]}]
        set_property PACKAGE_PIN V9  [get_ports {i_input_pixels_0[14]}]
        set_property PACKAGE_PIN V8  [get_ports {i_input_pixels_0[15]}]
        
        # JC Header (Next 8 bits)
        set_property PACKAGE_PIN AB6 [get_ports {i_input_pixels_0[16]}]
        set_property PACKAGE_PIN AB7 [get_ports {i_input_pixels_0[17]}]
        set_property PACKAGE_PIN AA4 [get_ports {i_input_pixels_0[18]}]
        set_property PACKAGE_PIN Y4  [get_ports {i_input_pixels_0[19]}]
        set_property PACKAGE_PIN T6  [get_ports {i_input_pixels_0[20]}]
        set_property PACKAGE_PIN R6  [get_ports {i_input_pixels_0[21]}]
        set_property PACKAGE_PIN U4  [get_ports {i_input_pixels_0[22]}]
        set_property PACKAGE_PIN T4  [get_ports {i_input_pixels_0[23]}]
        
        # JD Header (Upper 8 bits)
        set_property PACKAGE_PIN W7  [get_ports {i_input_pixels_0[24]}]
        set_property PACKAGE_PIN V7  [get_ports {i_input_pixels_0[25]}]
        set_property PACKAGE_PIN V4  [get_ports {i_input_pixels_0[26]}]
        set_property PACKAGE_PIN V5  [get_ports {i_input_pixels_0[27]}]
        set_property PACKAGE_PIN W5  [get_ports {i_input_pixels_0[28]}]
        set_property PACKAGE_PIN W6  [get_ports {i_input_pixels_0[29]}]
        set_property PACKAGE_PIN U5  [get_ports {i_input_pixels_0[30]}]
        set_property PACKAGE_PIN U6  [get_ports {i_input_pixels_0[31]}]
        
        # [입력 2] 제어 신호 (보드의 스위치 SW0, SW1 사용)
        set_property PACKAGE_PIN F22 [get_ports i_start_inference_0]
        set_property PACKAGE_PIN G22 [get_ports i_input_valid_0]
        
        # [출력 1] 인터럽트 신호 (보드의 LED LD0 사용)
        set_property PACKAGE_PIN T22 [get_ports o_done_interrupt_0]
        
        ## =========================================================
        ## 3. Operating Conditions
        ## =========================================================
        set_operating_conditions -grade industrial
        set_operating_conditions -process typical
        set_switching_activity -deassert_resets

- NPU_TOP.v도 동일한 방법을 적용

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_68.png" width="400"/>

<div align="left">

### AREA

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_69.png" width="400"/>

<div align="left">

- 먼저 면적(Area) 관점에서 분석을 수행한다. 위 제시된 바와 같이, 논문에서 보고된 Resource Utilization을 실제 Vivado 환경에서 직접 합성하여 추출한 결과와 비교하고자 한다. 본 실험에서는 sigmoidSize를 10으로 설정하였으므로, 논문 내에서 해당 설정에 대응하는 오른쪽 (c) 그래프의 자원 사용량 수치를 기준으로 삼는다.
- 아래에 제시된 값은 Reference Model을 동일한 조건에서 합성하여 얻은 Resource Utilization이다. LUT, FF, BRAM, DSP 등의 자원 사용량은 논문에 제시된 수치와 완벽히 일치하지는 않으나, 이는 synthesis strategy의 세부 설정, implementation seed, constraint 적용 여부 등 측정 방식의 차이에 기인한 오차로 해석된다. 단, 전체적인 자원 사용 경향은 논문과 근접하게 나타났으며, 구조적 차이를 설명하기에 통계적으로 유의미한 비교 기준으로 활용할 수 있다고 판단한다. 따라서 직접 추출한 Reference Model의 Resource Utilization을 비교 변인으로 설정 후, 제안 구조와의 Area 차이를 분석한다.

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_70.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_71.png" width="400"/>

Reference Model : Area

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_72.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_73.png" width="400"/>

Proposed Model : Area

<div align="left">

- 현재 제안 모델은 AXI Interface가 구현되지 않은 상태에서, 순수 RTL 로직만 합성된 결과에 해당한다. 이 경우 설계는 내부 레지스터 기반 연산 구조만을 사용하며, 대용량 데이터를 저장하거나 버퍼링하기 위한 메모리 블록을 포함하지 않는다. 따라서 합성 과정에서 BRAM 사용량이 0으로 나타난 것이다. 반면, I/O 사용량이 65%로 매우 높게 나타난 것은 설계 구조상 외부 포트로 노출된 신호 수가 많기 때문이다. AXI가 아직 연결되지 않은 상황에서는 대부분 신호가 top module의 개별 포트 형태로 직접 외부에 연결된다. FPGA에서 이러한 top-level 포트는 물리적 패키지 핀으로 매핑되며, Vivado는 이를 I/O 사용량으로 집계한다.

### POWER​

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_74.png" width="400"/>

<div align="left">

- 아래의 Reference Model의 총 On-Chip Power는 1.893W로 나타났다. 위 제시된 논문의 내용과 역시 다소 오차가 존재하지만, 이는 설정 차이에 기인한 것으로 분석된다. 이 중 Dynamic Power 1.742W로 전체의 92%를 차지하며, Static Power 0.151W로 8% 수준이다. 동적 전력의 세부 항목 중 PS7으로 1.572W, 즉 동적 전력의 약 90%에 해당한다. 반면, PL에 해당하는 합산 전력은 약 0.17W 수준에 해당한다. 리포트에 표시된 Confidence level이 Low로 나타난 점은 전력 추정이 정확한 시뮬레이션 기반 토글 데이터(SAIF/VCD)가 아니라 vectorless 분석에 기반하여 산출되어 신뢰도는 다소 낮을 수 있다. 

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_75.png" width="400"/>

Reference Model : Power

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_76.png" width="400"/>

Proposed Model : Power

<div align="left">

### Performance : Timing & Maximum Frequency

- Reference Model의 100MHz Clock Frequency에서 Setup, Hold, Pulse Width 세 가지 조건 모두 충족되었음을 확인할 수 있다. 먼저 Setup 분석에서 WNS이 1.329ns로 나타났으며, TNS는 0.000ns, Failing Endpoints 역시 0으로 보고되어, 모든 경로가 요구된 클럭 주기 내에 안정적으로 데이터 전이를 완료하고 있음을 의미한다. 특히 WNS가 양수라는 점은 클럭 주기 대비 여유 마진이 존재함을 보여준다. 최대 동작 주파수는 이 Setup Slack을 이용하여 계산할 수 있다. 일반적으로 WNS는 '요구 클럭 주기 - 실제 경로 지연'의 차이를 의미하므로, 현재 설계에서 설정한 클럭 주기를 T라 하면, 다음과 같이 계산할 수 있다.



