## [DNN with Systolic Array] NPU 최상위 module/Testbench 설계 및 분석

### NPU_TOP.v

- DUT

    - 0️⃣ Parameter & Port 정의
 
        module NPU_Top #(
            parameter dataWidth = 8,                   
            parameter integer ARRAY_ROW = 4,           
            parameter integer ARRAY_COL = 4  
            )(
            input wire                             i_clk,               
            input wire                             i_rst_n,             
            input wire                             i_start_inference,   // 추론 시작 트리거
            input wire [ARRAY_ROW*dataWidth-1:0]   i_input_pixels,      // 외부 입력 (4 × 8bit = 32bit)
            //32bit Bus = 4 samples Image x 8bit
            input wire                             i_input_valid,       // 입력 유효 신호
            // DONW State에서 o_done_interrupt <= 1 
            output reg          o_done_interrupt,                        // 추론 완료 인터럽트
            output wire [31:0]  o_result_class_0,                        // 이미지 0 분류 결과
            output wire [31:0]  o_result_class_1,                        // 이미지 1 분류 결과
            output wire [31:0]  o_result_class_2,                        // 이미지 2 분류 결과
            output wire [31:0]  o_result_class_3                         // 이미지 3 분류 결과
            );
        
            // ======================================================================================
            // 1. FSM State Definition
            // ======================================================================================
            localparam IDLE           = 0,   // 대기 상태
                       CALC_L1        = 1,   // Layer 1 연산 (784 inputs × 4 neurons/group)
                       BUFFER_WR_L1   = 2,   // Layer 1 결과 → Unified Buffer 저장
                       CALC_L2        = 3,   // Layer 2 연산 (30 inputs × 4 neurons/group)
                       BUFFER_WR_L2   = 4,   // Layer 2 결과 → Unified Buffer 저장
                       CALC_L3        = 5,   // Layer 3 연산 (20 inputs × 4 neurons/group)
                       BUFFER_WR_L3   = 6,   // Layer 3 결과 → Unified Buffer 저장
                       OUTPUT_SCAN    = 7,   // 최종 10개 출력 수집 → MaxFinder
                       DONE           = 8;   // 추론 완료
        
            localparam SIG            = 0,   // Sigmoid 활성화
                       REL            = 1;   // ReLU 활성화
        
            reg [3:0] state;                  // 현재 FSM 상태
            //4bit Register (2^4 = 16 state expression)
            //실제 사용 : 0 ~ 8 (9개)
        
            // ======================================================================================
            // 2. Internal Control Signals & Counters
            // ======================================================================================
            reg [9:0]  k_cnt;                 // 입력 시퀀스 카운터 (0 ~ input_len + ARRAY_ROW)
            //최대 784(Layer1 입력 길이) 이므로 실제로 10bit면 충분하기 때문에
            // 10개 FF로 개선
            reg [31:0] group_cnt;             // 뉴런 그룹 카운터 (0, 1, 2, ...)
            reg [4:0]  write_seq_cnt;         // 결과 기록 시퀀스 카운터 (0~16)
            reg [31:0] cur_layer_num;         // 현재 레이어 번호 (1, 2, 3)
            reg [31:0] cur_input_len;         // 현재 레이어 입력 데이터 길이
            reg [31:0] cur_neuron_total;      // 현재 레이어 전체 뉴런 수
            reg        cur_act_sel;           // 활성화 함수 선택 (0:Sigmoid, 1:ReLU)
        
            reg pe_rst;                       // PE 리셋 신호 (active-high)
            reg pe_en;                        // PE 연산 활성화 신호
            //1bit Control Signal
            //Systolic Array와 Data Skewing에 Broadcasting
            //HW : Single FF -> Fan-Out Tree (16개 PE에 분배)

