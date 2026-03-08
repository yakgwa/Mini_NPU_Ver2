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

      - CALC_L2/CALC_L3 전이 동작
        - CALC_L1 전이 동작과 큰 차이점은 없지만, 핵심적인 차이점은 다음과 같다.

                // CALC_L1:
                raw_input_data <= i_input_pixels;  // 외부 입력
                
                // CALC_L2:
                raw_input_data <= {buf_r_data[3], buf_r_data[2], buf_r_data[1], buf_r_data[0]};  // Buffer 읽기
                
                ------------------------------------------------------------------------
                
                //Buffer 읽기 데이터 패킹
                raw_input_data[31:0] = {
                    buf_r_data[3][7:0],  // Image 3
                    buf_r_data[2][7:0],  // Image 2
                    buf_r_data[1][7:0],  // Image 1
                    buf_r_data[0][7:0]   // Image 0
                }
                
                //읽기 주소
                buf_r_addr[0] = 0*32 + k_cnt = k_cnt      // Image 0 영역
                buf_r_addr[1] = 1*32 + k_cnt = 32 + k_cnt // Image 1 영역
                buf_r_addr[2] = 2*32 + k_cnt = 64 + k_cnt // Image 2 영역
                buf_r_addr[3] = 3*32 + k_cnt = 96 + k_cnt // Image 3 영역
                
                -------------------------------------------------------------------------
                데이터 재사용:
                Layer 1의 출력 (30개 뉴런)이 Layer 2의 입력
                Unified Buffer가 중간 결과 저장
                외부 메모리 접근 불필요 (On-chip 완료)

      - BUFFER_WR_L2/BUFFER_WR_L3 전이 동작
       
            BUFFER_WR_L1과 동일
            --------------------------------------------------------------------------
            Buffer 재사용
            Layer 2 결과를 동일한 Buffer에 덮어쓰기
            주소: (img_idx × 32) + neuron_idx
            Layer 1 결과는 더이상 필요없으므로 OK
            cur_neuron_total = 10
            
            Group 0: (0+1)*4 = 4  < 10  → 다음 그룹
            Group 1: (1+1)*4 = 8  < 10  → 다음 그룹
            Group 2: (2+1)*4 = 12 >= 10 → Layer 완료, OUTPUT_SCAN으로 전이
            
            최종 출력 위치 :
            Image 0: Addr 0~9   (10개 뉴런)
            Image 1: Addr 32~41
            Image 2: Addr 64~73
            Image 3: Addr 96~105

                        // ============================================================
                        // [OUTPUT_SCAN] Buffer에서 최종 10개 결과를 MaxFinder로 전달
                        // ============================================================
                        OUTPUT_SCAN: begin
                            if (k_cnt < 10) begin
                                mf_in_data[0][k_cnt*dataWidth +: dataWidth] <= buf_r_data[0];
                                mf_in_data[1][k_cnt*dataWidth +: dataWidth] <= buf_r_data[1];
                                mf_in_data[2][k_cnt*dataWidth +: dataWidth] <= buf_r_data[2];
                                mf_in_data[3][k_cnt*dataWidth +: dataWidth] <= buf_r_data[3];
                            end
        
                            if (k_cnt == 10) begin
                                mf_valid_pulse <= 1;
                                state <= DONE;
                                o_done_interrupt <= 1;
                            end else begin
                                k_cnt <= k_cnt + 1;
                            end
                        end
        
                        // ============================================================
                        // [DONE] 추론 완료 (대기)
                        // ============================================================
                        DONE: begin
                            // 아무것도 하지 않음. 외부 리셋으로 재시작.
                        end
                    endcase
                end
            end

      - OUTPUT_SCAN 전이 동작       
        1) 10개 출력 수집
    
                if (k_cnt < 10) begin
                    mf_in_data[0][k_cnt*dataWidth +: dataWidth] <= buf_r_data[0];
                    mf_in_data[1][k_cnt*dataWidth +: dataWidth] <= buf_r_data[1];
                    mf_in_data[2][k_cnt*dataWidth +: dataWidth] <= buf_r_data[2];
                    mf_in_data[3][k_cnt*dataWidth +: dataWidth] <= buf_r_data[3];
                end
                ----------------------------------------------------------------
                비트 슬라이싱 : 
                k_cnt = 0: mf_in_data[img][0*8 +: 8]  = [7:0]   = buf_r_data[img]
                k_cnt = 1: mf_in_data[img][1*8 +: 8]  = [15:8]  = buf_r_data[img]
                ...
                k_cnt = 9: mf_in_data[img][9*8 +: 8]  = [79:72] = buf_r_data[img]
                ----------------------------------------------------------------
                Buffer 읽기 주소 :
                k_cnt = 0:
                  buf_r_addr[0] = 0   → Image 0, Neuron 0
                  buf_r_addr[1] = 32  → Image 1, Neuron 0
                  buf_r_addr[2] = 64  → Image 2, Neuron 0
                  buf_r_addr[3] = 96  → Image 3, Neuron 0
                
                k_cnt = 9:
                  buf_r_addr[0] = 9   → Image 0, Neuron 9
                  buf_r_addr[1] = 41  → Image 1, Neuron 9
                  buf_r_addr[2] = 73  → Image 2, Neuron 9
                  buf_r_addr[3] = 105 → Image 3, Neuron 9
                ----------------------------------------------------------------
                Cycle N+0 (k_cnt=0): buf_r_addr = {..., 64, 32, 0}
                Cycle N+1: buf_r_data 유효, mf_in_data[x][7:0] 저장, k_cnt=1
                Cycle N+2: buf_r_data 유효, mf_in_data[x][15:8] 저장, k_cnt=2
                ...
                Cycle N+10: buf_r_data 유효, mf_in_data[x][79:72] 저장, k_cnt=10
      

        2) MaxFinder 트리거
       
                if (k_cnt == 10) begin
                    mf_valid_pulse <= 1;
                    state <= DONE;
                    o_done_interrupt <= 1;
                end

            - mf_valid_pulse = 1: MaxFinder 동작 트리거 (1-cycle 펄스)
            - state ← DONE: 완료 상태로 전이
            - o_done_interrupt = 1: 외부 시스템에 인터럽트 발생

### TB_NPU_TOP.sv

- 0️⃣ Parameter & Port 정의

        module TB_NPU_Top();
            localparam dataWidth = `dataWidth;       
            localparam TIMEOUT_CYCLES = 50000;        
        
            reg                          clk;
            reg                          rst_n;
            reg                          start_inference;
            reg  [4*dataWidth-1:0]       input_pixels;
            reg                          input_valid;
            wire                         done;
            wire [31:0]                  result_0, result_1, result_2, result_3;
        
            // [Image Memory]
            // 각 이미지: 784 pixels (index 0~783) + 1 label (index 784) = 785 entries
            reg [dataWidth-1:0] img_mem_0 [0:784];
            reg [dataWidth-1:0] img_mem_1 [0:784];
            reg [dataWidth-1:0] img_mem_2 [0:784];
            reg [dataWidth-1:0] img_mem_3 [0:784];
        
            // [Log File Handle]
            int log_fd;          
        
            // [Counters]
            int cycle_cnt;      
            int pass_count;
            int total_count;
        
            // [State Tracking]
            reg [3:0] prev_state;    // 이전 사이클 FSM 상태

- 1️⃣ DUT Instantiation

        // ======================================================================================
        // DUT Instantiation
        // ======================================================================================
        NPU_Top #(
            .dataWidth(dataWidth)
        ) dut (
            .i_clk(clk),
            .i_rst_n(rst_n),
            .i_start_inference(start_inference),
            .i_input_pixels(input_pixels),
            .i_input_valid(input_valid),
            .o_done_interrupt(done),
            .o_result_class_0(result_0),
            .o_result_class_1(result_1),
            .o_result_class_2(result_2),
            .o_result_class_3(result_3)
        );
        always #5 clk = ~clk;

- 2️⃣ Function / Task Definition

        // ======================================================================================
        // State Name Function
        // ======================================================================================
       function string get_state_name(input [3:0] st);
           case (st)
               0: return "IDLE";
               1: return "CALC_L1";
               2: return "BUFFER_WR_L1";
               3: return "CALC_L2";
               4: return "BUFFER_WR_L2";
               5: return "CALC_L3";
               6: return "BUFFER_WR_L3";
               7: return "OUTPUT_SCAN";
               8: return "DONE";
               default: return "UNKNOWN";
           endcase
       endfunction

        // ======================================================================================
        // Task: Log Matrix Visualization
        // ======================================================================================
        task log_matrix_view;
            begin
                // --- Header ---
                $fdisplay(log_fd, "");
                $fdisplay(log_fd, "====== [Cycle:%5d] State: %0s | k_cnt:%4d | Group:%2d | pe_en:%b | pe_rst:%b ======",
                    cycle_cnt, get_state_name(dut.state), dut.k_cnt, dut.group_cnt, dut.pe_en, dut.pe_rst);
    
                // --- Raw Data (Memory readout, before skewing) ---
                $fdisplay(log_fd, "  [Raw Data (MUX Output)]");
                $fdisplay(log_fd, "    Row: R0=%4d  R1=%4d  R2=%4d  R3=%4d",
                    $signed(dut.sys_row_in[1*dataWidth-1 -: dataWidth]),
                    $signed(dut.sys_row_in[2*dataWidth-1 -: dataWidth]),
                    ..
                $fdisplay(log_fd, "    Col: C0=%4d  C1=%4d  C2=%4d  C3=%4d",
                    $signed(dut.sys_col_in[1*dataWidth-1 -: dataWidth]),
                    $signed(dut.sys_col_in[2*dataWidth-1 -: dataWidth]),
                    ..
    
                // --- Skewed Data (after Data_Skewing, actual array edge values) ---
                $fdisplay(log_fd, "  [Skewed Data (Array Edge)]");
                $fdisplay(log_fd, "    Row: R0=%4d  R1=%4d  R2=%4d  R3=%4d",
                    $signed(dut.skewed_row_data[1*dataWidth-1 -: dataWidth]),
                    ..
                $fdisplay(log_fd, "    Col: C0=%4d  C1=%4d  C2=%4d  C3=%4d",
                    $signed(dut.skewed_col_data[1*dataWidth-1 -: dataWidth]),
                    ..

    - 먼저 Raw Data와 Skewed Data를 분리하여 검증한다.
    - 메모리나 버퍼에서 방금 읽어와 MUX를 통과한 Raw Data는 아직 지연(Delay)이 적용되지 않았으므로, Row 0..3, Col 0..3 모두 동일한 시점(k)의 데이터가 들어와 있는지 확인한다. 
    - 만약 여기서부터 데이터가 이상하다면 메모리 읽기 로직의 문제라고 판단할 수 있다.
    - Data_Skewing을 통과하여 지연이 적용된 후, 실제로 PE Array 입구에 도착한 데이터로서, Row/Col 별 지연이 제대로 먹혔는지 확인한다.

                // --- Wavefront Queue Visualization ---
                $fdisplay(log_fd, "  [Wavefront Entry Pattern]");
                $fdisplay(log_fd, "                           C0=%4d  C1=%4d  C2=%4d  C3=%4d  <-- Weight (Col)",
                    $signed(dut.skewed_col_data[1*dataWidth-1 -: dataWidth]),
                    $signed(dut.skewed_col_data[2*dataWidth-1 -: dataWidth]),
                    $signed(dut.skewed_col_data[3*dataWidth-1 -: dataWidth]),
                    $signed(dut.skewed_col_data[4*dataWidth-1 -: dataWidth]));
                $fdisplay(log_fd, "                           |       |       |       |");
                $fdisplay(log_fd, "                           v       v       v       v");
                $fdisplay(log_fd, "  R0=%4d  ---------->  [PE00]  [PE01]  [PE02]  [PE03]",
                    $signed(dut.skewed_row_data[1*dataWidth-1 -: dataWidth]));
                $fdisplay(log_fd, "  R1=%4d  ---------->  [PE10]  [PE11]  [PE12]  [PE13]",
                    $signed(dut.skewed_row_data[2*dataWidth-1 -: dataWidth]));
                $fdisplay(log_fd, "  R2=%4d  ---------->  [PE20]  [PE21]  [PE22]  [PE23]",
                    $signed(dut.skewed_row_data[3*dataWidth-1 -: dataWidth]));
                $fdisplay(log_fd, "  R3=%4d  ---------->  [PE30]  [PE31]  [PE32]  [PE33]",
                    $signed(dut.skewed_row_data[4*dataWidth-1 -: dataWidth]));

    - skewing이 적용된 데이터를 실제 적용함으로써 위쪽(C0~C3)에서 가중치가 내려오고, 왼쪽(R0..R3)에서 입력 데이터가 들어가는 모습을 시각적으로 체크한다. 이를 통해 어떤 값이 어느 PE로 들어가서 만나는지 직관적으로 파악하여 디버깅 속도를 높이도록 한다.

                    // --- PE Accumulator Matrix ---
                    $fdisplay(log_fd, "  [PE Accumulator Matrix] (N = Neuron %0d~%0d)",
                        dut.group_cnt*4, dut.group_cnt*4+3);
                    $fdisplay(log_fd, "                  N%-2d        N%-2d        N%-2d        N%-2d",
                        dut.group_cnt*4, dut.group_cnt*4+1, dut.group_cnt*4+2, dut.group_cnt*4+3);
                    $fdisplay(log_fd, "  Img0(R0) --> [%7d] [%7d] [%7d] [%7d]",
                        $signed(dut.pe_res_unpacked[0][0]),
                        $signed(dut.pe_res_unpacked[0][1]),
                        $signed(dut.pe_res_unpacked[0][2]),
                        $signed(dut.pe_res_unpacked[0][3]));
                    $fdisplay(log_fd, "  Img1(R1) --> [%7d] [%7d] [%7d] [%7d]",
                        $signed(dut.pe_res_unpacked[1][0]),
                        $signed(dut.pe_res_unpacked[1][1]),
                        $signed(dut.pe_res_unpacked[1][2]),
                        $signed(dut.pe_res_unpacked[1][3]));
                    $fdisplay(log_fd, "  Img2(R2) --> [%7d] [%7d] [%7d] [%7d]",
                        $signed(dut.pe_res_unpacked[2][0]),
                        $signed(dut.pe_res_unpacked[2][1]),
                        $signed(dut.pe_res_unpacked[2][2]),
                        $signed(dut.pe_res_unpacked[2][3]));
                    $fdisplay(log_fd, "  Img3(R3) --> [%7d] [%7d] [%7d] [%7d]",
                        $signed(dut.pe_res_unpacked[3][0]),
                        $signed(dut.pe_res_unpacked[3][1]),
                        $signed(dut.pe_res_unpacked[3][2]),
                        $signed(dut.pe_res_unpacked[3][3]));
                end
            endtask

    - 각 PE 내부의 acc register valude(dut.pe_res_unpacked[row][col]) 즉, Partial Sum을 추출하여 Reference Model과 일치하는지 비교하여 연산의 정확성을 검증하도록 한다.
 
                // ======================================================================================
                // Task: Log BUFFER_WR activity (Bias + Activation)
                // ======================================================================================
                task log_buffer_write;
                    begin
                        if (dut.buf_wen) begin
                            $fdisplay(log_fd, "  [BUF_WR] Cycle:%5d | addr=%3d | data=%4d | act_out=%4d | AU_psum=%6d | AU_bias=%6d | seq=%2d",
                                cycle_cnt, dut.buf_w_addr, $signed(dut.buf_w_data),
                                $signed(dut.act_out_val),
                                $signed(dut.au_in_psum), $signed(dut.au_in_bias),
                                dut.write_seq_cnt);
                        end
                    end
                endtask

    - NPU_TOP.v의 buf_wen(Buffer Write Enable) 일 때, 즉, 버퍼에 쓰지 않는 타이밍에 발생하는 무의미한 데이터를 걸러내고, 실제로 메모리에 유효한 값이 저장되는 순간만 포착하도록 trigger condition을 걸어줌으로써 출력 로그를 남긴다.
 
                1. cycle_cnt: 현재 시각.
                2. dut.buf_w_addr (Destination): 결과값이 저장될 버퍼의 주소
                                                 다음 레이어 계산 때 이 주소에서 데이터를 읽어옴
                3. $signed(dut.buf_w_data) (Final Result): 버퍼에 실제로 기록되는 최종 값
                4. $signed(dut.act_out_val) (After Activation): Activation Function 통과 직후 값
                5. $signed(dut.au_in_psum) (Raw Input 1): PE Array에서 계산되어 나온 Partial Sum
                                                          (아직 Bias 더하기 전)
                6. $signed(dut.au_in_bias) (Raw Input 2): 메모리나 레지스터에서 가져온 Bias
                7. dut.write_seq_cnt: 한 번에 여러 데이터를 쓸 때(Sequential Write), 
                                      현재 몇 번째 데이터를 쓰고 있는지 나타내는 카운터
 
===============================================================
                // ======================================================================================
                // Task: Verify MIF Loading
                // ======================================================================================
                task verify_mif_loading;
                    begin
                        $fdisplay(log_fd, "============================================================");
                        $fdisplay(log_fd, "  MIF Loading Verification");
                        $fdisplay(log_fd, "============================================================");
            
                        // Weight Bank: Layer 1, Neuron 0의 첫 5개 가중치 샘플 출력
                        $fdisplay(log_fd, "  [Weight] L1_N0 w[0]=%d, w[1]=%d, w[2]=%d, w[3]=%d, w[4]=%d",
                            $signed(dut.wb.w_l1[0]),
                            ..
                            $signed(dut.wb.w_l1[4]));
            
                        // Bias Bank: Layer 1의 첫 4개 Bias 샘플
                        $fdisplay(log_fd, "  [Bias]   L1: b[0]=%d, b[1]=%d, b[2]=%d, b[3]=%d",
                            $signed(dut.bb.w_b_l1[0]),
                            ..
                            $signed(dut.bb.w_b_l1[3]));
            
                        // Image Memory: 첫 5개 픽셀 (TB 측)
                        $fdisplay(log_fd, "  [Image0] px[0]=%d, px[1]=%d, px[2]=%d, px[3]=%d, px[4]=%d, label=%d",
                            $signed(img_mem_0[0]), $signed(img_mem_0[1]),
                            $signed(img_mem_0[2]), $signed(img_mem_0[3]),
                            $signed(img_mem_0[4]), img_mem_0[784]);
                        $fdisplay(log_fd, "  [Image1] px[0]=%d, label=%d", $signed(img_mem_1[0]), img_mem_1[784]);
                        $fdisplay(log_fd, "  [Image2] px[0]=%d, label=%d", $signed(img_mem_2[0]), img_mem_2[784]);
                        $fdisplay(log_fd, "  [Image3] px[0]=%d, label=%d", $signed(img_mem_3[0]), img_mem_3[784]);
            
                        $fdisplay(log_fd, "  MIF Loading Done.");
                        $fdisplay(log_fd, "============================================================");
            
                        // Console echo
                        $display("[TB] MIF Loading Done. Check log file for details.");
                    end
                endtask

    - Layer 1의 첫 번째 뉴런(N0)에 할당된 가중치 중 앞부분 5개(w[0]~w[4]) 확인
    - Layer 1의 뉴런 0, 1, 2, 3번에 더해질 Bias 4개 확인
    - img_mem_X[0] 등 앞부분 픽셀 값을 찍어보며 이미지가 로드 여부 확인
    - img_mem_X[784] 위치에 저장된 정답 라벨 확인
















