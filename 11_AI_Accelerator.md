## [AI Accelerator 기초] - Systolic Array와 FSM 구현3 (2D 4X4 Systolic Array +  Controller + ReLU)

### Systolic Array  + Controller + Activation Function(ReLU)

- DUT

    - 0️⃣ Parameter & Port 정의

          module systolic_controller_relu #(
          parameter integer K_DIM     = 4,
          parameter integer DATA_W    = 8,
          parameter integer ACC_W     = 2*DATA_W + $clog2(K_DIM),
          parameter integer ROWS      = 4,
          parameter integer COLS      = 4,
          // [추가] 활성화 함수용 파라미터 (Threshold/Bias)
          // 이 값보다 작은 연산 결과는 노이즈로 간주하고 0으로 만듦
          parameter integer BIAS      = 30,
          // Inner State Definition
          parameter integer IDLE      = 0,
          parameter integer RUN       = 1,
          parameter integer DONE      = 2,
          // skew 주입 횟수 (4X4 MAC 기준)
          // 쉽게 말해, 행/열마다 PE에 최초로 주입되는 입력 값이 딜레이가 생기는 만큼,
          // Systolic Array의 PE가 전부 계산될 때 까지 주입해야 하는 시간을 말한다.
          parameter integer INJ_CYCLES = K_DIM + ROWS + COLS - 2
          )(
          input  wire                                  clk,
          input  wire                                  rst_n,
          input  wire                                  i_start,        //"이번 연산 시작" 트리거
          input  wire [ROWS*K_DIM*DATA_W-1:0]          i_mat_a,        //packed port A
          input  wire [COLS*K_DIM*DATA_W-1:0]          i_mat_b,        //packed port B
        
          output wire                                  o_busy,         //RUN 동안 1
          output wire                                  o_done,         //DONE 동안 1
          output wire [ROWS*COLS*ACC_W-1:0]            o_mat           //packed Output
          );

    - 1️⃣ Parameter & Port 정의

            //==========================================================
            //1. FSM(based Moore Machine) & Counter
            //==========================================================
            reg [7:0] cnt;        //Cycle Time for 'RUN'
            reg [1:0] state;      //state register
            reg [1:0] next_state; //next-state register
            wire array_en;        //array calc enable signal
            wire array_clr;       //inner array acc register reset
        
            //==========================================================
            //next_state : comb logic
            //==========================================================
                always @(*) begin
                    next_state = state; // 기본값 (Latch 방지)
                
                    case (state)
                        IDLE : if(i_start) next_state = RUN;
                        RUN  : if(cnt >= INJ_CYCLES) next_state = DONE;
                        DONE : if(!i_start) next_state = IDLE;
                        default: next_state = IDLE;
            /*
            IDLE→RUN: i_start 레벨이 1이면 시작.
            RUN→DONE: cnt가 충분히 커지면 done.
            DONE→IDLE: start가 0으로 내려가야(버튼 떼기처럼) 다시 대기.
            따라서 이 FSM은 start를 “펄스”가 아니라 “레벨”로 사용하고 있으므로,
            start를 계속 1로 유지하면 DONE에서 유지되고, 0으로 내려야 IDLE로 복귀.
            */
                    endcase
                end
            
            //==========================================================
            //state, cnt : seq logic
            //==========================================================
                always @(posedge clk or negedge rst_n) begin
                    if(!rst_n) begin
                        state <= IDLE;
                        cnt <= 0;
                    end else begin
                        state <= next_state;
                        if (state == RUN) cnt <= cnt + 1;
                        else cnt <= 0;
                    end
            /*
            state <= next_state와 cnt <= ...는 같은 always 블록의 nonblocking이므로,
            if (state == RUN)은 이전 사이클의 state 기준으로 판단됨.
            즉, RUN으로 “진입하는 사이클”에서 cnt가 증가할지 여부는 1사이클 차이가 날 수 있음
            */
                end
            
            //==========================================================
            //output, control signal
            //==========================================================
                //Output Assignments
                assign o_busy = (state == RUN);
                assign o_done = (state == DONE);
                //Array Control Signals
                assign array_en = (state == RUN);
                // IDLE 상태에서 start가 들어오는 순간 Clear 수행 (Accumulator 초기화)
                // 이렇게 하면 PE 내부의 acc_sum이 0으로 리셋되고 새 연산을 시작합니다.
                assign array_clr = (state == IDLE) && i_start;

    - 2️⃣ Input Buffer Latch + <Packed→Unpacked> Mapping 정의

            //==========================================================
            //2. Input Buffer Latch
            //Unpacked ↔ Packed Matching & Input Buffer(input data latch for store)
            //==========================================================
            reg [DATA_W-1:0] latched_mat_a [0:ROWS-1][0:K_DIM-1]; // ROW별 분리된 Unpacked Array
            reg [DATA_W-1:0] latched_mat_b [0:K_DIM-1][0:COLS-1]; // COL별 분리된 Unpacked Array
            
            integer i, k; // Matrix A용 인덱스
            integer j, m; // Matrix B용 인덱스
            always @(posedge clk) begin
                if (state == IDLE && i_start) begin
                    // Matrix A Latching
                    for (i = 0; i < ROWS; i = i + 1) begin
                        for (k = 0; k < K_DIM; k = k + 1) begin
                            // Flat Index = (현재 행 * 한 행의 크기 + 현재 열) * 데이터 폭
                            latched_mat_a[i][k] <= i_mat_a[ (i * K_DIM + k) * DATA_W +: DATA_W ];
                        end
                    end
                    // Matrix B Latching: [K_dim][Cols] 구조
                    for (m = 0; m < K_DIM; m = m + 1) begin
                        for (j = 0; j < COLS; j = j + 1) begin
                            // Flat Index = (현재 행 * 한 행의 크기 + 현재 열) * 데이터 폭
                            latched_mat_b[m][j] <= i_mat_b[ (m * COLS + j) * DATA_W +: DATA_W ];
                        end
                    end
                end
            end

    - 3️⃣ Data Skewing Logic

            //==========================================================
            //3. Data Skewing Logic
            //==========================================================
            //Array Interface : 실제 Systolic Array로 들어가는 신호 정의
            wire [ROWS*DATA_W-1:0] a_in_row;
            wire [COLS*DATA_W-1:0] b_in_col;
        
            genvar r, c;
            generate
                // Row Logic (Matrix A)
                for (r = 0; r < ROWS; r = r + 1) begin : GEN_ROW_DATA
                    assign a_in_row[r*DATA_W +: DATA_W] = 
                        ( (cnt >= r) && ((cnt - r) < K_DIM) ) ? latched_mat_a[r][cnt - r] : 
                                                                {DATA_W{1'b0}};
                end
                // Col Logic (Matrix B)
                for (c = 0; c < COLS; c = c + 1) begin : GEN_COL_DATA
                    // [수정 2] latched_mat_a -> latched_mat_b 로 변경
                    assign b_in_col[c*DATA_W +: DATA_W] = 
                        ( (cnt >= c) && ((cnt - c) < K_DIM) ) ? latched_mat_b[cnt - c][c] : 
                                                                {DATA_W{1'b0}};
                end
            endgenerate




            // Row Logic (Matrix A)
            for (r = 0; r < ROWS; r = r + 1) begin : GEN_ROW_DATA
                assign a_in_row[r*DATA_W +: DATA_W] = 
                    ( (cnt >= r) && ((cnt - r) < K_DIM) ) ? latched_mat_a[r][cnt - r] : 
                                                            {DATA_W{1'b0}};

      - row마다 하드웨어가 ROWS개가 복제된다. 이때, a_in_row를 각 row 입력을 bus에 차례대로 packing해주고, 이때, 아래와 같은 조건에 대해 latched_mat_a를 넣어줄지 결정한다.

        (cnt >= r) && ((cnt - r) < K_DIM)
        //cnt >= r (row가 시작했는가?), cnt - r < K_DIM (k가 범위 안인가?)

      - systolic array에서 A[r][k]를 row r에 순서대로 넣고 싶은데, row가 아래로 갈수록 늦게 시작해야 대각선 정렬이 된다. 따라서 row r은 r만큼 늦게 시작하게 만들고, 그 이후엔 k가 0,1,2,3 순서로 들어갈 수 있게 한다.
      - cnt=r일 때 → k=0, cnt=r+1일 때 → k=1, cnt=r+(K_DIM-1)일 때 → k=K_DIM-1
     - 즉, 조건이 참이면 실제로 latched_mat_a[r][cnt - r]이 입력되며 시간에 따라 A[r][0], A[r][1], ...이 들어가게 된다. 만약, 조건이 거짓이라면 빈 구간에는 0을 흘려보내서 연산에 영향을 없게 한다.

























      - pe_systolic_cell은 기존에 최초 제시된 mac_pe와는 다르게, 연산기(mac_pe) + 입력 전달 목적 pipe register + next cell interconnect까지 포함된 Systolic Array용 PE cell이라고 요약할 수 있다. 
