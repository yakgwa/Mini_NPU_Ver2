## [DNN with Systolic Array] 문제점 및 개선 방안 제시 

### 최초 문제점 파악 및 개선 방안 분석 및 적용

- 1️⃣ Weight 값 Unknown 'x' 현상 분석
    - 초기화 단계에서 메모리 적재의 정합성을 점검하기 위해 MIF Loading Verification을 수행하였다. 테스트벤치 설정에 따라 Layer 1, Neuron 0의 초기 5개 Weight와 Layer 1의 초기 4개 Bias, 그리고 각 샘플 의 초기 픽셀 및 정답 라벨을 출력하여 데이터를 확인하였다. 1차 검증 결과, Bias 및 샘플 데이터는 정상적으로 확인되었으나, Weight는 전 구간에서 x (Unknown) 상태로 출력되는 문제가 발견되었다.

          ============================================================
            MIF Loading Verification
          ============================================================
            [Weight] L1_N0 w[0]=   x, w[1]=   x, w[2]=   x, w[3]=   x, w[4]=   x
            [Bias]   L1: b[0]=   7, b[1]=  -8, b[2]= -10, b[3]=   7
            [Image0] px[0]=   0, px[1]=   0, px[2]=   0, px[3]=   0, px[4]=   0, label=  7
            [Image1] px[0]=   0, label=  2
            [Image2] px[0]=   0, label=  1
            [Image3] px[0]=   0, label=  0
            MIF Loading Done.
          ============================================================

    - 나머지 데이터가 0으로 나오는 현상이 실제 데이터인지 로딩 실패인지 명확히 구분하고, 가중치 문제를 특정하기 위해 조건을 변경하여 2차 검증을 수행하였다. 

            ============================================================
              MIF Loading Verification
            ============================================================
              [Weight] L1_N0 w[100]=   x, w[101]=   x, w[102]=   x, w[103]=   x, w[104]=   x
              [Bias]   L1: b[5]= -18, b[6]= -12, b[7]= -11, b[8]=  14
              [Image0] px[202~206]=  42, 92, 79, 75, 30, label=  7
              [Image1] px[202]=   0, label=  2
              [Image2] px[202]=   0, label=  1
              [Image3] px[202]=   0, label=  0
              MIF Loading Done.
            ============================================================

    - 가중치는 101..105번 인덱스, 편향은 6..9번 인덱스, 픽셀 데이터는 유효 값이 존재하는 203..207번 인덱스를 확인하였다. 그 결과, 변경된 구간의 값이 실제 파일의 값과 일치하게 로드됨을 확인하였다. 이를 통해 샘플 데이터와 Bias는 정상적으로 초기화되었음이 입증되었으나, Weight는 여전히 x 상태로 출력되고 있어 Weight Bank의 초기화 또는 파일 로딩 과정인지, 단순 검증 실수 중의 하나라고 판단하였다.

    - 🚀 [개선 방안] Testbench 모니터링 포인트 수정 및 데이터 정합성 검증
      - 초기 시뮬레이션 단계에서 발생한 문제는 TB 내 신호 Observation Point가 부적절한 것으로 판단되었다. 최초 검증 코드는 상위 모듈인 Weight_Bank의 출력 인터페이스(wire 타입)를 모니터링하였으나, 해당 신호는 현재 활성화된 뉴런의 출력값만을 전달하는 연결선일 뿐 데이터를 저장하는 공간이 아니었다. 특히, 선언된 와이어 배열의 인덱스 범위를 초과하여 접근함으로써 시뮬레이터가 유효하지 않은 신호를 반환한 것으로 분석되었다. 이에 대해 RTL Hierarchy를 파악하여 Sub-instantiation인 Weight_Memory 인스턴스 내부 reg mem에 직접 접근하여 해결하였다.

            NPU_Top
            │
            ├─ Data_Skewing (ds)
            │  └─ [Internal Shift Registers for Row/Col skewing]
            │
            ├─ Systolic_Array (sa)
            │  └─ pe_systolic_cell [4][4]  (16 instances)
            │     ├─ a_reg, b_reg (1-cycle pipeline registers)
            │     └─ acc (accumulator)
            │
            ├─ Weight_Bank (wb)
            │  └─ Weight_Memory [4]  (4 instances, one per column)
            │     └─ RAM blocks initialized from MIF files
            │
            ├─ Bias_Bank (bb)
            │  └─ Bias_Memory [4]  (4 instances)
            │     └─ RAM blocks initialized from MIF files
            │
            ├─ Activation_Unit (au)
            │  ├─ Sig_ROM (sig_inst)
            │  │  └─ ROM[1024]  (Sigmoid lookup table)
            │  └─ ReLU (relu_inst)
            │
            ├─ Unified_Buffer (ub)
            │  └─ Dual-port SRAM [128]  (4 read ports, 1 write port)
            │
            └─ maxFinder [4]  (mf_inst_0~3)
               └─ Combinational max-finding logic


            // Weight Bank: Layer 1, Neuron 0의 첫 5개 가중치 샘플 출력
            $fdisplay(log_fd, "  [Weight] L1_N0 w[202]=%d, w[203]=%d, w[204]=%d, w[205]=%d, w[206]=%d",
                $signed(dut.wb.wm_1_00.mem[202]),
                $signed(dut.wb.wm_1_00.mem[203]),
                $signed(dut.wb.wm_1_00.mem[204]),
                $signed(dut.wb.wm_1_00.mem[205]),
                $signed(dut.wb.wm_1_00.mem[206]));

            ============================================================
              MIF Loading Verification
            ============================================================
              [Weight] L1_N0 w[0]=  -2, w[1]=   0, w[2]=   2, w[3]=   1, w[4]=   3
              [Bias]   L1: b[0]=   7, b[1]=  -8, b[2]= -10, b[3]=   7
              [Image0] px[0]=   0, px[1]=   0, px[2]=   0, px[3]=   0, px[4]=   0, label=  7
              [Image1] px[0]=   0, label=  2
              [Image2] px[0]=   0, label=  1
              [Image3] px[0]=   0, label=  0
              MIF Loading Done.
            ============================================================

- 2️⃣ Input/Weight 입력 타이밍 분석 및 관련 문제점 파악
  1) 초기화 및 시작 트리거 (Cycle 0 ~ 1) :
     - Cycle 0에서는 상태가 IDLE로 설정되며 모든 카운터가 0으로 초기화되고, Cycle 1에서 IDLE 상태를 유지한 채 i_start_inference=1이 되면서 본격 구동된다. 이때, 입력 데이터 길이(cur_input_len)는 784, 뉴런 수(cur_neuron_total)는 30으로 설정되며, 다음 클럭에서 CALC_L1 상태로 전환된다.
  2) 상태 진입 및 데이터 요청 (Cycle 2: Setup Phase) : 
     - Cycle 2가 되면 CALC_L1으로 진입한다. 이 시점이 상태상으로는 시작이지만, 하드웨어적으로는 '연산 준비 및 요청' 단계에 해당된다. k_cnt=0인 상태에서 메모리를 향해 "0번지 주소의 데이터를 달라"는 요청(Addr <= 0)을 보내고, 동시에 연산 유닛을 킨다.(pe_en <= 1). 하지만 이 동작들은 Rising Edge에 트리거 되는 명령이기에 실제 로그 상 아직 데이터가 도착하지 않았고 PE도 활성화되지 않는 상태이다.

            ############ STATE CHANGE:  --> CALC_L1 (Cycle 2, Time 125000) ############
            ====== [Cycle:    2] State: CALC_L1 | k_cnt:   0 | Group: 0 | pe_en:0 | pe_rst:1 ======

  3) 실질적 연산 수행 (Cycle 3: Execution Phase) : 
     - 따라서 데이터 처리는 Cycle 3부터 시작된다. 앞선 Cycle 2에서 요청했던 데이터가 메모리로부터 도착하여 Raw Data가 출력되고, pe_en=1로 반영되어 실질적인 MAC 연산을 수행하게 된다.

            ====== [Cycle:    3] State: CALC_L1 | k_cnt:   1 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=   0
              [Wavefront Entry Pattern]
                                       C0=   0  C1=   0  C2=   0  C3=   0  <-- Weight (Col)
                                       |       |       |       |
                                       v       v       v       v
              R0=   0  ---------->  [PE00]  [PE01]  [PE02]  [PE03]
              R1=   0  ---------->  [PE10]  [PE11]  [PE12]  [PE13]
              R2=   0  ---------->  [PE20]  [PE21]  [PE22]  [PE23]
              R3=   0  ---------->  [PE30]  [PE31]  [PE32]  [PE33]
              [PE Accumulator Matrix] (N = Neuron 0~3)
                              N 0        N1         N2         N3 
              Img0(R0) --> [      0] [      0] [      0] [      0]
              Img1(R1) --> [      0] [      0] [      0] [      0]
              Img2(R2) --> [      0] [      0] [      0] [      0]
              Img3(R3) --> [      0] [      0] [      0] [      0]

     - 가장 먼저 데이터가 제대로 입력되고 있는지 확인하기 위해 검증 포인트를 재설정했다. Cycle 3부터 0번째 Pixel과 Weight가 인가되지만, 초기 데이터의 대부분이 0인 관계로 최초로 0이 아닌 유효 값이 나타나는 시점을 찾아냈다. w_1_0.mif 파일 분석 결과, Layer 1 Neuron 0의 44번째 데이터(mem[44])가 1임을 확인했고, 이 값을 TB 모니터링 로그 상에서 정확히 언제 관측되는지 우선적으로 확인해야 한다.

            ============================================================
              MIF Loading Verification
            ============================================================
              [Weight] L1_N0 w[44]=   1

     - Cycle 3에 0번째 데이터가 인가되므로, 44번째 데이터는 3 + 44 = 47, 즉 Cycle 47에 나타날 것으로 예측된다. 따라서 1차적으로는 Cycle 47의 로그를 중점적으로 확인하였다.

            ====== [Cycle:   47] State: CALC_L1 | k_cnt:  45 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=   0
            
            ====== [Cycle:   48] State: CALC_L1 | k_cnt:  46 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   1  C1=   0  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   1  C1=   0  C2=   0  C3=   0

     - 로그 분석 결과, 예상했던 Cycle 47이 아닌 1 Cycle 지연된 시점(Cycle 48)에 데이터가 관측되었다. 이는 하드웨어의 Cycle-by-Cycle 데이터 전달 메커니즘을 통해 설명할 수 있다.
     1) Cycle 47 (데이터 도달) : Rising 직후, Data Skewing을 거쳐 PE의 i_a, i_b 도달한다. 하지만 이 시점은 데이터가 포트에만 와있을 뿐, 아직 내부 a_reg에는 저장되지 않은 상태이다.
     2) Cycle 48 (데이터 캡처) : Rising 순간에 pe_systolic_cell에 대기 중이던 데이터가 a_reg, b_reg로 Capture된다. 이 시점부터 PE가 데이터를 인식하게 되며, 로그 상에서도 유효 데이터로 출력된다.
     - 이러한 현상은 입력 레지스터(a_reg)를 활용한 Pipelining 설계의 특징이다. assign o_a = a_reg; 구문을 통해 현재 사이클에 캡처한 데이터를 다음 사이클에 우측 PE로 전달한다. 이는 데이터 전달과 MAC을 안정적으로 분리하여 Critical Path를 최적화하고 Fmax를 확보하기 위함이다.

### 분석 : 구체적인 타이밍 검증1

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_46.png" width="400"/>

각 열 별로 왼쪽은 test_data_0000..0003.txt value / 오른쪽은 w_1_0..3.mif value에 해당한다

<div align="left">

- 1️⃣ 검증 기준점 설정 : test_data_0000..0003.txt과 w_1_0..3.mif의 0-indexed 기준 69번째 데이터를 검증 포인트로 잡는다. 앞서 분석한 4 Cycle(초기화+메모리 읽기+레지스터링)을 고려할 때, 해당 데이터는 Cycle 73부터 관측되어야 한다.
  
- 2️⃣ Raw Data : 만약, Weight가 읽혀 나온 직후 Raw Data는 Skewing 없이 모든 행이 동시에 출력되어야 한다. 따라서 위 그림에서 69번째 줄부터의 내용이 Cycle 73부터 순차적으로 찍혀야 한다.
    - Cycle 73: 0, 1, 0, 0 (69번째 데이터)
    - Cycle 74: -2, 0, 0, 0 (70번째 데이터)
    - Cycle 75: -2, -1, 0, 0 (71번째 데이터)
    - Cycle 76: 0, 0, 0, -1 (72번째 데이터)
    - Cycle 77: 0, 0, 0, -1 (73번째 데이터)
    - 
- 3️⃣ Skewed Data : Data Skewing 모듈을 통과한 각 Row마다 인덱스만큼의 지연(Row 0: 0 delay, Row 1: 1 delay...)이 반영된 형태로 입력되어야 한다. 이를 Cycle 73 기준으로 재배열하면 다음과 같다.
    - Cycle 73: 0, 0, 0, -1
    - Cycle 74: -2, 1, 0, -1
    - Cycle 75: -2, 0, 0, 0
    - Cycle 76: 0, -1, 0, 0
    - Cycle 77: 0, 0, 0, 0

            ====== [Cycle:   73] State: CALC_L1 | k_cnt:  71 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   1  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=  -1
            
            ====== [Cycle:   74] State: CALC_L1 | k_cnt:  72 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=  -2  C1=   0  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=  -2  C1=   1  C2=   0  C3=  -1
            
            ====== [Cycle:   75] State: CALC_L1 | k_cnt:  73 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=  -2  C1=  -1  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=  -2  C1=   0  C2=   0  C3=   0
            
            ====== [Cycle:   76] State: CALC_L1 | k_cnt:  74 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=  -1
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=  -1  C2=   0  C3=   0

### 분석 : 구체적인 타이밍 검증2

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_47.png" width="400"/>

<div align="left">

- 1️⃣ 검증 기준점 재설정 (Index 202) : 앞선 테스트 구간보다 Pixel과 Weight가 0이 아닌 유효 값으로 다양하게 분포된 202번째 데이터를 새로운 검증 포인트로 선정해보자. Sparse한 데이터가 아닌 실제 연산 값이 몰리는 구간을 확인함으로써, Corner Case에서의 동작 안정성을 확보하도록 한다.
  
- 2️⃣ 타겟 사이클 계산 (Cycle 206) : 앞서 고려된 4 Cycle를 적용하여 모니터링 시점을 계산한다.
    - Target Index : 202번째 데이터
    - System Latency : +4 Cycle
    - Verification Point : 202 + 4 = 206 Cycle

            ====== [Cycle:  206] ======
              [Skewed Data (Array Edge)]
                Row: R0=  42  R1=   0  R2=   0  R3=   0
                Col: C0=  -2  C1=   0  C2=   0  C3=   0
              [Wavefront Entry Pattern]
                                       C0=  -2  C1=   0  C2=   0  C3=   0  <-- Weight (Col)
                                       |       |       |       |
                                       v       v       v       v
              R0=  42  ---------->  [PE00]  [PE01]  [PE02]  [PE03]
              R1=   0  ---------->  [PE10]  [PE11]  [PE12]  [PE13]
              R2=   0  ---------->  [PE20]  [PE21]  [PE22]  [PE23]
              R3=   0  ---------->  [PE30]  [PE31]  [PE32]  [PE33]

- 3️⃣ 현재 상태: 입력 레지스터 대기 시점에서 데이터는 pe_systolic_cell.v a_reg에 Latch된 상태일 뿐, 데이터가 PE 셀의 문턱은 넘었으나, 실제 연산을 수행하는 PE.v의 Combinational Logic을 통과하여 결과가 확정된 상태는 아닌 상황이다.

        ====== [Cycle:  207] ======
          [Skewed Data (Array Edge)]
            Row: R0=  92  R1=   0  R2=   0  R3=   0
            Col: C0=   0  C1=   0  C2=   1  C3=   0
          [Wavefront Entry Pattern]
                                   C0=   0  C1=   0  C2=   1  C3=   0  <-- Weight (Col)
                                   |       |       |       |
                                   v       v       v       v
          R0=  92  ---------->  [PE00]  [PE01]  [PE02]  [PE03]
          R1=   0  ---------->  [PE10]  [PE11]  [PE12]  [PE13]
          R2=   0  ---------->  [PE20]  [PE21]  [PE22]  [PE23]
          R3=   0  ---------->  [PE30]  [PE31]  [PE32]  [PE33]

- 4️⃣ 연산 수행 (Execution Phase, +1 Cycle) : 실질적인 연산은 다음 사이클에서 수행된다. a_reg에 저장된 값들이 비로소 PE.v 내부로 전달되어 MAC 연산을 수행한다. 이 과정은 한 클럭 주기 동안 이루어지며, 연산 결과는 다음 Rising Edge에 반영된다. 

        ====== [Cycle:  208] ======
          [Skewed Data (Array Edge)]
            Row: R0=  79  R1=  38  R2=   0  R3=   0
            Col: C0=   2  C1=   3  C2=   3  C3=  -2
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [    -84] [      0] [      0] [      0]
          Img1(R1) --> [   3353] [   1206] [  -3230] [  -7682]
          Img2(R2) --> [   1425] [   -664] [   -835] [   -488]
          Img3(R3) --> [    782] [    314] [  -8273] [  -2056]

- 5️⃣ 결과 반영 (Update Phase, +2 Cycle) : 연산이 완료된 값은 해당 Rising Edge에 Acc 레지스터에 업데이트된다. 따라서, 현재 시점을 기준으로 +1 Cycle 뒤에 연산이 완료되고, 다시 +1 Cycle이 지난 시점(총 2 Cycle 후)이 되어야 비로소 Accumulator Matrix에서 갱신된 값(-84)을 확인할 수 있다.

                                   C0=2         C1=3         C2=3       C3=-2  
                                   |            |             |          |
                                   v            v             v          v
          R0=  79  ---------->  [PE00] 92 ->  [PE01] 42 -> [PE02] 0 -> [PE03]

- 6️⃣ 이에 따라 위와 같은 데이터 입력에 따라 2 Cycle 이후에는 다음과 같은 PE 값이 형성되어야 한다.

        //예측
        PE[0][0] = -84 + 79x2   = 74
        PE[0][1] =  0  + 92x3   = 276
        PE[0][2] =  0  + 42X3   = 128
        PE[0][3] =  0  + 0X(-2) = 0

        //실측
        ====== [Cycle:  209] ======
          [Skewed Data (Array Edge)]
            Row: R0=  75  R1= 125  R2=   0  R3=   0
            Col: C0=   1  C1=   0  C2=   8  C3=   0
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [    -84] [      0] [      0] [      0]
          Img1(R1) --> [   3353] [   1206] [  -3230] [  -7682]
          Img2(R2) --> [   1425] [   -664] [   -835] [   -488]
          Img3(R3) --> [    782] [    314] [  -8273] [  -2056]
        
        ====== [Cycle:  210] ======
          [Skewed Data (Array Edge)]
            Row: R0=  30  R1= 105  R2=   0  R3=   0
            Col: C0=   3  C1=   1  C2=   4  C3=  -1
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [     74] [    276] [    126] [      0]
          Img1(R1) --> [   3353] [   1206] [  -3230] [  -7682]
          Img2(R2) --> [   1425] [   -664] [   -835] [   -488]
          Img3(R3) --> [    782] [    314] [  -8273] [  -2056]

### 분석 : 구체적인 타이밍 검증3 (Group 0, Layer 1 최초 종료 시점)

- 1️⃣ 검증 목표
    - 대상: 4 Sample에 대한 Layer 1 Neuron 0~3의 최종 누적 결과값 확인.
    - 시점: CALC_L1 상태가 최초 종료되는 마지막 사이클.
- 2️⃣ 종료 사이클 계산은 로직에 따라 CALC_L1 상태의 유지 기간을 계산하면 다음과 같다.
    - 초기 오버헤드 (Idle Latency) : 시스템 리셋 및 시작 트리거 대기: 2 Cycle
    - k_cnt 조건 : 784 - 1 + 2*(4-1) = 789
- 3️⃣ 검증 포인트는 Cycle 791 시점에 다음 동작이 올바르게 수행되는지 확인해야 한다.
    - Last Accumulation: 마지막 입력에 대한 MAC 연산이 수행되고, Accumulator가 최종 결과값을 확인
    - Control Signal: pe_en 등의 활성화 신호가 의도대로 Low로 떨어지는지 확인.

            ====== [Cycle:  791] State: CALC_L1 | k_cnt: 789 | Group: 0 | pe_en:1 | pe_rst:0 ======
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=   0
              [PE Accumulator Matrix] (N = Neuron 0~3)
                              N 0        N1         N2         N3 
              Img0(R0) --> [  -1201] [  -1726] [   4829] [  -4097]
              Img1(R1) --> [  18004] [   1232] [ -28361] [ -32768]
              Img2(R2) --> [   8248] [   2265] [      5] [    884]
              Img3(R3) --> [    914] [  -5433] [ -15979] [ -13389]

    - 하지만, 여기서 간과한 점이 있다. 현재는 마지막 데이터(Layer1 Neuron 0~3)가 모두 0이라 결과에 문제가 없어 보이지만, 이는 데이터 특성에 의해 가려진 것일 뿐 실질적인 타이밍 설계에는 오류가 있다. 구체적으로 로그를 확인한 결과, 다음 두 가지 지연 요소가 종료 시점에 제대로 반영되지 않았음을 확인했다.
 
      1) Pipelining Delay: pe_systolic_cell.v 내부의 a_reg를 거치며 발생하는 1 Cycle 지연.
      2) Data Skew: 각 행(Row)마다 데이터가 순차적으로 밀려서 도달하는 Skew Delay. 설정된 종료 시점(Cycle 791)은 Corner를 커버하기에 부족하므로 종료 타이밍을 재설계할 필요가 있다.

    - 🚀 [개선 방안] Pipelining 및 Skew Data를 고려한 k_cnt limit 최적화

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_48.png" width="400"/>

가장 마지막 값이 입력되는 시점은 790 Cycle

<div align="left">

- 최종 데이터 도달 및 연산 종료 시점 분석 결과, Row/Col 3의 마지막 값이 PE에 도달하는 시점은 Cycle 790으로 확인되었다. MAC 연산까지 고려하면 각 Col별 종료 시점은 다음과 같이 순차적으로 지연된다.
    - Col 0: Cycle 792
    - Col 0: Cycle 793
    - Col 0: Cycle 794
    - Col 3: Cycle 795 (최종 종료)

- 이를 바탕으로 CALC_L1 상태의 카운터(k_cnt) 종료 조건을 재설정하였다. (Idle 2 cycle 제외 기준)

        k_cnt == cur_input_len + 3*(ARRAY_ROW - 1)
        // 784 + 3*(4 - 1) = 784 + 3*3 = 784 + 9 = 793

- ​추가 문제 발생: Data Skewing의 입력 조기 차단 카운터 조건을 확장했음에도 불구하고, 마지막 데이터(강제로 0이 아닌 값 입력) 결과에 반영되지 않는 문제가 발생했다. 원인은 Data_Skewing 모듈로 들어가는 입력 신호(sys_row_in, sys_col_in)의 Valid 조건에 있었다.

            assign sys_row_in = (k_cnt < cur_input_len) ? raw_input_data : {ARRAY_ROW*dataWidth{1'b0}};
            assign sys_col_in = (k_cnt < cur_input_len) ? raw_weight_data : {ARRAY_COL*dataWidth{1'b0}};

- 기존 로직은 k_cnt=784(cur_input_len)에 입력을 0으로 차단했다. 하지만 Skewing이 데이터를 지연시켜 내보내려면, 지연되는 시간만큼 입력 데이터가 유효하게 유지되어야 한다는 점을 고려해야 한다.​

            assign sys_row_in = (k_cnt < cur_input_len + 2) ? raw_input_data : {ARRAY_ROW*dataWidth{1'b0}};
            assign sys_col_in = (k_cnt < cur_input_len + 2) ? raw_weight_data : {ARRAY_COL*dataWidth{1'b0}};

- 입력 유효 구간(Valid Window) 확장 Skewing 모듈이 지연된 데이터를 충분히 확보할 수 있도록, 입력 할당문의 조건을 +2 사이클만큼 확장하여 수정하였다.(아래의 7과 31은 임의로 .txt, .mif의 마지막 데이터를 변경하여 검증을 수행한 것이다. 검증 결과, 문제가 개선된 것을 확인할 수 있다.)

        ====== [Cycle:  790] State: CALC_L1 | k_cnt: 788 | Group: 0 | pe_en:1 | pe_rst:0 ======
          [Skewed Data (Array Edge)]
            Row: R0=   0  R1=   0  R2=   0  R3=   7
            Col: C0=   0  C1=   0  C2=   0  C3=  31
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [  -1201] [  -1726] [   4829] [  -4097]
          Img1(R1) --> [  18004] [   1232] [ -28361] [ -32768]
          Img2(R2) --> [   8248] [   2265] [      5] [    884]
          Img3(R3) --> [    914] [  -5433] [ -15979] [ -13389] (실제로는 이 값이 맞는 값임)
        
        ...
        
        ====== [Cycle:  796] State: CALC_L1 | k_cnt: 794 | Group: 0 | pe_en:1 | pe_rst:0 ======
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [  -1201] [  -1726] [   4829] [  -4097]
          Img1(R1) --> [  18004] [   1232] [ -28361] [ -32768]
          Img2(R2) --> [   8248] [   2265] [      5] [    884]
          Img3(R3) --> [    914] [  -5433] [ -15979] [ -13172]

- 3️⃣ FSM State Transition Timing & Golden Model 비교
    - 최종 MAC 결과를 BUFFER_WR_L1 state에서 Unified Buffer에 저장해야 한다. 그 전에, Golden Model에서 Testbench를 재작성하여 각 sample 별 Layer 1 출력 결과 중 일부를 살펴보자.

            ====== [Image 0] LAYER 1 OUTPUT (Cycle: 807) ======
                N[ 0- 4]:   72    17    95    30   
            
            ====== [Image 1] LAYER 1 OUTPUT (Cycle: 1735) ======
                N[ 0- 4]:  127    51     0     0   
            
            ====== [Image 2] LAYER 1 OUTPUT (Cycle: 2663) ======
                N[ 0- 4]:  127    66    28   100  
            
            ====== [Image 3] LAYER 1 OUTPUT (Cycle: 3591) ======
                N[ 0- 4]:  100     3     0     0     

    - 저장 결과에 대한 출력 로그는 다음과 같다.

            ############ STATE CHANGE:  --> BUFFER_WR_L1 (Cycle 796, Time 8065000) ############
              [BUF_WR] Cycle:  798 | addr=  0 | data=  72 | act_out=  17 | AU_psum=  4829 | AU_bias= -2560 | seq= 2
              [BUF_WR] Cycle:  799 | addr=  1 | data=  17 | act_out=  95 | AU_psum= -4097 | AU_bias=  1792 | seq= 3
              [BUF_WR] Cycle:  800 | addr=  2 | data=  95 | act_out=  30 | AU_psum= 18004 | AU_bias=  1792 | seq= 4
              [BUF_WR] Cycle:  801 | addr=  3 | data=  30 | act_out= 127 | AU_psum=  1232 | AU_bias= -2048 | seq= 5
              [BUF_WR] Cycle:  802 | addr= 32 | data= 127 | act_out=  51 | AU_psum=-28361 | AU_bias= -2560 | seq= 6
              [BUF_WR] Cycle:  803 | addr= 33 | data=  51 | act_out=   0 | AU_psum=-32768 | AU_bias=  1792 | seq= 7
              [BUF_WR] Cycle:  804 | addr= 34 | data=   0 | act_out=   0 | AU_psum=  8248 | AU_bias=  1792 | seq= 8
              [BUF_WR] Cycle:  805 | addr= 35 | data=   0 | act_out= 127 | AU_psum=  2265 | AU_bias= -2048 | seq= 9
              [BUF_WR] Cycle:  806 | addr= 64 | data= 127 | act_out=  66 | AU_psum=     5 | AU_bias= -2560 | seq=10
              [BUF_WR] Cycle:  807 | addr= 65 | data=  66 | act_out=  28 | AU_psum=   884 | AU_bias=  1792 | seq=11
              [BUF_WR] Cycle:  808 | addr= 66 | data=  28 | act_out= 100 | AU_psum=   914 | AU_bias=  1792 | seq=12
              [BUF_WR] Cycle:  809 | addr= 67 | data= 100 | act_out= 100 | AU_psum= -5433 | AU_bias= -2048 | seq=13
              [BUF_WR] Cycle:  810 | addr= 96 | data= 100 | act_out=   3 | AU_psum=-15979 | AU_bias= -2560 | seq=14
              [BUF_WR] Cycle:  811 | addr= 97 | data=   3 | act_out=   0 | AU_psum=-13389 | AU_bias=  1792 | seq=15
              [BUF_WR] Cycle:  812 | addr= 98 | data=   0 | act_out=   0 | AU_psum= -1201 | AU_bias=  1792 | seq=16

    - 로그 분석 결과, test_data_0000 ~ 0002.txt 구간까지는 Layer 1(Neuron 0~3)의 연산 결과가 의도된 메모리 주소에 정확히 저장됨을 확인했다. 그러나 마지막 샘플인 test_data_0003.txt 처리 구간에서 데이터가 전부 저장되지 않는 현상이 발견되었다. 이는 데이터 저장 주소 카운터인 seq 값이 데이터 도달 시점보다 일찍 종료되어, 마지막 결과값을 담을 주소 공간을 할당하지 못했기 때문이다. 이는 파이프라인 지연이 누적되면서, 결과 데이터를 버퍼에 쓰는 타이밍이 데이터 유효 구간보다 짧아져서 발생한 문제이다.

    - 🚀 Unified Buffer 쓰기 타이밍 마진 개선: 마지막 샘플 저장 누락 해결
      - Buffer Write Timing을 확장함으로써 누락된 마지막 데이터를 저장할 수 있다. 즉, 카운터 로직에 +1 마진을 추가하여, seq가 유효하게 유지되도록 수정했다.

                            BUFFER_WR_L1: begin
                                //16 PE 결과를 순차적으로 처리
                                if (write_seq_cnt > 0 && write_seq_cnt <= 16+1) begin
                                    ...
                                // 16개 PE 처리 완료 시 다음 동작 결정
                                if (write_seq_cnt == 16+1) begin
                                    ...
            ############ STATE CHANGE:  --> BUFFER_WR_L1 (Cycle 796, Time 8065000) ############
              ...
              [BUF_WR] Cycle:  810 | addr= 96 | data= 100 | act_out=   3 | AU_psum=-15979 | AU_bias= -2560 | seq=14
              [BUF_WR] Cycle:  811 | addr= 97 | data=   3 | act_out=   0 | AU_psum=-13389 | AU_bias=  1792 | seq=15
              [BUF_WR] Cycle:  812 | addr= 98 | data=   0 | act_out=   0 | AU_psum= -1201 | AU_bias=  1792 | seq=16
              [BUF_WR] Cycle:  813 | addr= 99 | data=   0 | act_out=  72 | AU_psum= -1726 | AU_bias= -2048 | seq=17

      - 이제 BUF_WR이 마무리 되었다면, 다음과 같은 조건에 의해 Layer 1의 Neuron 4~7번째를 계산할 차례이다. 즉, Group1에 대한 CALC_L1 state로 넘어갈 차례이다.

                                if (write_seq_cnt == 16+1) begin
                                    // 모든 뉴런 그룹 처리 완료?
                                    if ((group_cnt + 1) * 4 >= cur_neuron_total) begin
                                       // Layer 1 완료 → Layer 2로 전환
                                        ...
            -> 아직 뉴런 4개만 처리했으므로 아래 if 조건 성립 x
                                    end else begin
                                        // 다음 뉴런 그룹 처리
                                        state <= CALC_L1;
                                        group_cnt <= group_cnt + 1;
                                        k_cnt <= 0;
                                        pe_rst <= 1;

      - 따라서 그 다음 Group1 CALC_L1 state에 대한 출력 로그를 분석해보자.

            ====== [Cycle:  814] State: CALC_L1 | k_cnt:   0 | Group: 1 | pe_en:0 | pe_rst:1 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=x  C1=x  C2=x  C3=x
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=x  C1=   0  C2=   0  C3=   0]
              [PE Accumulator Matrix] (N = Neuron 4~7)
                              N4         N5         N6         N7 
              Img0(R0) --> [  -1201] [  -1726] [   4829] [  -4097]
              Img1(R1) --> [  18004] [   1232] [ -28361] [ -32768]
              Img2(R2) --> [   8248] [   2265] [      5] [    884]
              Img3(R3) --> [    914] [  -5433] [ -15979] [ -13389]
            
            ====== [Cycle:  815] State: CALC_L1 | k_cnt:   1 | Group: 1 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=x  C1=x  C2=x  C3=x
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=x  C1=   0  C2=   0  C3=   0
              [PE Accumulator Matrix] (N = Neuron 4~7)
                              N4         N5         N6         N7 
              Img0(R0) --> [      0] [      0] [      0] [      0]
              Img1(R1) --> [      0] [      0] [      0] [      0]
              Img2(R2) --> [      0] [      0] [      0] [      0]
              Img3(R3) --> [      0] [      0] [      0] [      0]
            
            ====== [Cycle:  816] State: CALC_L1 | k_cnt:   2 | Group: 1 | pe_en:1 | pe_rst:0 ======
              [Raw Data (MUX Output)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=   0  C2=   0  C3=   0
              [Skewed Data (Array Edge)]
                Row: R0=   0  R1=   0  R2=   0  R3=   0
                Col: C0=   0  C1=x  C2=   0  C3=   0
              [PE Accumulator Matrix] (N = Neuron 4~7)
                              N4         N5         N6         N7 
              Img0(R0) --> [      0] [      0] [      0] [      0]
              Img1(R1) --> [      0] [      0] [      0] [      0]
              Img2(R2) --> [      0] [      0] [      0] [      0]
              Img3(R3) --> [      0] [      0] [      0] [      0]

      - 🔴 문제 원인 정리

        1. Cycle 813 (BUFFER_WR_L1 종료) : 
            - state <= CALC_L1, group_cnt <= 1, k_cnt <= 0, pe_rst <= 1 설정.
            - Non-blocking (<=)에 의해 Cycle 814의 Rising Edge에 적용
        2. Cycle 814 (CALC_L1 진입) : 
            - 하지만 Weight_Memory는 동기식이므로, Cycle 814에 새 주소를 받고, Cycle 815에 출력
            - 결과: Cycle 814에는 부적절한 가중치 (Group 0의 마지막 값 또는 'x')가 나옴
        3. Cycle 815 : 
            - Weight_Bank는 idx = 4, 5, 6, 7을 계산하여 w_1_4 ~ w_1_7을 선택.
            - 하지만 Weight_Memory는 여전히 부적절한 가중치 (Group 0의 마지막 값 또는 'x')가 나옴​

    - 🚀 [개선 방안] Group State Transition시 Unknown 발생 / PRE_CALC state 추가
      - 따라서 제어 흐름이 명확히하고, 각 상태의 역할이 분명하게 하기 위해 state를 추가한다. 추가된 부분은 아래와 같이 FSM state를 각 CALC마다 추가한다.
        
            localparam PRE_CALC_L1 = 9,  // Layer 1 사전 준비
                       PRE_CALC_L2 = 10, // Layer 2 사전 준비
                       PRE_CALC_L3 = 11; // Layer 3 사전 준비
                       
            reg pre_calc_cnt; //PRE_CALC Cycle Counter
        
      - 특히, Weight_Memory에서 부적절한 weight가 나오기 때문에, 해당 모듈의 관점에서 Timing을 분석 해보고, State가 어느 정도 사이클이 필요한지 분석하고 이를 기반으로 FSM을 재설계한다.

      - Weight_Memory : Synchronous ROM

            // Weight_Memory.v
            always @(posedge i_clk) begin
                if (i_ren) begin
                    o_wout <= mem[i_radd];  // ← Non-blocking! 다음 클럭에 출력
                end
            end
        
      - Cycle 813 → 814 (PRE_CALC_L1 진입)

            // Cycle 813 종료 시점
            group_cnt <= 1;  // Non-blocking
            k_cnt <= 0;      // Non-blocking
            // 레지스터 값
            group_cnt = 1    // ✅ 갱신 완료
            k_cnt = 0        // ✅ 갱신 완료
            
            // Weight_Bank (조합 로직 - 즉시 계산)
            idx = 1*4 + k = 4, 5, 6, 7  // ✅ 정확
            
            // Weight_Memory에 전달
            i_radd = k_cnt = 0  // ✅ 정확
            
            // ❌ 하지만! Weight_Memory는 동기식
            // Cycle 814 Rising Edge: 주소 0 받음
            // Cycle 814 동안: 이전 주소(793)의 데이터 출력
            // Cycle 815 Rising Edge: 주소 0의 데이터 출력
            
            // PRE_CALC_L1에서
            raw_weight_data <= w_bank_out;  
            // ← w_bank_out은 아직 주소 0의 데이터가 아님!

      - Cycle 815 (Group 1의 CALC_L1 진입) :
     
            // raw_weight_data = Cycle 814에서 할당된 값
            //                 = Cycle 814의 w_bank_out
            //                 = Weight_Memory가 Cycle 814에 출력한 값
            //                 = 이전 주소(793)의 데이터 ❌
            
            // sys_col_in (조합 로직)
            sys_col_in = raw_weight_data  // ← 부적절한 값!
            Cycle 816
            
            // CALC_L1에서 (Cycle 815 실행)
            raw_weight_data <= w_bank_out;
            // w_bank_out = Cycle 815의 Weight_Memory 출력
            //            = 주소 0의 데이터 ✅ 정상!
            
            // Cycle 816에서 비로소
            raw_weight_data = w_1_4[0] ~ w_1_7[0]  ✅
            
            최종적으로 2 Cycle의 prefetch 필요!
            주소 변경 → [1 cycle] → 메모리 출력 → [1 cycle] → 레지스터 저장

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_49.png" width="400"/>

<div align="left">

### FSM 코드 전면 수정

                        // ============================================================
                        // [BUFFER_WR_L1] Layer 1 결과 저장
                        // ============================================================
                        BUFFER_WR_L1: begin
                            if (write_seq_cnt > 0 && write_seq_cnt <= 16+1) begin
                                if (wr_global_neuron_idx < cur_neuron_total) begin
                                    ..동일
                                end
                            end         
                            if (write_seq_cnt == 16+1) begin
                                if ((group_cnt + 1) * 4 >= cur_neuron_total) begin                    
                                    ..동일
                                end else begin
                                    // 다음 뉴런 그룹 처리
                                    state <= PRE_CALC_L1; //변경사항1
                                    ..동일
                                    pre_calc_cnt <= 0;    //변경사항2
                                end
                                ..동일
        
                        // ============================================================
                        // [PRE_CALC_L1] 
                        // ============================================================
                        PRE_CALC_L1 : begin
                            pe_rst <= 1; //리셋 유지
                            pe_en  <= 0; //PE 비활성화
                        
                            //Data Pre-Fetch
                            raw_input_data <= i_input_pixels;
                            raw_weight_data <= w_bank_out;
                            
                            if(pre_calc_cnt == 0) begin
                                //1st cycle : address 안정화
                                pre_calc_cnt <= 1;
                            end else begin
                                //2nd cycle : Memory Data Setting Done
                                state <= CALC_L1;
                                pre_calc_cnt <=0;
                            end
                        end
        (나머지 CALC_L2,L3 / PRE_CALC_L2,L3도 동일한 방식)

        ############ STATE CHANGE:  --> PRE_CALC_L1 (Cycle 814, Time 8245000) ############
        ############ STATE CHANGE:  --> CALC_L1 (Cycle 816, Time 8265000) ############
        ====== [Cycle:  816] State: CALC_L1 | k_cnt:   0 | Group: 1 | pe_en:0 | pe_rst:1 ======
          [Raw Data (MUX Output)]
            Row: R0=   0  R1=   0  R2=   0  R3=   0
            Col: C0=   0  C1=   0  C2=   0  C3=   0
          [Skewed Data (Array Edge)]
            Row: R0=   0  R1=   0  R2=   0  R3=   0
            Col: C0=   0  C1=   0  C2=   0  C3=   0
          [PE Accumulator Matrix] (N = Neuron 4~7)
                          N4         N5         N6         N7 
          Img0(R0) --> [      0] [      0] [      0] [      0]
          Img1(R1) --> [      0] [      0] [      0] [      0]
          Img2(R2) --> [      0] [      0] [      0] [      0]
          Img3(R3) --> [      0] [      0] [      0] [      0]
        
        ====== [Cycle:  817] State: CALC_L1 | k_cnt:   1 | Group: 1 | pe_en:1 | pe_rst:0 ======
          [Raw Data (MUX Output)]
            Row: R0=   0  R1=   0  R2=   0  R3=   0
            Col: C0=   0  C1=   0  C2=   0  C3=   0
          [Skewed Data (Array Edge)]
            Row: R0=   0  R1=   0  R2=   0  R3=   0
            Col: C0=   0  C1=   0  C2=   0  C3=   0
          [PE Accumulator Matrix] (N = Neuron 4~7)
                          N4         N5         N6         N7 
          Img0(R0) --> [      0] [      0] [      0] [      0]
          Img1(R1) --> [      0] [      0] [      0] [      0]
          Img2(R2) --> [      0] [      0] [      0] [      0]
          Img3(R3) --> [      0] [      0] [      0] [      0]

- 최종적으로 Unknown 현상 및 Group1의 CALC_L1 수행 결과(Data Skew 및 MAC 연산 결과)가 정상적으로 동작했음을 확인했다.

- 4️⃣ 다음 Group 연산 결과에 대한 BUF_WR 오류

        ====== [Cycle: 1609] State: CALC_L1 | k_cnt: 793 | Group: 1 | pe_en:1 | pe_rst:0 ======
          [PE Accumulator Matrix] (N = Neuron 4~7)
                          N4         N5         N6         N7 
          Img0(R0) --> [ -14115] [   2838] [   7881] [   3101]
          Img1(R1) --> [  -1926] [   5469] [  -7912] [  17971]
          Img2(R2) --> [   2995] [  -1404] [   4921] [  -7454]
          Img3(R3) --> [  -7661] [   5968] [  -2668] [  16330]
        
        ############ STATE CHANGE:  --> BUFFER_WR_L1 (Cycle 1610, Time 16205000) ############
          [BUF_WR] Cycle: 1610 | addr=  0 | data=  72 | act_out=   0 | AU_psum=-14115 | AU_bias=     0 | seq= 0
          [BUF_WR] Cycle: 1611 | addr=  0 | data=  72 | act_out=   0 | AU_psum=  2838 | AU_bias= -4608 | seq= 1
          [BUF_WR] Cycle: 1612 | addr=  4 | data=   0 | act_out=  37 | AU_psum=  7881 | AU_bias= -3072 | seq= 2
          [BUF_WR] Cycle: 1613 | addr=  5 | data=  37 | act_out= 116 | AU_psum=  3101 | AU_bias= -2816 | seq= 3
          [BUF_WR] Cycle: 1614 | addr=  6 | data= 116 | act_out=  67 | AU_psum= -1926 | AU_bias=     0 | seq= 4
          [BUF_WR] Cycle: 1615 | addr=  7 | data=  67 | act_out=  35 | AU_psum=  5469 | AU_bias= -4608 | seq= 5
          [BUF_WR] Cycle: 1616 | addr= 36 | data=  35 | act_out=  76 | AU_psum= -7912 | AU_bias= -3072 | seq= 6
          [BUF_WR] Cycle: 1617 | addr= 37 | data=  76 | act_out=   0 | AU_psum= 17971 | AU_bias= -2816 | seq= 7
          [BUF_WR] Cycle: 1618 | addr= 38 | data=   0 | act_out= 127 | AU_psum=  2995 | AU_bias=     0 | seq= 8
          [BUF_WR] Cycle: 1619 | addr= 39 | data= 127 | act_out= 103 | AU_psum= -1404 | AU_bias= -4608 | seq= 9
          [BUF_WR] Cycle: 1620 | addr= 68 | data= 103 | act_out=   6 | AU_psum=  4921 | AU_bias= -3072 | seq=10
          [BUF_WR] Cycle: 1621 | addr= 69 | data=   6 | act_out=  90 | AU_psum= -7454 | AU_bias= -2816 | seq=11
          [BUF_WR] Cycle: 1622 | addr= 70 | data=  90 | act_out=   0 | AU_psum= -7661 | AU_bias=     0 | seq=12
          [BUF_WR] Cycle: 1623 | addr= 71 | data=   0 | act_out=   2 | AU_psum=  5968 | AU_bias= -4608 | seq=13
          [BUF_WR] Cycle: 1624 | addr=100 | data=   2 | act_out=  84 | AU_psum= -2668 | AU_bias= -3072 | seq=14
          [BUF_WR] Cycle: 1625 | addr=101 | data=  84 | act_out=   7 | AU_psum= 16330 | AU_bias= -2816 | seq=15
          [BUF_WR] Cycle: 1626 | addr=102 | data=   7 | act_out= 127 | AU_psum=-14115 | AU_bias=     0 | seq=16
          [BUF_WR] Cycle: 1627 | addr=103 | data= 127 | act_out=   0 | AU_psum=  2838 | AU_bias= -4608 | seq=17

    - 위 출력 로그는 Group1(Layer1, Neuron4~7번째)에 대한 최종 연산 결과 및 BUF_WR 과정이다. 현 로그에서 보이는 문제점은 Golden Model과의 addr, data 불일치성으로 확인된다.

            ====== [Image 0] LAYER 1 OUTPUT (Cycle: 807) ======
                N[ 0- 4]:   72    17    95    30     0
                N[ 5- 9]:   37   116    67   127    17
                N[10-14]:    0     0     0   115     0
                N[15-19]:  127     0    55     0    36
                N[20-24]:    0    18    17    35    84
                N[25-29]:    1    73     3    12    95
              [DEBUG_L1] Layer 1 -> Layer 2 data transfer starting...
            
            -> N[4] ~ N[7] : 0, 37, 116,67이 addr 4 5 6 7에 들어가야 하는데, 
            -> 현재 출력 로그 결과는 어딘가 Mismatch되는 것으로 보임
            ====== [Image 1] LAYER 1 OUTPUT (Cycle: 1735) ======
              [DEBUG_L1] Layer 1 computation completed
              [DEBUG_L1] Number of neurons: 30
              [DEBUG_L1] All neuron outputs (after activation):
                N[ 0- 4]:  127    51     0     0    35
                N[ 5- 9]:   76     0   127   127   127
                N[10-14]:    0    17     4     0     0
                N[15-19]:  127   125     0    16   126
                N[20-24]:  126    96    60   127     9
                N[25-29]:  125    17   126   122     0
              [DEBUG_L1] Layer 1 -> Layer 2 data transfer starting...
            
            -> N[4] ~ N[7] : 35, 76, 0, 127이 addr 36 37 38 39에 들어가야 하는데, 이 부분은 정확히 반영됨.
            -> 추가로 나머지 image2, 3에 대해서도 정확히 반영됨.

    - 따라서 근본적인 문제 원인은 test_data_0000.txt가 Unified Buffer에 정확히 반영이 되지 않고 있다.

            [BUF_WR] Cycle: 1610 | addr=  0 | data=  72 | seq= 0  ← ❌ 원치 않는 쓰기!
            [BUF_WR] Cycle: 1611 | addr=  0 | data=  72 | seq= 1  ← ❌ 또 쓰기!
            [BUF_WR] Cycle: 1612 | addr=  4 | data=   0 | seq= 2  ← ✅ 정상 (Group 1 시작)

    - 🚀 [개선 방안] Group State Transition시 Unknown 발생 / PRE_CALC state 추가
      - 이는 write_seq_cnt = 0일 때 조건 불만족되며 buf_wen 갱신 안 되는 현상으로 판단하였다. 즉, 이전 사이클의 buf_wen = 1이 유지되고, 레지스터 buf_w_addr, buf_w_data도 이전 값 유지하다보니, 의도치 않은 메모리 쓰기가 발생한 것으로 판단된다. 따라서 다음과 같이 BUFFER_WR_L1,L2,L3에 buf_wen을 리셋하는 구문을 추가한다.

                            // ============================================================
                            // [BUFFER_WR_L1] Layer 1 결과 저장
                            // ============================================================
                            BUFFER_WR_L1: begin
                                if (write_seq_cnt > 0 && write_seq_cnt <= 16+1) begin
                                    ..동일
                                end else begin //변경사항 추가1
                                    buf_wen <= 0; 
                                end
                                if (write_seq_cnt == 16+1) begin
                                    if ((group_cnt + 1) * 4 >= cur_neuron_total) begin                    
                                        ..동일
                                    end
                                    write_seq_cnt <= 0;
                                    buf_wen <= 0; //변경사항 추가1
                                end else begin
                                ..동일

      - 따라서 다음과 같이 출력 로그가 개선된 것을 확인할 수 있다.

            ############ STATE CHANGE:  --> BUFFER_WR_L1 (Cycle 1610, Time 16205000) ############
              [BUF_WR] Cycle: 1612 | addr=  4 | data=   0 | act_out=  37 | AU_psum=  7881 | AU_bias= -3072 | seq= 2
              [BUF_WR] Cycle: 1613 | addr=  5 | data=  37 | act_out= 116 | AU_psum=  3101 | AU_bias= -2816 | seq= 3
              [BUF_WR] Cycle: 1614 | addr=  6 | data= 116 | act_out=  67 | AU_psum= -1926 | AU_bias=     0 | seq= 4
              [BUF_WR] Cycle: 1615 | addr=  7 | data=  67 | act_out=  35 | AU_psum=  5469 | AU_bias= -4608 | seq= 5
              [BUF_WR] Cycle: 1616 | addr= 36 | data=  35 | act_out=  76 | AU_psum= -7912 | AU_bias= -3072 | seq= 6
              [BUF_WR] Cycle: 1617 | addr= 37 | data=  76 | act_out=   0 | AU_psum= 17971 | AU_bias= -2816 | seq= 7
              [BUF_WR] Cycle: 1618 | addr= 38 | data=   0 | act_out= 127 | AU_psum=  2995 | AU_bias=     0 | seq= 8
              [BUF_WR] Cycle: 1619 | addr= 39 | data= 127 | act_out= 103 | AU_psum= -1404 | AU_bias= -4608 | seq= 9
              [BUF_WR] Cycle: 1620 | addr= 68 | data= 103 | act_out=   6 | AU_psum=  4921 | AU_bias= -3072 | seq=10
              [BUF_WR] Cycle: 1621 | addr= 69 | data=   6 | act_out=  90 | AU_psum= -7454 | AU_bias= -2816 | seq=11
              [BUF_WR] Cycle: 1622 | addr= 70 | data=  90 | act_out=   0 | AU_psum= -7661 | AU_bias=     0 | seq=12
              [BUF_WR] Cycle: 1623 | addr= 71 | data=   0 | act_out=   2 | AU_psum=  5968 | AU_bias= -4608 | seq=13
              [BUF_WR] Cycle: 1624 | addr=100 | data=   2 | act_out=  84 | AU_psum= -2668 | AU_bias= -3072 | seq=14
              [BUF_WR] Cycle: 1625 | addr=101 | data=  84 | act_out=   7 | AU_psum= 16330 | AU_bias= -2816 | seq=15
              [BUF_WR] Cycle: 1626 | addr=102 | data=   7 | act_out= 127 | AU_psum=-14115 | AU_bias=     0 | seq=16
              [BUF_WR] Cycle: 1627 | addr=103 | data= 127 | act_out=   0 | AU_psum=  2838 | AU_bias= -4608 | seq=17

      - 아래는 마지막 Group7의 CALC_L1까지 정상적으로 BUF_WR이 동작함을 알 수 있다.

            ############ STATE CHANGE:  --> BUFFER_WR_L1 (Cycle 6494, Time 65045000) ############
              [BUF_WR] Cycle: 6496 | addr= 28 | data=  12 | act_out=  95 | AU_psum=     0 | AU_bias=     0 | seq= 2
              [BUF_WR] Cycle: 6497 | addr= 29 | data=  95 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq= 3
              [BUF_WR] Cycle: 6498 | addr= 29 | data=  95 | act_out=  64 | AU_psum=  6335 | AU_bias=   256 | seq= 4
              [BUF_WR] Cycle: 6499 | addr= 29 | data=  95 | act_out= 122 | AU_psum=-32768 | AU_bias=  1280 | seq= 5
              [BUF_WR] Cycle: 6500 | addr= 60 | data= 122 | act_out=   0 | AU_psum=     0 | AU_bias=     0 | seq= 6
              [BUF_WR] Cycle: 6501 | addr= 61 | data=   0 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq= 7
              [BUF_WR] Cycle: 6502 | addr= 61 | data=   0 | act_out=  64 | AU_psum=  4515 | AU_bias=   256 | seq= 8
              [BUF_WR] Cycle: 6503 | addr= 61 | data=   0 | act_out= 116 | AU_psum= -7846 | AU_bias=  1280 | seq= 9
              [BUF_WR] Cycle: 6504 | addr= 92 | data= 116 | act_out=   4 | AU_psum=     0 | AU_bias=     0 | seq=10
              [BUF_WR] Cycle: 6505 | addr= 93 | data=   4 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq=11
              [BUF_WR] Cycle: 6506 | addr= 93 | data=   4 | act_out=  64 | AU_psum= -5635 | AU_bias=   256 | seq=12
              [BUF_WR] Cycle: 6507 | addr= 93 | data=   4 | act_out=   8 | AU_psum=-26676 | AU_bias=  1280 | seq=13
              [BUF_WR] Cycle: 6508 | addr=124 | data=   8 | act_out=   0 | AU_psum=     0 | AU_bias=     0 | seq=14
              [BUF_WR] Cycle: 6509 | addr=125 | data=   0 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq=15
              [BUF_WR] Cycle: 6510 | addr=125 | data=   0 | act_out=  64 | AU_psum= -4783 | AU_bias=   256 | seq=16
              [BUF_WR] Cycle: 6511 | addr=125 | data=   0 | act_out=  12 | AU_psum=   899 | AU_bias=  1280 | seq=17

- 5️⃣ FSM State Transition시 추가적인 Unknown 현상 발생

        ############ STATE CHANGE:  --> BUFFER_WR_L1 (Cycle 6494, Time 65045000) ############
          BUF_WR 로그 결과는 바로 위에서 확인 가능
        
        ############ STATE CHANGE:  --> CALC_L2 (Cycle 6512, Time 65225000) ############
        
        ====== [Cycle: 6512] State: CALC_L2 | k_cnt:   0 | Group: 0 | pe_en:0 | pe_rst:1 ======
          [Raw Data (MUX Output)]
            Row: R0=   0  R1=   0  R2=   0  R3=   0
            Col: C0=x  C1=x  C2=   0  C3=   0
          [Skewed Data (Array Edge)]
            Row: R0=   0  R1=   0  R2=   0  R3=   0
            Col: C0=x  C1=   0  C2=   0  C3=   0
          [Wavefront Entry Pattern]
                                   C0=x  C1=   0  C2=   0  C3=   0  <-- Weight (Col)
                                   |       |       |       |
                                   v       v       v       v
          R0=   0  ---------->  [PE00]  [PE01]  [PE02]  [PE03]
          R1=   0  ---------->  [PE10]  [PE11]  [PE12]  [PE13]
          R2=   0  ---------->  [PE20]  [PE21]  [PE22]  [PE23]
          R3=   0  ---------->  [PE30]  [PE31]  [PE32]  [PE33]
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [  -4783] [    899] [      0] [      0]
          Img1(R1) --> [   6335] [ -32768] [      0] [      0]
          Img2(R2) --> [   4515] [  -7846] [      0] [      0]
          Img3(R3) --> [  -5635] [ -26676] [      0] [      0]
        
        ====== [Cycle: 6513] State: CALC_L2 | k_cnt:   1 | Group: 0 | pe_en:1 | pe_rst:0 ======
          [Raw Data (MUX Output)]
            Row: R0=  72  R1= 127  R2= 127  R3= 100
            Col: C0= -19  C1= -45  C2=  -2  C3=   9
          [Skewed Data (Array Edge)]
            Row: R0=  72  R1=   0  R2=   0  R3=   0
            Col: C0= -19  C1=   0  C2=   0  C3=   0
          [Wavefront Entry Pattern]
                                   C0= -19  C1=   0  C2=   0  C3=   0  <-- Weight (Col)
                                   |       |       |       |
                                   v       v       v       v
          R0=  72  ---------->  [PE00]  [PE01]  [PE02]  [PE03]
          R1=   0  ---------->  [PE10]  [PE11]  [PE12]  [PE13]
          R2=   0  ---------->  [PE20]  [PE21]  [PE22]  [PE23]
          R3=   0  ---------->  [PE30]  [PE31]  [PE32]  [PE33]
          [PE Accumulator Matrix] (N = Neuron 0~3)
                          N 0        N1         N2         N3 
          Img0(R0) --> [      0] [      0] [      0] [      0]
          Img1(R1) --> [      0] [      0] [      0] [      0]
          Img2(R2) --> [      0] [      0] [      0] [      0]
          Img3(R3) --> [      0] [      0] [      0] [      0]

      👉 앞서 살펴본 문제와 동일하게 Layer 2 연산 즉, Unified Buffer에서 Layer1 결과를 가져오고, Weight_Bank에서 Layer2에 대한 Weight를 가져오는 과정에서 Memory Latency 현상이 관찰된다.

    - 🚀 [개선 방안] FSM 일부 수정
      - 이는 기존 FSM을 다음과 같이 일부 수정함으로써 해결할 수 있다. 구체적으로는 BUFFER_WR_L1,L2 역시 바로 CALC_L2,L3로 넘어가지 않고, PRE_CALC_L2,L3로 넘어감으로써 Memory 갱신 시간을 확보한다.

                            // ============================================================
                            // [BUFFER_WR_L1] Layer 1 결과 저장
                            // ============================================================
                            BUFFER_WR_L1: begin
                                if (write_seq_cnt > 0 && write_seq_cnt <= 16+1) begin
                                    ..동일
                                end
                                if (write_seq_cnt == 16+1) begin
                                    if ((group_cnt + 1) * 4 >= cur_neuron_total) begin                    
                                        state <= PRE_CALC_L2; //변경사항 추가 추가1
                                        ..동일
                                        pre_calc_cnt <= 0; //변경사항 추가 추가2
                                    end else begin
                                        ..동일
        
                            ====== [Cycle: 6514] State: CALC_L2 | k_cnt:   0 | Group: 0 | pe_en:0 | pe_rst:1 ======
                              [Raw Data (MUX Output)]
                                Row: R0=  72  R1= 127  R2= 127  R3= 100
                                Col: C0=  -9  C1=   4  C2=   1  C3=  -7
                              [Skewed Data (Array Edge)]
                                Row: R0=  72  R1=   0  R2=   0  R3=   0
                                Col: C0=  -9  C1=   0  C2=   0  C3=   0
                            
                            ====== [Cycle: 6515] State: CALC_L2 | k_cnt:   1 | Group: 0 | pe_en:1 | pe_rst:0 ======
                              [Raw Data (MUX Output)]
                                Row: R0=  72  R1= 127  R2= 127  R3= 100
                                Col: C0=  -9  C1=   4  C2=   1  C3=  -7
                              [Skewed Data (Array Edge)]
                                Row: R0=  72  R1=   0  R2=   0  R3=   0
                                Col: C0=  -9  C1=   0  C2=   0  C3=   0
                            
                            ====== [Cycle: 6516] State: CALC_L2 | k_cnt:   2 | Group: 0 | pe_en:1 | pe_rst:0 ======
                              [Raw Data (MUX Output)]
                                Row: R0=  17  R1=  51  R2=  66  R3=   3
                                Col: C0=  -9  C1=   4  C2=   1  C3=  -7
                              [Skewed Data (Array Edge)]
                                Row: R0=  17  R1= 127  R2=   0  R3=   0
                                Col: C0=  -9  C1=   4  C2=   0  C3=   0
                            
                            ====== [Cycle: 6517] State: CALC_L2 | k_cnt:   3 | Group: 0 | pe_en:1 | pe_rst:0 ======
                              [Raw Data (MUX Output)]
                                Row: R0=  95  R1=   0  R2=  28  R3=   0
                                Col: C0= -17  C1=  -9  C2= -16  C3=  -1
                              [Skewed Data (Array Edge)]
                                Row: R0=  95  R1=  51  R2= 127  R3=   0
                                Col: C0= -17  C1=   4  C2=   1  C3=   0

      - Unknown 현상은 해결하였으나, Skew Data의 C0가 1 Cycle 더 추가되어 들어감으로써 ROW/COL 데이터의 불일치성을 추가로 관찰하였다.

    - 🚀 [개선 방안] ROW Data 1 Cycle 지연 목적 레지스터 삽입
      - 다음과 같은 지연 목적 레지스터를 추가로 삽입함으로써 ROW Data를 1 Cycle 늦춤으로써 ROW/COL 데이터의 정합성을 맞췄다.

                reg [ARRAY_ROW*dataWidth-1:0] raw_input_data_d1;
                always @(posedge i_clk) begin
                    if (!i_rst_n) raw_input_data_d1 <= 0;
                    else raw_input_data_d1 <= raw_input_data;
                end
            
                assign sys_row_in = 
                // Layer 1: 기존대로 (지연 없음)
                ((cur_layer_num == 1) && (k_cnt < cur_input_len + 2)) ? raw_input_data :
                // Layer 2, 3: 1 Cycle 지연된 데이터 사용 
                (((cur_layer_num == 2) || (cur_layer_num == 3)) && (k_cnt < cur_input_len + 2)) ? raw_input_data_d1 :
                // Default: 0 Padding
                {ARRAY_ROW*dataWidth{1'b0}};    
                            
                assign sys_col_in = ( ((cur_layer_num == 1) && (k_cnt < cur_input_len + 2)) || 
                                      (((cur_layer_num == 2) || (cur_layer_num == 3)) && (k_cnt < cur_input_len + 2)) )
                                    ? raw_weight_data : {ARRAY_COL*dataWidth{1'b0}};

            ############ STATE CHANGE:  --> BUFFER_WR_L2 (Cycle 6554, Time 65645000) ############
              [BUF_WR] Cycle: 6556 | addr=  0 | data= 118 | act_out= 122 | AU_psum=  4169 | AU_bias=  -256 | seq= 2
              [BUF_WR] Cycle: 6557 | addr=  1 | data= 122 | act_out= 111 | AU_psum= -5385 | AU_bias=  -768 | seq= 3
              [BUF_WR] Cycle: 6558 | addr=  2 | data= 111 | act_out=   5 | AU_psum= -4885 | AU_bias= -1792 | seq= 4
              [BUF_WR] Cycle: 6559 | addr=  3 | data=   5 | act_out=   4 | AU_psum= -5960 | AU_bias=  1024 | seq= 5
              [BUF_WR] Cycle: 6560 | addr= 32 | data=   4 | act_out=  10 | AU_psum= -5107 | AU_bias=  -256 | seq= 6
              [BUF_WR] Cycle: 6561 | addr= 33 | data=  10 | act_out=   8 | AU_psum=  5514 | AU_bias=  -768 | seq= 7
              [BUF_WR] Cycle: 6562 | addr= 34 | data=   8 | act_out= 116 | AU_psum= -3375 | AU_bias= -1792 | seq= 8
              [BUF_WR] Cycle: 6563 | addr= 35 | data= 116 | act_out=   9 | AU_psum= -4244 | AU_bias=  1024 | seq= 9
              [BUF_WR] Cycle: 6564 | addr= 64 | data=   9 | act_out=  21 | AU_psum= -2060 | AU_bias=  -256 | seq=10
              [BUF_WR] Cycle: 6565 | addr= 65 | data=  21 | act_out=  30 | AU_psum= -2071 | AU_bias=  -768 | seq=11
              [BUF_WR] Cycle: 6566 | addr= 66 | data=  30 | act_out=  25 | AU_psum=  9021 | AU_bias= -1792 | seq=12
              [BUF_WR] Cycle: 6567 | addr= 67 | data=  25 | act_out= 124 | AU_psum=   762 | AU_bias=  1024 | seq=13
              [BUF_WR] Cycle: 6568 | addr= 96 | data= 124 | act_out=  89 | AU_psum=  2992 | AU_bias=  -256 | seq=14
              [BUF_WR] Cycle: 6569 | addr= 97 | data=  89 | act_out= 100 | AU_psum=  4668 | AU_bias=  -768 | seq=15
              [BUF_WR] Cycle: 6570 | addr= 98 | data= 100 | act_out= 110 | AU_psum=  7006 | AU_bias= -1792 | seq=16
              [BUF_WR] Cycle: 6571 | addr= 99 | data= 110 | act_out= 118 | AU_psum=  5455 | AU_bias=  1024 | seq=17

            //전 Layer1 결과가 Golden Model 결과와 일치하는 것으로 확인됨
            ====== [Image 0] LAYER 1 OUTPUT (Cycle: 807) ======
                N[ 0- 4]:  118   122   111     5     7
                N[ 5- 9]:  125     0    64   113    86
                N[10-14]:   91   125    73     1    78
                N[15-19]:   40   118   105     0   125
            
            ====== [Image 1] LAYER 1 OUTPUT (Cycle: 1735) ======
              [DEBUG_L1] Layer 1 computation completed
              [DEBUG_L1] Number of neurons: 30
              [DEBUG_L1] All neuron outputs (after activation):
                N[ 0- 4]:    4    10     8   116    91
                N[ 5- 9]:  102    43    21    12    85
                N[10-14]:    1   115     2     4    59
                N[15-19]:  100    29    22    34    63

      - 데이터 정합성(Alignment) 및 연산 로직은 정상이나, Write Address 스케줄링에 문제가 있는 것을 추가적으로 확인하였다. Layer 2 결과가 아직 필요한 Layer 1 연산 결과 영역을 Overwrite하고 있어, 데이터 오염으로 인한 연산 불가 현상이 발생 중인 것으로 파악된다.

    - 🚀 [개선 방안] Ping-Pong Buffering
      - Memory Overwrite 현상을 해결하기 위해 Buffer를 Bank 0 (0~127), Bank(128 ~255)로 분할하여 read/write 영역을 분리하도록 한다. 블록 다이어그램으로 설명한 개선 전략은 아래와 같다.

            ┌─────────────────────────────────────────────────────┐
            │ Bank 0 (addr 0~127)                                 │
            │  - Layer 1 출력 저장                                │
            │  - Layer 2 입력으로 사용                            │
            │  - Layer 3 출력 저장 (Layer 1 재사용)               │
            ├─────────────────────────────────────────────────────┤
            │ Bank 1 (addr 128~255)                               │
            │  - Layer 2 출력 저장                                │
            │  - Layer 3 입력으로 사용                            │
            └─────────────────────────────────────────────────────┘

      - 구체적인 코드 수정 포인트는 다음과 같다.

            // 1. 새로운 신호 추가 (Ping-Pong Buffering Offset)
            reg [7:0] buf_base_offset_rd;  // 읽기용 Base Address Offset
            reg [7:0] buf_base_offset_wr;  // 쓰기용 Base Address Offset
            
            // 2. Read Address 수정
            assign buf_r_addr[0] = buf_base_offset_rd + (0 * 32) + k_cnt;
            assign buf_r_addr[1] = buf_base_offset_rd + (1 * 32) + k_cnt;
            assign buf_r_addr[2] = buf_base_offset_rd + (2 * 32) + k_cnt;
            assign buf_r_addr[3] = buf_base_offset_rd + (3 * 32) + k_cnt;
            
            // 3. Write Address 수정
            buf_w_addr <= buf_base_offset_wr + (wr_img_idx * 32) + wr_global_neuron_idx;

      - 즉, offset register를 추가로 삽입하여 address 계산을 전반적으로 수정하였다. 이렇게 수정된 address를 통해 FSM 내부를 일부 수정한다.
     
            // Layer 1 → Layer 2 전환 부분 수정
            if ((group_cnt + 1) * 4 >= cur_neuron_total) begin
                state <= PRE_CALC_L2;
                ...
                // ===== Layer 2 Buffer Offset 설정 =====
                buf_base_offset_rd <= 0;    // Bank 0에서 읽기 (Layer 1 출력)
                buf_base_offset_wr <= 128;  // Bank 1에 쓰기 (덮어쓰기 방지)
            end
            
            // Layer 2 → Layer 3 전환
            if ((group_cnt + 1) * 4 >= cur_neuron_total) begin
                state <= PRE_CALC_L3;
                ...
                // ===== Layer 3 Buffer Offset 설정 =====
                buf_base_offset_rd <= 128;  // Bank 1에서 읽기 (Layer 2 출력)
                buf_base_offset_wr <= 0;    // Bank 0에 쓰기 (Layer 1 재사용 가능)
            end

      - 이렇게 수정 후 블록 다이어그램으로 본 메모리 맵은 다음과 같다.   

            [Layer 1 완료]
            Bank 0: ┌───────────────────────┐
                    │ L1 출력 (addr 0~127)  │ ← Layer 2가 읽을 데이터
                    └───────────────────────┘
            Bank 1: ┌───────────────────────┐
                    │ (Empty)               │
                    └───────────────────────┘
            
            [Layer 2 완료]
            Bank 0: ┌───────────────────────┐
                    │ L1 출력 (보존됨!)     │
                    └───────────────────────┘
            Bank 1: ┌───────────────────────┐
                    │ L2 출력 (128~255)     │ ← Layer 3가 읽을 데이터 ✅
                    └───────────────────────┘
            
            [Layer 3 완료]
            Bank 0: ┌───────────────────────┐
                    │ L3 출력 (0~127)       │ ← MaxFinder가 읽을 최종 결과 ✅
                    └───────────────────────┘
            Bank 1: ┌───────────────────────┐
                    │ L2 출력 (보존됨)      │
                    └───────────────────────┘

      - 반영된 최종 CALC_L2의 BUF_WR에 대한 로그는 다음과 같이 Golden Model과 일치함을 알 수 있다.
      - Layer 2 : N[0]..N[3]

            ############ STATE CHANGE:  --> BUFFER_WR_L2 (Cycle 6554, Time 65645000) ############
              [BUF_WR] Cycle: 6556 | addr=128 | data= 118 | act_out= 122 | AU_psum=  4169 | AU_bias=  -256 | seq= 2
              [BUF_WR] Cycle: 6557 | addr=129 | data= 122 | act_out= 111 | AU_psum= -5385 | AU_bias=  -768 | seq= 3
              [BUF_WR] Cycle: 6558 | addr=130 | data= 111 | act_out=   5 | AU_psum= -4885 | AU_bias= -1792 | seq= 4
              [BUF_WR] Cycle: 6559 | addr=131 | data=   5 | act_out=   4 | AU_psum= -5960 | AU_bias=  1024 | seq= 5
              [BUF_WR] Cycle: 6560 | addr=160 | data=   4 | act_out=  10 | AU_psum= -5107 | AU_bias=  -256 | seq= 6
              [BUF_WR] Cycle: 6561 | addr=161 | data=  10 | act_out=   8 | AU_psum=  5514 | AU_bias=  -768 | seq= 7
              [BUF_WR] Cycle: 6562 | addr=162 | data=   8 | act_out= 116 | AU_psum= -3375 | AU_bias= -1792 | seq= 8
              [BUF_WR] Cycle: 6563 | addr=163 | data= 116 | act_out=   9 | AU_psum= -4244 | AU_bias=  1024 | seq= 9
              [BUF_WR] Cycle: 6564 | addr=192 | data=   9 | act_out=  21 | AU_psum= -2060 | AU_bias=  -256 | seq=10
              [BUF_WR] Cycle: 6565 | addr=193 | data=  21 | act_out=  30 | AU_psum= -2071 | AU_bias=  -768 | seq=11
              [BUF_WR] Cycle: 6566 | addr=194 | data=  30 | act_out=  25 | AU_psum=  9021 | AU_bias= -1792 | seq=12
              [BUF_WR] Cycle: 6567 | addr=195 | data=  25 | act_out= 124 | AU_psum=   762 | AU_bias=  1024 | seq=13
              [BUF_WR] Cycle: 6568 | addr=224 | data= 124 | act_out=  89 | AU_psum=  2992 | AU_bias=  -256 | seq=14
              [BUF_WR] Cycle: 6569 | addr=225 | data=  89 | act_out= 100 | AU_psum=  4668 | AU_bias=  -768 | seq=15
              [BUF_WR] Cycle: 6570 | addr=226 | data= 100 | act_out= 110 | AU_psum=  7006 | AU_bias= -1792 | seq=16
              [BUF_WR] Cycle: 6571 | addr=227 | data= 110 | act_out= 118 | AU_psum=  5455 | AU_bias=  1024 | seq=17

      - Layer 2 : N[16]..N[19]
     
            ############ STATE CHANGE:  --> BUFFER_WR_L2 (Cycle 6794, Time 68045000) ############
              [BUF_WR] Cycle: 6796 | addr=144 | data= 118 | act_out= 105 | AU_psum= -7965 | AU_bias= -2560 | seq= 2
              [BUF_WR] Cycle: 6797 | addr=145 | data= 105 | act_out=   0 | AU_psum= 10843 | AU_bias= -2560 | seq= 3
              [BUF_WR] Cycle: 6798 | addr=146 | data=   0 | act_out= 125 | AU_psum= -3003 | AU_bias=   512 | seq= 4
              [BUF_WR] Cycle: 6799 | addr=147 | data= 125 | act_out=  29 | AU_psum= -3594 | AU_bias=   512 | seq= 5
              [BUF_WR] Cycle: 6800 | addr=176 | data=  29 | act_out=  22 | AU_psum=   536 | AU_bias= -2560 | seq= 6
              [BUF_WR] Cycle: 6801 | addr=177 | data=  22 | act_out=  34 | AU_psum=  2517 | AU_bias= -2560 | seq= 7
              [BUF_WR] Cycle: 6802 | addr=178 | data=  34 | act_out=  63 | AU_psum= -1948 | AU_bias=   512 | seq= 8
              [BUF_WR] Cycle: 6803 | addr=179 | data=  63 | act_out=  41 | AU_psum= -7761 | AU_bias=   512 | seq= 9
              [BUF_WR] Cycle: 6804 | addr=208 | data=  41 | act_out=   3 | AU_psum=  8627 | AU_bias= -2560 | seq=10
              [BUF_WR] Cycle: 6805 | addr=209 | data=   3 | act_out= 121 | AU_psum=  5452 | AU_bias= -2560 | seq=11
              [BUF_WR] Cycle: 6806 | addr=210 | data= 121 | act_out= 102 | AU_psum= -4699 | AU_bias=   512 | seq=12
              [BUF_WR] Cycle: 6807 | addr=211 | data= 102 | act_out=  14 | AU_psum=  4578 | AU_bias=   512 | seq=13
              [BUF_WR] Cycle: 6808 | addr=240 | data=  14 | act_out= 118 | AU_psum=   581 | AU_bias= -2560 | seq=14
              [BUF_WR] Cycle: 6809 | addr=241 | data= 118 | act_out=  35 | AU_psum=  3259 | AU_bias= -2560 | seq=15
              [BUF_WR] Cycle: 6810 | addr=242 | data=  35 | act_out=  73 | AU_psum=  4641 | AU_bias=   512 | seq=16
              [BUF_WR] Cycle: 6811 | addr=243 | data=  73 | act_out= 118 | AU_psum=  2679 | AU_bias=   512 | seq=17

            //Layer 2 Golden Model Result Summary
            ====== [Image 0] LAYER 2 OUTPUT (Cycle: 844) ======
                N[ 0- 4]:  118   122   111     5     7
                N[ 5- 9]:  125     0    64   113    86
                N[10-14]:   91   125    73     1    78
                N[15-19]:   40   118   105     0   125
            
            ====== [Image 1] LAYER 2 OUTPUT (Cycle: 1772) ======
                N[ 0- 4]:    4    10     8   116    91
                N[ 5- 9]:  102    43    21    12    85
                N[10-14]:    1   115     2     4    59
                N[15-19]:  100    29    22    34    63
            
            ====== [Image 2] LAYER 2 OUTPUT (Cycle: 2700) ======
                N[ 0- 4]:    9    21    30    25     4
                N[ 5- 9]:   87   114    20   102    92
                N[10-14]:   13   104    18    67    23
                N[15-19]:    7    41     3   121   102
            
            ====== [Image 3] LAYER 2 OUTPUT (Cycle: 3628) ======
                N[ 0- 4]:  124    89   100   110   113
                N[ 5- 9]:    8     1   120     3    13
                N[10-14]:   16    17    99     4    74
                N[15-19]:  102    14   118    35    73

      - Layer 3 : N[0]..N[3]

            ############ STATE CHANGE:  --> BUFFER_WR_L3 (Cycle 6844, Time 68545000) ############
              [BUF_WR] Cycle: 6846 | addr=  0 | data=   0 | act_out=   0 | AU_psum= -8217 | AU_bias= -5632 | seq= 2
              [BUF_WR] Cycle: 6847 | addr=  1 | data=   0 | act_out=   0 | AU_psum= -8703 | AU_bias= -1024 | seq= 3
              [BUF_WR] Cycle: 6848 | addr=  2 | data=   0 | act_out=   1 | AU_psum= -8549 | AU_bias= -6144 | seq= 4
              [BUF_WR] Cycle: 6849 | addr=  3 | data=   1 | act_out=   0 | AU_psum= -1764 | AU_bias= -6656 | seq= 5
              [BUF_WR] Cycle: 6850 | addr= 32 | data=   0 | act_out=   2 | AU_psum= 10960 | AU_bias= -5632 | seq= 6
              [BUF_WR] Cycle: 6851 | addr= 33 | data=   2 | act_out= 119 | AU_psum=-10008 | AU_bias= -1024 | seq= 7
              [BUF_WR] Cycle: 6852 | addr= 34 | data= 119 | act_out=   0 | AU_psum=-25806 | AU_bias= -6144 | seq= 8
              [BUF_WR] Cycle: 6853 | addr= 35 | data=   0 | act_out=   0 | AU_psum= 15282 | AU_bias= -6656 | seq= 9
              [BUF_WR] Cycle: 6854 | addr= 64 | data=   0 | act_out= 126 | AU_psum=-12795 | AU_bias= -5632 | seq=10
              [BUF_WR] Cycle: 6855 | addr= 65 | data= 126 | act_out=   0 | AU_psum=-15873 | AU_bias= -1024 | seq=11
              [BUF_WR] Cycle: 6856 | addr= 66 | data=   0 | act_out=   0 | AU_psum= 18123 | AU_bias= -6144 | seq=12
              [BUF_WR] Cycle: 6857 | addr= 67 | data=   0 | act_out= 127 | AU_psum=-29797 | AU_bias= -6656 | seq=13
              [BUF_WR] Cycle: 6858 | addr= 96 | data= 127 | act_out=   0 | AU_psum= -7282 | AU_bias= -5632 | seq=14
              [BUF_WR] Cycle: 6859 | addr= 97 | data=   0 | act_out=   0 | AU_psum=-14091 | AU_bias= -1024 | seq=15
              [BUF_WR] Cycle: 6860 | addr= 98 | data=   0 | act_out=   0 | AU_psum=-12245 | AU_bias= -6144 | seq=16
              [BUF_WR] Cycle: 6861 | addr= 99 | data=   0 | act_out=   0 | AU_psum=-11900 | AU_bias= -6656 | seq=17

      - Layer 3 : N[4]..N[7]

            ############ STATE CHANGE:  --> BUFFER_WR_L3 (Cycle 6894, Time 69045000) ############
              [BUF_WR] Cycle: 6896 | addr=  4 | data=   0 | act_out=   0 | AU_psum=-32768 | AU_bias= -2560 | seq= 2
              [BUF_WR] Cycle: 6897 | addr=  5 | data=   0 | act_out=   0 | AU_psum= 18238 | AU_bias= -8704 | seq= 3
              [BUF_WR] Cycle: 6898 | addr=  6 | data=   0 | act_out= 126 | AU_psum= -4732 | AU_bias= -7936 | seq= 4
              [BUF_WR] Cycle: 6899 | addr=  7 | data= 126 | act_out=   0 | AU_psum=-25363 | AU_bias= -4864 | seq= 5
              [BUF_WR] Cycle: 6900 | addr= 36 | data=   0 | act_out=   0 | AU_psum= -8842 | AU_bias= -2560 | seq= 6
              [BUF_WR] Cycle: 6901 | addr= 37 | data=   0 | act_out=   0 | AU_psum=-13262 | AU_bias= -8704 | seq= 7
              [BUF_WR] Cycle: 6902 | addr= 38 | data=   0 | act_out=   0 | AU_psum= -8956 | AU_bias= -7936 | seq= 8
              [BUF_WR] Cycle: 6903 | addr= 39 | data=   0 | act_out=   0 | AU_psum= -4706 | AU_bias= -4864 | seq= 9
              [BUF_WR] Cycle: 6904 | addr= 68 | data=   0 | act_out=   1 | AU_psum=-10207 | AU_bias= -2560 | seq=10
              [BUF_WR] Cycle: 6905 | addr= 69 | data=   1 | act_out=   0 | AU_psum= -8088 | AU_bias= -8704 | seq=11
              [BUF_WR] Cycle: 6906 | addr= 70 | data=   0 | act_out=   0 | AU_psum=-12320 | AU_bias= -7936 | seq=12
              [BUF_WR] Cycle: 6907 | addr= 71 | data=   0 | act_out=   0 | AU_psum=-10684 | AU_bias= -4864 | seq=13
              [BUF_WR] Cycle: 6908 | addr=100 | data=   0 | act_out=   0 | AU_psum=-14481 | AU_bias= -2560 | seq=14
              [BUF_WR] Cycle: 6909 | addr=101 | data=   0 | act_out=   0 | AU_psum= -9051 | AU_bias= -8704 | seq=15
              [BUF_WR] Cycle: 6910 | addr=102 | data=   0 | act_out=   0 | AU_psum=-13049 | AU_bias= -7936 | seq=16
              [BUF_WR] Cycle: 6911 | addr=103 | data=   0 | act_out=   0 | AU_psum= -9578 | AU_bias= -4864 | seq=17

      - Layer 3 : N[8]..N[9]

            ############ STATE CHANGE:  --> BUFFER_WR_L3 (Cycle 6944, Time 69545000) ############
              [BUF_WR] Cycle: 6946 | addr=  8 | data=   0 | act_out=   0 | AU_psum=     0 | AU_bias=     0 | seq= 2
              [BUF_WR] Cycle: 6947 | addr=  9 | data=   0 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq= 3
              [BUF_WR] Cycle: 6948 | addr=  9 | data=   0 | act_out=  64 | AU_psum= -4196 | AU_bias= -5888 | seq= 4
              [BUF_WR] Cycle: 6949 | addr=  9 | data=   0 | act_out=   0 | AU_psum=-16402 | AU_bias= -5888 | seq= 5
              [BUF_WR] Cycle: 6950 | addr= 40 | data=   0 | act_out=   0 | AU_psum=     0 | AU_bias=     0 | seq= 6
              [BUF_WR] Cycle: 6951 | addr= 41 | data=   0 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq= 7
              [BUF_WR] Cycle: 6952 | addr= 41 | data=   0 | act_out=  64 | AU_psum= -4807 | AU_bias= -5888 | seq= 8
              [BUF_WR] Cycle: 6953 | addr= 41 | data=   0 | act_out=   0 | AU_psum=-12134 | AU_bias= -5888 | seq= 9
              [BUF_WR] Cycle: 6954 | addr= 72 | data=   0 | act_out=   0 | AU_psum=     0 | AU_bias=     0 | seq=10
              [BUF_WR] Cycle: 6955 | addr= 73 | data=   0 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq=11
              [BUF_WR] Cycle: 6956 | addr= 73 | data=   0 | act_out=  64 | AU_psum= -8600 | AU_bias= -5888 | seq=12
              [BUF_WR] Cycle: 6957 | addr= 73 | data=   0 | act_out=   0 | AU_psum= -7576 | AU_bias= -5888 | seq=13
              [BUF_WR] Cycle: 6958 | addr=104 | data=   0 | act_out=   0 | AU_psum=     0 | AU_bias=     0 | seq=14
              [BUF_WR] Cycle: 6959 | addr=105 | data=   0 | act_out=  64 | AU_psum=     0 | AU_bias=     0 | seq=15
              [BUF_WR] Cycle: 6960 | addr=105 | data=   0 | act_out=  64 | AU_psum=-22617 | AU_bias= -5888 | seq=16
              [BUF_WR] Cycle: 6961 | addr=105 | data=   0 | act_out=   0 | AU_psum=-12119 | AU_bias= -5888 | seq=17

            ====== [Image 0] LAYER 3 OUTPUT (Cycle: 871) ======
                N[ 0- 4]:    0     0     0     1     0
                N[ 5- 9]:    0     0   126     0     0
            
            ====== [Image 1] LAYER 3 OUTPUT (Cycle: 1799) ======
                N[ 0- 4]:    0     2   119     0     0
                N[ 5- 9]:    0     0     0     0     0
            
            ====== [Image 2] LAYER 3 OUTPUT (Cycle: 2727) ======
                N[ 0- 4]:    0   126     0     0     0
                N[ 5- 9]:    1     0     0     0     0
            
            ====== [Image 3] LAYER 3 OUTPUT (Cycle: 3655) ======
                N[ 0- 4]:  127     0     0     0     0
                N[ 5- 9]:    0     0     0     0     0

- 6️⃣ 연산 완료 시점과 인터럽트 타이밍 경합 : Race Condition
    - Layer 1~3의 연산 결과는 Golden과 일치함을 확인하였고, 이제는 MaxFinder 결과를 확인할 차례이다.

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_50.png" width="400"/>

<div align="left">


    - MaxFinder 결과 Result가 모두 0으로 출력되는 문제가 발생하였다. 모든 레이어에 대한 연산 결과가 일치하였지만, 결과가 다르게 나왔다는 의미는 어떠한 제어 신호가 문제가 있는 것으로 초기에 파악하였다.
    
                    if (k_cnt == 10) begin
                        mf_valid_pulse <= 1; // Start MaxFinder
                        state <= DONE;
                        o_done_interrupt <= 1;
                    end else begin
                        k_cnt <= k_cnt + 1;
                    end

    - 구체적으로는 위 기존 코드에서는 OUTPUT_SCAN에서 MaxFinder 연산을 시작하라는 mf_valid_pulse를 보냄과 동시에, 충분한 타이밍 없이 외부(Testbench)에게 종료를 알리는 o_done_interrupt 신호를 발생시켰다. 이는 하드웨어적으로 MaxFinder가 10개의 입력값들을 비교하고 최종 결과값인 mf_out을 출력 포트에 안정적으로 띄우기 위해서는 comb logic을 통과할 물리적인 시간이 필요할 것이다. 그러나 이 연산 시간이 확보되기도 전에 Testbench가 o_done_interrupt 신호를 감지하고 곧바로 데이터를 읽으려 했기 때문에 문제가 발생하였다.
        
    - 🚀 [개선 방안] Timing 확보를 위한 안정적인 Handshaking 구현
      - OUTPUT_SCAN 상태의 마지막 사이클에서 mf_valid_pulse <= 1;로 설정하여 MaxFinder를 동작시키고, 곧바로 o_done_interrupt를 띄우는 대신, state <= DONE;을 통해 먼저 next state로 transition했다. 이를 통해 1 Cycle 마진을 확보함으로써 MaxFinder의 연산을 마치고 출력 레지스터에 래치할 수 있게 된다. 이후 DONE state에서 o_done_interrupt <= 1;로 설정함으로써 Testbench가 신호를 받고 데이터를 읽는 시점에는 이미 Valid한 결과를 출력하도록 타이밍을 동기화 시켰다.

                                OUTPUT_SCAN: begin
                                    ...
                
                                    if (k_cnt == 10) begin
                                        mf_valid_pulse <= 1;
                                        state <= DONE;
                
                                    end else begin
                                        k_cnt <= k_cnt + 1;
                                        mf_valid_pulse <= 0;
                                    end
                                end
                
                                DONE: begin
                                    mf_valid_pulse <= 0;
                                    o_done_interrupt <= 1;
                                end






    
