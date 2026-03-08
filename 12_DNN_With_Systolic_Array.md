## [DNN with Systolic Array] - PE, Systolic Array, Activation, Memory, Buffer 설계

### PE.v

- DUT

    module PE #(
        parameter integer dataWidth = 8
        )(
        input signed  [dataWidth-1:0]   i_a,        // op1
        input signed  [dataWidth-1:0]   i_b,        // op2
        
        input                           i_clk,      // clock
        input                           i_rst,      // reset
        input                           i_en,       // enable
        
        output signed [2*dataWidth-1:0] o_psum      // partial sum
        `ifdef DEBUG
      , output signed [2*dataWidth-1:0] o_debugmul  // debug output
        `endif
        );
        
        // 16bit ACC register
        reg signed [2*dataWidth-1:0] sum = 0;
        
        // 16bit Multiply Result
        wire signed [2*dataWidth-1:0] mul; 
        
        // 임시 덧셈 결과용 (Saturation Check)
        wire signed [2*dataWidth:0] Comboadd;     
        
        // 부호 비트 확인용
        wire sign_mul;
        wire sign_sum;
        wire sign_res;
    
        // 1. Multiply
        assign mul = i_a * i_b;
     
        // 2. Add (Pre-Calc)
        assign Comboadd = sum + mul;
    
        // 3. Signed-Bit 추출 (MSB)
        assign sign_mul = mul[2*dataWidth-1];
        assign sign_sum = sum[2*dataWidth-1];
        assign sign_res = Comboadd[2*dataWidth-1];
    
        `ifdef DEBUG
        assign o_debugmul = mul; 
        `endif
        
        always @(posedge i_clk) begin
            if(i_rst) begin
                sum <= 0;
            end else if (i_en) begin 
                // Case 1: 양수 + 양수 = 음수 (Positive Overflow)
                if (!sign_mul && !sign_sum && sign_res) begin
                    sum <= {1'b0, {(2*dataWidth-1){1'b1}}}; 
    
                // Case 2: 음수 + 음수 = 양수 (Negative Overflow)
                end else if (sign_mul && sign_sum && !sign_res) begin
                    sum <= {1'b1, {(2*dataWidth-1){1'b0}}};
    
                // Case 3: 정상 범위
                end else begin
                    sum <= Comboadd ;
                end
            end
        end
        
        assign o_psum = sum;
    
    endmodule

    - 1️⃣ FSM & Conter 정의

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

    - 4️⃣ PPU(ReLU + Bias)(즉, 후처리) + Output Mapping 정의

            //==========================================================
            // 4. Post-Processing Unit (ReLU + Bias)
            //==========================================================
            wire [ROWS*COLS*ACC_W-1:0] pe_acc_sum;
            generate
              for (r = 0; r < ROWS; r = r + 1) begin : GEN_PPU_ROW
                for (c = 0; c < COLS; c = c + 1) begin : GEN_PPU_COL
                  wire [ACC_W-1:0] raw; //o_mat으로 전달하기 위한 wire 
                  assign raw = pe_acc_sum[(r*COLS + c)*ACC_W +: ACC_W];
                  // Bias Subtraction (Signed)
                  wire signed [ACC_W:0] tmp;
                  assign tmp = $signed({1'b0, raw}) - $signed(BIAS);
                  // ReLU Activation Function + o_mat assign
                  assign o_mat[(r*COLS + c)*ACC_W +: ACC_W] =
                      (tmp < 0) ? {ACC_W{1'b0}} : tmp[ACC_W-1:0];
                end
              end
            endgenerate

      - Systolic Array가 만든 연산 결과(Packed Array, pe_acc_sum)를 각 PE 위치별로 partial하여 Bias를 빼고, ReLu Activation Function을 적용한 뒤, 다시 기존에 port로 정의한 출력(o_mat)에 꽂는 역할을 한다. 따라서 pe_acc_sum은 o_mat과 동일하게 wire 선언한다.

            wire [ROWS*COLS*ACC_W-1:0] pe_acc_sum;

      - 이후, PE(r,c)마다 동일한 PPU 로직을 복제하기 위해 generate 구문을 사용하여 ROWSXCOLS개의 동일한 조합논리 블록을 만들어 준다.

            generate
              for (r = 0; r < ROWS; r = r + 1) begin : GEN_PPU_ROW
                for (c = 0; c < COLS; c = c + 1) begin : GEN_PPU_COL

      - generate 구문 내부에서는 for문을 수행할 떄마다 누산 결과 하나를 저장하고 Bias + ReLU를 수행하여 o_mat에 전달할 수 있도록 output 비트 폭 만큼의 변수를 wire 선언할 필요가 있다.

            wire [ACC_W-1:0] raw;
            assign raw = pe_acc_sum[(r*COLS + c)*ACC_W +: ACC_W];

      - 이때 Bias를 빼면, 음수가 될 수 잇으므로 Signed로 선언하고, 부호 비트를 안전하게 표현하기 위해 비트 폭을 ACC_W+1로 선언한다. 이후, tmp로 bias subtraction 연산
     
            assign tmp = $signed({1'b0, raw}) - $signed(BIAS);
            //그냥 $signed(raw)해버리면 MSB 1인 경우에 대해 음수로 오해석될 위험이 있다.
            //따라서 MSB에 추가로 0을 붙여서 강제로 양수로 확장시킨다.

      - 이후, Output을 packed array로 출력함과 동시에 ReLU activation function 원리를 적용하여 assign 구문을 적용시켜준다.

            assign o_mat[(r*COLS + c)*ACC_W +: ACC_W] =
                (tmp < 0) ? {ACC_W{1'b0}} : tmp[ACC_W-1:0];

    - 5️⃣ 5) DUT Instantiation

            //==========================================================
            //5. systolic_array_2d.v Instantiation
            //==========================================================   
             systolic_array_2d #(
               .DATA_W (DATA_W),
               .ACC_W  (ACC_W),
               .ROWS   (ROWS),
               .COLS   (COLS)
             ) u_core_array (
               .clk        (clk),
               .rst_n      (rst_n),
               .clr        (array_clr),
               .en         (array_en),
               .a_in_row   (a_in_row),
               .b_in_col   (b_in_col),
               .pe_mul     (),
               .pe_acc_sum (pe_acc_sum) 
             );
        endmodule

      - DUT 요약
         1) 입력 행렬 A,B를 packed array(i_mat_a, i_mat_b)로 받음
         2) 시작 신호(i_start)에 맞춰 A,B를 내부 버퍼에 래치(latched_mat_a/b)
         3) 카운터(cnt) 기반으로 A/B를 스큐 주입(a_in_row, b_in_col)
         4) systolic_array_2d에 en/clr 제어와 함께 입력 주입
         5) systolic_array_2d 출력 pe_acc_sum에 대해 Bias + ReLU를 적용하여 o_mat(packed array) 생성

      - Moore Machine 기반 FSM 상태 전이 요약
        
        |State|입력/조건|Next-State|cnt 동작|출력|
        | :---: | :---: | :---:| :---:| :---:|
        |**IDLE**|**i_start=0**|**IDLE**|**0 유지**|**busy=0, done=0**|
        |**IDLE**|**i_start=1**|**RUN**|**다음 사이클에 증가**|**busy=0→1**|
        |**RUN**|**cnt < INJ_CYCLES**|**RUN**|**cnt++**|**busy=1**|
        |**RUN**|**cnt > INJ_CYCLES**|**DONE**|**다음 사이클 cnt=0**|**busy=0, done=1**|
        |**DONE**|**i_start=1**|**DONE**|**0 유지**|**done=1 유지**|
        |**DONE**|**i_start=0**|**IDLE**|**0 유지**|**done=0**|

      - 전체 블록 다이어그램 요약
        - i_mat_a → <Packed→Unpacked> Mapping → latched_mat_a → Skew → a_in_row → SA 입력
        - i_mat_b → <Packed→Unpacked> Mapping → latched_mat_b → Skew → b_in_col → SA 입력
        - state/cnt → Skew Timing 결정
        - state → array_en, array_clr, o_busy, o_done
        - SA 출력 pe_acc_sum → PPU(ReLU+Bias) → o_ma

      - 전체 블록 다이어그램 요약
<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_36.png" width="400"/>

<div align="left">

- Testbench

        module tb_systolic_controller_relu;
        
          localparam int K_DIM  = 4;
          localparam int DATA_W = 8;
          localparam int ACC_W  = 2*DATA_W + $clog2(K_DIM);
          localparam int ROWS   = 4;
          localparam int COLS   = 4;
          localparam int BIAS   = 50;
        
          //==========================================================
          // Step 1) Interface Definition
          //==========================================================
          logic clk;
          logic rst_n;
        
          logic i_start;
          logic o_busy;
          logic o_done;
        
          logic [ROWS*K_DIM*DATA_W-1:0] i_mat_a;   //packed array dut bus
          logic [COLS*K_DIM*DATA_W-1:0] i_mat_b;   //packed array dut bus
          logic [ROWS*COLS*ACC_W-1:0]   o_mat;     //packed array dut bus
        
          logic [DATA_W-1:0] tb_mat_a [0:ROWS-1][0:K_DIM-1];  //inner tb unpacked array
          logic [DATA_W-1:0] tb_mat_b [0:K_DIM-1][0:COLS-1];  //inner tb unpacked array
          logic [ACC_W-1:0]  tb_mat_c [0:ROWS-1][0:COLS-1];   //inner tb unpacked array

    - 1️⃣ Parameter + Interface Definition

          //==========================================================
          // Step 2) Constrained Random Transaction
          //==========================================================
          class sa_ctrl_txn;
            rand bit [DATA_W-1:0] A [0:ROWS-1][0:K_DIM-1];
            rand bit [DATA_W-1:0] B [0:K_DIM-1][0:COLS-1];
            rand bit do_start;
            constraint c_start { do_start dist {1 := 70, 0 := 30}; }
          endclass
        
          sa_ctrl_txn sa = new();

    - 2️⃣ Constrained Random Transaction
      - 기존에 정의한 class와 큰 차이점은 없지만 이번 transaction에서는 두 입력과 start를 걸지 말지 rand bit do_start를 dist 구문을 활용한 확률 분포 제약을 건 세 가지 케이스를 random transaction한다.

              //==========================================================
              // Step 3) DUT Instantiation
              //==========================================================
              systolic_controller_relu #(
                ...
              ) dut (
                .clk     (clk),
                .rst_n   (rst_n),
                .i_start (i_start),
                .i_mat_a (i_mat_a),
                .i_mat_b (i_mat_b),
                .o_busy  (o_busy),
                .o_done  (o_done),
                .o_mat   (o_mat)
              );
            
              //==========================================================
              // Clock / Reset
              //==========================================================
              initial begin
                clk = 1'b0;
                forever #5 clk = ~clk;
              end
            
              initial begin
                rst_n = 1'b0;
                i_start = 1'b0;
            
                i_mat_a = '0;
                i_mat_b = '0;
                tb_mat_a = '{default:'0}; //inner tb 2D array → 0 reset
                tb_mat_b = '{default:'0}; //inner tb 2D array → 0 reset
                tb_mat_c = '{default:'0}; //inner tb 2D array → 0 reset
            
                #30;
                rst_n = 1'b1;
              end  
            
              //==========================================================
              // Golden Model (Bias + ReLU)
              //==========================================================
              int unsigned      C_ref_int  [0:ROWS-1][0:COLS-1];
              //golden model output을 최초로 int로 저장하는 2D array
              //Bias subtraction 및 ReLU와 같은 PPU 진행과정에서 overflow 관리 목적
              logic [ACC_W-1:0] C_ref [0:ROWS-1][0:COLS-1];
              //golden model output인 C_ref_int를 dut 출력과 동일한 ACC_W 폭으로 저장
            
              task automatic calc_golden_relu();
                for (int r=0; r<ROWS; r++) begin
                  for (int c=0; c<COLS; c++) begin
                    int tmp = 0; //(r,c) 하나의 누산 결과를 int로 계산할 임시 변수
                    logic [ACC_W-1:0] tmp_c_ref;  
            
                    for (int k=0; k<K_DIM; k++) begin
                      tmp += (sa.A[r][k] * sa.B[k][c]);
                    end
                    /*Bias*/ tmp -= BIAS; //누산 이후, Bias Subtraction
                    /*ReLU*/ if (tmp < 0) tmp = 0; //ReLU Activation Function
            
                    C_ref_int[r][c] = tmp;
                    tmp_c_ref = tmp;              
                    C_ref[r][c] = tmp_c_ref;     
                  end
                end
              endtask

    - 3️⃣ DUT Instantiation + Clock/Reset 생성 + Golden Model & Checker
      - Golden Model은 task automatic으로 calc_golden_relu 함수를 생성하여 Bias + ReLu의 PPU가 모두 적용된 최종적인 C_ref의 unpacked array를 생성시킨다.

              covergroup cg_ctrl;
                cp_do_start : coverpoint sa.do_start { bins b0={0}; bins b1={1}; }
            
                cp_A00 : coverpoint sa.A[0][0] {
                  bins zero={0}; 
                  bins one={1}; 
                  bins max={(2**DATA_W)-1};
                  bins mid={[2:(2**DATA_W)-2]};
                }
            
                cp_B00 : coverpoint sa.B[0][0] {
                  bins zero={0}; 
                  bins one={1}; 
                  bins max={(2**DATA_W)-1};
                  bins mid={[2:(2**DATA_W)-2]};
                }
              endgroup
              cg_ctrl cg = new();

    - 4️⃣ Functional Coverage

        //==========================================================
        // Packed ↔ Unpacked Array Mapping tasks
        //==========================================================
          task automatic pack_inputs();
            i_mat_a = '0;
            i_mat_b = '0;
        
            for (int i=0; i<ROWS; i++) begin
              for (int k=0; k<K_DIM; k++) begin
                i_mat_a[(i*K_DIM + k)*DATA_W +: DATA_W] = tb_mat_a[i][k];
              end
            end
        
            for (int m=0; m<K_DIM; m++) begin
              for (int j=0; j<COLS; j++) begin
                i_mat_b[(m*COLS + j)*DATA_W +: DATA_W] = tb_mat_b[m][j];
              end
            end
          endtask
        
          task automatic unpack_outputs();
            for (int r=0; r<ROWS; r++) begin
              for (int c=0; c<COLS; c++) begin
                tb_mat_c[r][c] = o_mat[(r*COLS + c)*ACC_W +: ACC_W];
              end
            end
          endtask
        //==========================================================
        // Driver tasks
        //==========================================================
          task automatic drive_inputs_from_txn();
            for (int r=0; r<ROWS; r++)
              for (int k=0; k<K_DIM; k++)
                tb_mat_a[r][k] = sa.A[r][k];
        
            for (int k=0; k<K_DIM; k++)
              for (int c=0; c<COLS; c++)
                tb_mat_b[k][c] = sa.B[k][c];
        
            pack_inputs();
          endtask
          //i_start를 1로 계속 유지하면 IDLE 상태에서 즉시 RUN으로 재시작할 수 있는 
          //원치않는 trigger가 발생할 위험이 있다.
          //따라서 1 pulse만 i_start를 넣어주는 함수를 정의한다.
          task automatic start_pulse_1cycle();
            @(posedge clk);
            i_start = 1'b1;
            @(posedge clk);
            i_start = 1'b0;
          endtask

    - 4️⃣ Main 진입 전 driver 정의 
      - unpacked ↔ packed mapping 함수 2개
      - constraint random transaction에서 생성된 입력을 tb input과 연결하는 함수 start를 1 cycle pulse 주는 각종 driver utility들을 정의

              //==========================================================
              // Step 5) main
              //==========================================================
              localparam int  NUM_TESTS    = 1000; //최대 testcase 수
              localparam int  FLUSH_CYCLES = 5;
              //Skew 주입 종료 후, 내부 pipe에 남아있는 데이터가 끝까지 전달되고
              //acc_sum이 안정화되도록 0을 더 넣는 구간
              //경험적으로 5정도로 설정
              localparam real COV_TARGET   = 100.0;
              //coverage 목표치, 100이 되면 조기 종료
            
              int err_cnt = 0;
              //testcase 1개에서 발생한 에러 수
              int err_cnt_total = 0;
              //전체 누적된 에러 수
             
              initial begin
                @(posedge rst_n);
                @(posedge clk);
            
                $display("==================================================");
                $display("[TB] systolic_controller_relu Random Verification START");
                $display("  ROWS=%0d, COLS=%0d, K_DIM=%0d, DATA_W=%0d, ACC_W=%0d, BIAS=%0d",
                         ROWS, COLS, K_DIM, DATA_W, ACC_W, BIAS);
                $display("  NUM_TESTS=%0d", NUM_TESTS);
                $display("==================================================");
            
                for (int tc=0; tc<NUM_TESTS; tc++) begin
                  err_cnt = 0;
                  
                  assert(sa.randomize());
                  cg.sample();
                  
                  //sa.A/B → tb_mat_a/b 복사 → pack_inputs()로 i_mat_a/b 세팅
                  drive_inputs_from_txn();
                  //golden model calculation
                  calc_golden_relu();
            
                  if (sa.do_start) begin
                    //1. DUT가 아직 이전 작업 때문에 busy면, idle이 될 때까지 기다렸다가 다음 시작.
                    if (o_busy) wait(o_busy == 0);
                    //2. i_start 1 pulse injection
                    start_pulse_1cycle();
                    //3. DUT가 done을 1로 올릴 때까지 연산 결과 대기.
                    wait(o_done == 1);
                    //4. Flush : 내부 파이프, 출력 레지스터 갱신, 안정화 시간을 주는 역할.
                    repeat (FLUSH_CYCLES) @(posedge clk);
                    //5. output : o_mat(flat packed)를 tb_mat_c[r][c](2D)로 풀어 담음.
                    unpack_outputs();
            
                    $display("--------------------------------------------------");
                    $display("[TB] TestCase %0d (do_start=1) Checking...", tc);
            
                    for (int r=0; r<ROWS; r++) begin
                      for (int c=0; c<COLS; c++) begin
                        if (tb_mat_c[r][c] !== C_ref[r][c]) begin
                          err_cnt++;
                          $display("ERROR C[%0d][%0d]: DUT=%0d | REF=%0d | time=%0t",
                                   r, c, tb_mat_c[r][c], C_ref[r][c], $time);
                        end
                        else begin
                          if (tb_mat_c[r][c] == '0)
                            $display("PASS  C[%0d][%0d]: DUT=%0d | REF=%0d | (ReLU) | time=%0t",
                                     r, c, tb_mat_c[r][c], C_ref[r][c], $time);
                          else
                            $display("PASS  C[%0d][%0d]: DUT=%0d | REF=%0d | time=%0t",
                                     r, c, tb_mat_c[r][c], C_ref[r][c], $time);
                        end
                      end
                    end
            
                    if (err_cnt == 0) $display("[TB] RESULT: PASS");
                    else              $display("[TB] RESULT: FAIL (err=%0d)", err_cnt);
            
                    err_cnt_total += err_cnt;
            
                  end else begin
                    $display("[TB] TestCase %0d (do_start=0) -> skip checking", tc);
                  end
            
                  if (cg.get_inst_coverage() >= COV_TARGET) begin
                    $display("[TB] Coverage reached %0.2f%% at testcase=%0d, time=%0t",
                             cg.get_inst_coverage(), tc, $time);
                    break;
                  end
                end
            
                $display("==================================================");
                $display("[TB] Simulation Summary");
                $display("  Total errors   : %0d", err_cnt_total);
                if (err_cnt_total == 0) $display("  RESULT         : PASS");
                else                    $display("  RESULT         : FAIL");
                $display("--------------------------------------------------");
                $display("[TB] Functional Coverage");
                $display("  TOTAL      : %0.2f %%", cg.get_inst_coverage());
                $display("  do_start   : %0.2f %%", cg.cp_do_start.get_inst_coverage());
                $display("  A[0][0]    : %0.2f %%", cg.cp_A00.get_inst_coverage());
                $display("  B[0][0]    : %0.2f %%", cg.cp_B00.get_inst_coverage());
                $display("==================================================");
                $finish;
              end
            endmodule

    - 5️⃣ Main 
      - unpacked ↔ packed mapping 함수 2개

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_37.png" width="400"/>

log 값

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_38.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_39.png" width="400"/>

Timing Diagram Full View

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_40.png" width="400"/>

<div align="left">
