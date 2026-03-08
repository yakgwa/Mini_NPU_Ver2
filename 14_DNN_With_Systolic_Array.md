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














