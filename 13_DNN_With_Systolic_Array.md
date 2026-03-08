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

                // ======================================================================================
                // 3. Data Path Wires
                // ======================================================================================
                // [Systolic Array I/O]
                wire signed [ARRAY_ROW*ARRAY_COL*2*dataWidth-1:0] array_out_packed; // SA 전체 출력 (256-bit)
                //16 PE x 16bit Accumulator
            
                wire [ARRAY_ROW*dataWidth-1:0] sys_row_in;  // MUX → Skewing 입력 (Row)  [CHANGED: reg → wire]
                wire [ARRAY_COL*dataWidth-1:0] sys_col_in;  // MUX → Skewing 입력 (Col)  [CHANGED: reg → wire]
                //MUX 출력 와이어
            
                // [Data_Skewing 출력 와이어]
                wire [ARRAY_ROW*dataWidth-1:0] skewed_row_data;  // Skewing 출력 → SA Row 에지
                wire [ARRAY_COL*dataWidth-1:0] skewed_col_data;  // Skewing 출력 → SA Col 에지
                //디버깅 용도로 중간 신호를 와이어로 노출, 실제 입력 엣지
            
                // [FSM → MUX 데이터]
                reg  [ARRAY_ROW*dataWidth-1:0] raw_input_data;   // FSM에서 결정된 Row 원본 데이터
                reg  [ARRAY_COL*dataWidth-1:0] raw_weight_data;   // FSM에서 결정된 Col 원본 데이터
                //register로서, FSM always 블록에 할당
                // Layer 1: `raw_input_data <= i_input_pixels`
                // Layer 2/3: `raw_input_data <= {buf_r_data[3:0]}`
            
                // [Weight/Bias Bank 출력]
                wire [ARRAY_ROW*dataWidth-1:0]   w_bank_out;      // Weight Bank → 4개 가중치 (32-bit)
                wire [ARRAY_ROW*2*dataWidth-1:0] b_bank_out;      // Bias Bank → 4개 바이어스 (64-bit)
            
                // [Unified Buffer I/O]
                reg buf_wen;                                       // Buffer 쓰기 활성화
                reg  [7:0] buf_w_addr;                             // Buffer 쓰기 주소
                reg  [dataWidth-1:0] buf_w_data;                   // Buffer 쓰기 데이터
                wire [dataWidth-1:0] buf_r_data [3:0];             // Buffer 읽기 데이터 (4 포트)
                wire [31:0] buf_r_addr [3:0];                      // Buffer 읽기 주소 (4 포트)
                // 1개 Write Port: `buf_wen`, `buf_w_addr`, `buf_w_data`
                // 4개 Read Ports: `buf_r_addr[0~3]`, `buf_r_data[0~3]`
                // buf_w_addr[31:0]: 32bit 주소
                // 실제 SRAM(Unified Buffer) 크기: 256 words → 8bit 주소면 충분 (2^8=256)
            
                // [Activation Unit I/O]
                wire [2*dataWidth-1:0] au_in_psum;                 // AU 입력: PE 누적합
                wire [2*dataWidth-1:0] au_in_bias;                 // AU 입력: Bias
                wire [dataWidth-1:0] act_out_val;                  // AU 출력: 활성화 결과
                // [Unpacked PE Results & Bias]
                wire [2*dataWidth-1:0] pe_res_unpacked [ARRAY_ROW-1:0][ARRAY_COL-1:0]; // 4×4 PE 결과
                wire [2*dataWidth-1:0] bias_unpacked [3:0];        // 4개 Bias 값
                wire [2*dataWidth-1:0] debug_mul_00;               // PE[0][0] 곱셈 디버그
                // pe_res_unpacked[4][4] : 2D 배열 (16개 × 16-bit)
                // 합성: `array_out_packed`의 비트 슬라이싱 결과
                // Activation Unit MUX :
                // `au_in_psum` = `pe_res_unpacked[in_img_idx][in_neu_idx]`
                // `au_in_bias` = `bias_unpacked[in_neu_idx]`
                // HW : 16:1 MUX (write_seq_cnt로 선택)
                
                // [BUFFER_WR 인덱스 계산]
                wire [1:0]  in_img_idx;           // write_seq_cnt에서 이미지 인덱스 추출
                wire [1:0]  in_neu_idx;           // write_seq_cnt에서 뉴런 인덱스 추출
                wire [4:0]  delayed_seq_cnt;      // 1사이클 지연된 시퀀스 카운터
                wire [1:0]  wr_img_idx;           // 실제 기록용 이미지 인덱스
                wire [1:0]  wr_neu_idx;           // 실제 기록용 뉴런 인덱스
                wire [31:0] wr_global_neuron_idx; // 전역 뉴런 인덱스 (group*4 + neu)
            
                // [MaxFinder]
                reg [10*dataWidth-1:0] mf_in_data [3:0];   // MaxFinder 입력 (10개 × 8bit × 4이미지)
                reg mf_valid_pulse;                          // MaxFinder 유효 펄스
                wire [31:0] mf_out [3:0];                    // MaxFinder 출력 (분류 결과)
                wire [3:0] mf_out_valid;                     // MaxFinder 출력 유효

    - 1️⃣ assign : Combination Logic

             // ======================================================================================
            // 4. Combinational Logic
            // ======================================================================================
            // BUFFER_WR 상태에서 16개 PE 결과를 순차적으로 읽어오기 위한 인덱스 계산
            // write_seq_cnt = {img_idx[1:0], neu_idx[1:0]} 형태로 16 = 4이미지 × 4뉴런
            assign in_img_idx = write_seq_cnt[3:2];   // 상위 2비트 = 이미지 번호
            assign in_neu_idx = write_seq_cnt[1:0];   // 하위 2비트 = 뉴런 번호
        
            // Activation Unit 출력이 1사이클 지연되므로 실제 기록은 delayed_seq_cnt 기반
            assign delayed_seq_cnt = write_seq_cnt - 1;
            assign wr_img_idx = delayed_seq_cnt[3:2];
            assign wr_neu_idx = delayed_seq_cnt[1:0];
        
            // 전역 뉴런 인덱스: 현재 그룹의 4개 뉴런 중 몇 번째인지
            assign wr_global_neuron_idx = group_cnt * 4 + wr_neu_idx;
        
            // Activation Unit 입력 MUX: write_seq_cnt가 가리키는 PE와 Bias를 선택
            assign au_in_psum = pe_res_unpacked[in_img_idx][in_neu_idx];
            assign au_in_bias = bias_unpacked[in_neu_idx];
        
            // Unified Buffer 읽기 주소: 이미지별로 32-word 오프셋
            // Layer 2/3에서 이전 레이어 출력을 읽어올 때 사용
            assign buf_r_addr[0] = (0 * 32) + k_cnt;   // Image 0 영역
            assign buf_r_addr[1] = (1 * 32) + k_cnt;   // Image 1 영역
            assign buf_r_addr[2] = (2 * 32) + k_cnt;   // Image 2 영역
            assign buf_r_addr[3] = (3 * 32) + k_cnt;   // Image 3 영역
        
            // Input MUX with Zero-Padding for Skew Flush
            // k_cnt가 입력해야 할 데이터 길이(cur_input_len)보다 작은가? 
            // 즉, 아직 입력할 진짜 데이터가 남았다면, raw_input_data를 통과시키고,
            // 아니라면 모두 0을 통과시킨다.
            assign sys_row_in = (k_cnt < cur_input_len) ? raw_input_data : {ARRAY_ROW*dataWidth{1'b0}};
            assign sys_col_in = (k_cnt < cur_input_len) ? raw_weight_data : {ARRAY_COL*dataWidth{1'b0}};
            
            // 결과 출력 연결
            assign o_result_class_0 = mf_out[0];
            assign o_result_class_1 = mf_out[1];
            assign o_result_class_2 = mf_out[2];
            assign o_result_class_3 = mf_out[3];
        
            // Packing ↔ Unpacking
            genvar i, j;
            generate
                // PE 결과 unpack: array_out_packed → pe_res_unpacked[img][neuron]
                for (i = 0; i < ARRAY_ROW; i = i + 1) begin : UNPACK_IMG
                    for (j = 0; j < ARRAY_ROW; j = j + 1) begin : UNPACK_NEURON
                        assign pe_res_unpacked[i][j] = 
                            array_out_packed[((i*ARRAY_ROW+j)+1)*2*dataWidth-1 : (i*ARRAY_ROW+j)*2*dataWidth];
                    end
                end
                // Bias unpack: b_bank_out → bias_unpacked[neuron]
                for (j = 0; j < 4; j = j + 1) begin : UNPACK_BIAS
                    assign bias_unpacked[j] = b_bank_out[(j+1)*2*dataWidth-1 : j*2*dataWidth];
                end
            endgenerate

    - 2️⃣ DUT Instantiation
 
            // ======================================================================================
            // 5. Module Instantiations
            // ======================================================================================
        
            // ----- Weight Bank -----
            // Layer/Group/Address에 따라 4개 뉴런의 가중치를 동시 출력
            Weight_Bank #(
                .dataWidth(dataWidth),
                .ARRAY_ROW(ARRAY_ROW),
                .ARRAY_COL(ARRAY_COL)
            ) wb (
                .i_clk(i_clk),
                .i_layer_idx(cur_layer_num),
                .i_neuron_group(group_cnt),
                .i_w_addr(k_cnt),
                .w_out_packed(w_bank_out)
            );
        
            // ----- Bias Bank -----
            // Layer/Group에 따라 4개 뉴런의 Bias를 동시 출력
            Bias_Bank #(
                .dataWidth(dataWidth),
                .ARRAY_ROW(ARRAY_ROW),
                .ARRAY_COL(ARRAY_COL)
            ) bb (
                .i_clk(i_clk),
                .i_layer_idx(cur_layer_num),
                .i_neuron_group(group_cnt),
                .b_out_packed(b_bank_out)
            );
        
            // ----- Data Skewing -----
            // Raw 데이터에 대각선 Wavefront 지연을 적용한 후 Systolic Array로 전달
            Data_Skewing #(
                .dataWidth(dataWidth),
                .ARRAY_ROW(ARRAY_ROW),
                .ARRAY_COL(ARRAY_COL)
            ) ds (
                .i_clk(i_clk),
                .i_rst(pe_rst),               // PE 리셋과 동기화 (새 그룹 시작 시 SR 초기화)
                .i_en(pe_en),                 // PE enable과 동기화
                .i_row_data(sys_row_in),      // MUX 출력 (Raw, zero-padded)
                .i_col_data(sys_col_in),      // MUX 출력 (Raw, zero-padded)
                .o_row_data(skewed_row_data), // Skewed → SA row edge
                .o_col_data(skewed_col_data)  // Skewed → SA col edge
            );
        
            // ----- Systolic Array -----
            Systolic_Array #(
                .dataWidth(dataWidth),
                .ARRAY_ROW(ARRAY_ROW),
                .ARRAY_COL(ARRAY_COL)
            ) sa (
                .i_clk(i_clk),
                .i_rst(pe_rst),
                .i_en(pe_en),
                .i_row_data(skewed_row_data),   // Raw → Skewed
                .i_col_data(skewed_col_data),   // Raw → Skewed
                .o_array_out(array_out_packed)
                `ifdef DEBUG
                , .o_debug_pe00_mul(debug_mul_00)
                `endif
            );
        
            // ----- Activation Unit -----
            Activation_Unit #(
                .dataWidth(dataWidth)
            ) au (
                .i_clk(i_clk),
                .i_psum(au_in_psum),
                .i_bias(au_in_bias),
                .i_actsel(cur_act_sel),
                .o_out(act_out_val)
            );
        
            // ----- Unified Buffer -----
            Unified_Buffer #(
                .dataWidth(dataWidth)
            ) ub (
                .i_clk(i_clk),
                .i_wen(buf_wen),
                .i_w_addr(buf_w_addr),
                .i_w_data(buf_w_data),
                .i_r_addr_0(buf_r_addr[0]),
                .i_r_addr_1(buf_r_addr[1]),
                .i_r_addr_2(buf_r_addr[2]),
                .i_r_addr_3(buf_r_addr[3]),
                .o_r_data_0(buf_r_data[0]),
                .o_r_data_1(buf_r_data[1]),
                .o_r_data_2(buf_r_data[2]),
                .o_r_data_3(buf_r_data[3])
            );
        
            // ----- MaxFinder (4 instances, 각 이미지별) -----
            genvar img;
            generate
                for (img = 0; img < ARRAY_ROW; img = img + 1) begin : MAX_FINDER_GEN
                    maxFinder #(
                        .numInput(10),
                        .inputWidth(dataWidth)
                    ) mf (
                        .i_clk(i_clk),
                        .i_data(mf_in_data[img]),
                        .i_valid(mf_valid_pulse),
                        .o_data(mf_out[img]),
                        .o_data_valid(mf_out_valid[img])
                    );
                end
            endgenerate

    - 3️⃣ FSM Definition
 
                // ======================================================================================
                // 7. FSM Controller
                // ======================================================================================
            
                always @(posedge i_clk or negedge i_rst_n) begin
                    if (!i_rst_n) begin
                        // ===== 리셋 시 초기화 =====
                        state <= IDLE;           //대기 상태로 시작
                        k_cnt <= 0;              //모든 카운터 = 0
                        group_cnt <= 0;      
                        write_seq_cnt <= 0;
                        cur_layer_num <= 0;
                        cur_input_len <= 0;
                        cur_neuron_total <= 0;
                        cur_act_sel <= 0;
                        pe_rst <= 1;              // PE와 skewing SR 리셋 유지
                        pe_en <= 0;               // MAC 연산 비활성화
                        buf_wen <= 0;             // Buffer 쓰기 비활성화
                        buf_w_addr <= 0;
                        buf_w_data <= 0;
                        mf_valid_pulse <= 0;
                        o_done_interrupt <= 0;   
            
                    end else begin
                        case (state)
                            // ============================================================
                            // [IDLE] 추론 시작 대기
                            // ============================================================
                            IDLE: begin
                                pe_rst <= 1;
                                buf_wen <= 0;
                                if (i_start_inference) begin //외부에서 추론 시작 트리거
                                    state <= CALC_L1;
                                    cur_layer_num <= 1;
                                    cur_input_len <= `numWeightLayer1;     // 784
                                    cur_neuron_total <= `numNeuronLayer1;  // 30
                                    cur_act_sel <= SIG;                    // Sigmoid
                                    k_cnt <= 0;
                                    group_cnt <= 0;
                                    pe_rst <= 1;            // PE 누적기 초기화
                                    raw_input_data <= 0;
                                    raw_weight_data <= 0;
                                end
                            end
            
                            // ============================================================
                            // [CALC_L1] Layer 1
                            // ============================================================
                            CALC_L1: begin
                                pe_rst <= 0; //PE 리셋 해제 및 ACC 시작
            
                                // k_cnt가 유효 범위 내에서만 PE 활성화
                                if (k_cnt <= cur_input_len - 1 + 2*(ARRAY_ROW - 1)) pe_en <= 1;
                                else pe_en <= 0;
            
                                // Layer 1은 외부 입력 사용
                                raw_input_data <= i_input_pixels;  // TB에서 매 사이클 갱신
                                raw_weight_data <= w_bank_out;     // Weight Bank에서 자동 출력
            
                                // 입력 시퀀스 종료 조건
                                if (k_cnt == cur_input_len - 1 + 2*(ARRAY_ROW - 1)) begin
                                    pe_en <= 0;
                                    state <= BUFFER_WR_L1;
                                    write_seq_cnt <= 0;
                                end else begin
                                    k_cnt <= k_cnt + 1;
                                end
                            end
            
                            // ============================================================
                            // [BUFFER_WR_L1] Layer 1 결과 저장
                            // ============================================================
                            BUFFER_WR_L1: begin
                                //16 PE 결과를 순차적으로 처리
                                if (write_seq_cnt > 0 && write_seq_cnt <= 16) begin
                                    if (wr_global_neuron_idx < cur_neuron_total) begin
                                        buf_wen <= 1;
                                        buf_w_addr <= (wr_img_idx * 32) + wr_global_neuron_idx;
                                        buf_w_data <= act_out_val;
                                    end
                                end
                                // 16개 PE 처리 완료 시 다음 동작 결정
                                if (write_seq_cnt == 16) begin
                                    // 모든 뉴런 그룹 처리 완료?
                                    if ((group_cnt + 1) * 4 >= cur_neuron_total) begin
            /*
            cur_neuron_total = 30 (Layer 1)
            
            Group 0: (0+1)*4 = 4  < 30  → 다음 그룹
            Group 1: (1+1)*4 = 8  < 30  → 다음 그룹
            ...
            Group 6: (6+1)*4 = 28 < 30  → 다음 그룹
            Group 7: (7+1)*4 = 32 >= 30 → Layer 완료, CALC_L2로 전이
            */
                                        // Layer 1 완료 → Layer 2로 전환
                                        state <= CALC_L2;
                                        cur_layer_num <= 2;
                                        cur_input_len <= `numNeuronLayer1;     // 30
                                        cur_neuron_total <= `numNeuronLayer2;  // 20
                                        cur_act_sel <= SIG;
                                        group_cnt <= 0;
                                        k_cnt <= 0;
                                        pe_rst <= 1;
                                    end else begin
                                        // 다음 뉴런 그룹 처리
                                        state <= CALC_L1;
                                        group_cnt <= group_cnt + 1;
                                        k_cnt <= 0;
                                        pe_rst <= 1;
                                    end
                                    write_seq_cnt <= 0;
                                end else begin
                                    write_seq_cnt <= write_seq_cnt + 1;
                                end
                            end

      - IDLE 전이 동작       
        1) state ← CALC_L1: Layer 1 연산 시작
        2) cur_layer_num ← 1: Weight/Bias Bank 선택
        3) cur_input_len ← 784: MNIST 이미지 크기
        4) cur_neuron_total ← 30: Layer 1 뉴런 수
        5) cur_act_sel ← SIG: Sigmoid 활성화
        6) group_cnt ← 0: 첫 번째 뉴런 그룹
        7) k_cnt ← 0: 입력 시퀀스 초기화

      - CALC_L1 전이 동작      
        1) PE 리셋 해제
        2) PE 활성화 조건에 따른 pe_en <= 1 ;
            - <cur_input_len - 1 (입력 종료 시점)>
              - 마지막 데이터(Data_Last)가 어레이의 입구(0행, 0열)에 들어가는 시간
              - 0-based index이므로 -1
            - <+ (ARRAY_ROW - 1) (Vertical Skew, 행 지연)>
              - Row 0에서 Row 3으로 갈수록 입력이 한 사이클 늦게 들어감 
              - Row 3까지 데이터가 내려가는 데 걸리는 시간을 고려
            - <+ (ARRAY_ROW - 1) (Horizontal Propagation, 열 전파)>
              - 데이터가 행에 도착한 뒤, 왼쪽(Col 0)에서 오른쪽 끝(Col 3)까지 전달되는 시간을 고려

                    [Cycle T]      : D_last가 PE(0,0)에 입력됨 (cur_input_len - 1)
                          ↓
                    [Cycle T + 1]  : PE(1,0) 도착
                          ↓
                    [Cycle T + 2]  : PE(2,0) 도착
                          ↓
                    [Cycle T + 3]  : PE(3,0) 도착 (가장 아래 행 도달) -> 첫 번째 (ARRAY_ROW - 1)
                          →
                    [Cycle T + 4]  : PE(3,1) 이동
                          →
                    [Cycle T + 5]  : PE(3,2) 이동
                          →
                    [Cycle T + 6]  : PE(3,3) 도착 (가장 오른쪽 열 도달) -> 두 번째 (ARRAY_ROW - 1)

        3) MUX 선택
        4) State Transition (그렇지 않을 경우 매 사이클 k_cnt++하여 입력 시퀀스 진행)

                state ← BUFFER_WR_L1 
                pe_en ← 0 
                write_seq_cnt ← 0

      - BUFFER_WR_L1 전이 동작     
        1) Buffer 쓰기 로직
       
                Cycle N (write_seq_cnt=0):
                  - 대기
                  - write_seq_cnt++
                
                Cycle N+1 (write_seq_cnt=1):
                  - in_idx = (0,0) → PE[0][0] 선택 → AU 입력
                  - delayed_seq_cnt = 0 (음수 방지)
                  - wr_idx 계산 안됨
                  - buf_wen = 0 (아직 유효 데이터 없음) 
                
                Cycle N+2 (write_seq_cnt=2):
                  - in_idx = (0,1) → PE[0][1] 선택
                  - wr_idx = (0,0) → PE[0][0] 결과 쓰기
                  - buf_w_addr = (0 × 32) + (0×4+0) = 0
                  - buf_w_data = act_out_val (Cycle N+1의 AU 출력)
                  - buf_wen = 1
                
                ...
                
                Cycle N+17 (write_seq_cnt=17):
                  - in_idx = overflow (사용 안됨)
                  - wr_idx = (3,3) → PE[3][3] 결과 쓰기
                  - buf_w_addr = (3 × 32) + (0×4+3) = 99
                  - buf_wen = 1
                
                주소 계산 : 
                buf_w_addr = (wr_img_idx × 32) + wr_global_neuron_idx
                wr_global_neuron_idx = group_cnt * 4 + wr_neu_idx
           
                주소 계산 예시 :
                wr_idx=(0,0): addr = 0*32 + 0*4+0 = 0   (Image 0, Neuron 0)
                wr_idx=(0,1): addr = 0*32 + 0*4+1 = 1   (Image 0, Neuron 1)
                wr_idx=(1,0): addr = 1*32 + 0*4+0 = 32  (Image 1, Neuron 0)
                wr_idx=(3,3): addr = 3*32 + 0*4+3 = 99  (Image 3, Neuron 3)

        2) 다음 동작 결정

                cur_neuron_total = 30 (Layer 1)
                
                Group 0: (0+1)*4 = 4  < 30  → 다음 그룹
                Group 1: (1+1)*4 = 8  < 30  → 다음 그룹
                ...
                Group 6: (6+1)*4 = 28 < 30  → 다음 그룹
                Group 7: (7+1)*4 = 32 >= 30 → Layer 완료, CALC_L2로 전이

        3) 레이어 완료 시 초기화
                // ============================================================
                // [CALC_L2] Layer 2 연산
                // ============================================================
                CALC_L2: begin
                    pe_rst <= 0;

                    if (k_cnt <= cur_input_len - 1 + 2*(ARRAY_ROW - 1)) pe_en <= 1;
                    else pe_en <= 0;

                    // Buffer에서 이전 레이어 출력 읽어서 Row 데이터로 사용
                    raw_input_data <= {buf_r_data[3], buf_r_data[2], buf_r_data[1], buf_r_data[0]};
                    raw_weight_data <= w_bank_out;

                    if (k_cnt == cur_input_len - 1 + 2*(ARRAY_ROW - 1)) begin
                        state <= BUFFER_WR_L2;
                        pe_en <= 0;
                        write_seq_cnt <= 0;
                    end else begin
                        k_cnt <= k_cnt + 1;
                    end
                end

                // ============================================================
                // [BUFFER_WR_L2] Layer 2 결과 저장
                // ============================================================
                BUFFER_WR_L2: begin
                    if (write_seq_cnt > 0 && write_seq_cnt <= 16) begin
                        if (wr_global_neuron_idx < cur_neuron_total) begin
                            buf_wen <= 1;
                            buf_w_addr <= (wr_img_idx * 32) + wr_global_neuron_idx;
                            buf_w_data <= act_out_val;
                        end
                    end

                    if (write_seq_cnt == 16) begin
                        if ((group_cnt + 1) * 4 >= cur_neuron_total) begin
                            state <= CALC_L3;
                            cur_layer_num <= 3;
                            cur_input_len <= `numNeuronLayer2;     // 20
                            cur_neuron_total <= `numNeuronLayer3;  // 10
                            cur_act_sel <= SIG;
                            group_cnt <= 0;
                            k_cnt <= 0;
                            pe_rst <= 1;
                        end else begin
                            state <= CALC_L2;
                            group_cnt <= group_cnt + 1;
                            k_cnt <= 0;
                            pe_rst <= 1;
                        end
                        write_seq_cnt <= 0;
                    end else begin
                        write_seq_cnt <= write_seq_cnt + 1;
                    end
                end
                .
                .
                .
                CALC_L3 / BUFFER_WR_L3 생략


