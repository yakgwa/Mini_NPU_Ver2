## [AI Accelerator 기초] - Systolic Array와 FSM 구현3 (2D 4X4 Systolic Array +  Controller + ReLU)

### Systolic Array  + Controller + Activation Function(ReLU)

- DUT
1️⃣2️⃣3️⃣4️⃣5️⃣6️⃣7️⃣8️⃣9️⃣🔟
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







      - pe_systolic_cell은 기존에 최초 제시된 mac_pe와는 다르게, 연산기(mac_pe) + 입력 전달 목적 pipe register + next cell interconnect까지 포함된 Systolic Array용 PE cell이라고 요약할 수 있다. 
