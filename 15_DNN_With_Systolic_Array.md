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

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_7.png" width="400"/>

OK 클릭 후 생성된 .bd 파일을 'set as TOP'으로 지정해야 함

<div align="left">












