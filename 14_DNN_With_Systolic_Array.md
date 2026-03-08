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





























































































