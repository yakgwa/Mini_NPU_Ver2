## [AI Accelerator 기초] - Systolic Array와 FSM 구현2 (2D 4X4 Systolic Array) + Debugging Point

### 2D (1-dimension) 4X4 Systolic Array (Output Stationary)

(이전 글에서 제시한 mac_pe dut를 그대로 활용)

- DUT

        module pe_systolic_cell #(
            ...
        )(
            ... 
            );
            
            //a_in, b_in -> register 한 번 잡고 전달
            reg [DATA_W-1:0] a_reg;
            reg [DATA_W-1:0] b_reg;
            
            assign a_out = a_reg;
            assign b_out = b_reg;
            
            always @(posedge clk or negedge rst_n) begin
                if (!rst_n) begin
                    a_reg <= 0;
                    b_reg <= 0;
                end else if (en) begin
                    a_reg <= a_in;
                    b_reg <= b_in;
                end
            end
            
            // 내부 MAC
            mac_pe #(
                ...
            ) dut (
                ...
                .a       (a_reg),
                .b       (b_reg),
                ...
          );
        endmodule

    - pe_systolic_cell은 기존에 최초 제시된 mac_pe와는 다르게, 연산기(mac_pe) + 입력 전달 목적 pipe register + next cell interconnect까지 포함된 Systolic Array용 PE cell이라고 요약할 수 있다. 
 
            always @(posedge clk or negedge rst_n) begin
                if (!rst_n) begin
                    a_reg <= 0;
                    b_reg <= 0;
                end else if (en) begin
                    a_reg <= a_in;
                    b_reg <= b_in;
                end
            end

    - 특히, 위 입력 레지스터 업데이트의 의미를 정확히 분석하면 다음과 같다.
      - (1) Reset : rst_n=0이면, a_reg/b_reg <= 0, 셀 전체 스트림이 0으로 초기화
      - (2) enable : en==1일 때만 a_in/b_in <= a_in/b_in, en==0이면 값 유지(hold), 결과적으로 셀은 en에 의해 “데이터 흐름을 멈추거나 흘릴 수 있는” 게이트 역할

- DUT

        module systolic_array_2d #(
            parameter integer DATA_W = 8,
            parameter integer ACC_W  = 2*DATA_W,
            parameter integer ROWS   = 4,
            parameter integer COLS   = 4
        )(
            input  wire [ROWS*DATA_W-1:0]        a_in_row,
            input  wire [COLS*DATA_W-1:0]        b_in_col,
        
            input  wire                          clk,
            input  wire                          rst_n,
            input  wire                          clr,
            input  wire                          en,
        
            output wire [ROWS*COLS*ACC_W-1:0]    pe_mul,
            output wire [ROWS*COLS*ACC_W-1:0]    pe_acc_sum
        /*
        - 4x4=16개 PE의 `mul`과 `acc_sum`을 flat bus로 밖에 내보냄.
        - 각 PE 결과 폭이 `ACC_W`니까 총 `ROWS*COLS*ACC_W` 비트
        */
        );
            //Unpacked Array for Debug
            wire [DATA_W-1:0] a_data         [0:ROWS-1];               // a_in_row -> row별 분리
            wire [DATA_W-1:0] b_data         [0:COLS-1];               // b_in_col -> col별 분리
            wire [ACC_W-1:0]  mul            [0:ROWS-1][0:COLS-1];     // Each PE MUL Debug
            wire [ACC_W-1:0]  acc            [0:ROWS-1][0:COLS-1];     // Each ACC Debug
        
            //Systolic Array Net
            wire [DATA_W-1:0] a_conn [0:ROWS-1][0:COLS];               //a가 좌→우로 흐르는 경로
            wire [DATA_W-1:0] b_conn [0:ROWS][0:COLS-1];               //b가 상→하로 흐르는 경로
            
            genvar r, c;
            
            //<1D Vector(a,b_in_row) → 2D Vector(a,b_data)>
            // +: DATA_W : Part-select(starting index + width)
            generate
              for (r=0; r<ROWS; r=r+1) begin : GEN_SPLIT_A
                assign a_data[r] = a_in_row[r*DATA_W +: DATA_W];
              end
              for (c=0; c<COLS; c=c+1) begin : GEN_SPLIT_B
                assign b_data[c] = b_in_col[c*DATA_W +: DATA_W];
              end
            endgenerate
            
            //<Systolic Array First Node Connect>
            // A-data flow:
            //   a_data[r]      → a_conn[r][0]
            //   a_conn[r][c]   → PE[r][c].a_in
            //   PE[r][c].a_out → a_conn[r][c+1]
            //   → A 데이터는 row 방향으로 좌 → 우 이동
            //
            // B-data flow:
            //   b_data[c]      → b_conn[0][c]
            //   b_conn[r][c]   → PE[r][c].b_in
            //   PE[r][c].b_out → b_conn[r+1][c]
            //   → B 데이터는 column 방향으로 상 → 하 이동
            //
            // Each PE performs:
            //   mul = a_in * b_in
            //   acc_sum += mul
            //==========================================================
            generate
              for (r=0; r<ROWS; r=r+1) begin : GEN_A_EDGE
                assign a_conn[r][0] = a_data[r];
              end
              for (c=0; c<COLS; c=c+1) begin : GEN_B_EDGE
                assign b_conn[0][c] = b_data[c];
              end
            endgenerate
        
            //<PE Array Instantiation>    
            generate
              for (r=0; r<ROWS; r=r+1) begin : GEN_ROW
                for (c=0; c<COLS; c=c+1) begin : GEN_COL
                  assign pe_mul[(r*COLS + c)*ACC_W +: ACC_W]     = mul[r][c];
                  assign pe_acc_sum[(r*COLS + c)*ACC_W +: ACC_W] = acc[r][c];
                  
                  pe_systolic_cell #(
                    .DATA_W(DATA_W),
                    .ACC_W (ACC_W)
                  ) u_cell (
                    .a_in    (a_conn[r][c]),
                    .b_in    (b_conn[r][c]),
                    .a_out   (a_conn[r][c+1]),
                    .b_out   (b_conn[r+1][c]),
        
                    .clk     (clk),
                    .rst_n   (rst_n),
                    .clr     (clr),
                    .en      (en),
        
                    .mul     (mul[r][c]),
                    .acc_sum (acc[r][c])
                  );
                end
              end
            endgenerate
        endmodule

    - 우선 최초 외부에서 들어오는 입력을 다음과 같이 flat bus로 받는다.

          input  wire [ROWS*DATA_W-1:0] a_in_row,
          input  wire [COLS*DATA_W-1:0] b_in_col,

    - Verilog 문법 상 module 내부에서는 unpacked array로 선언할 수 없기 때문에, a_in_row는 "각 row마다 DATA_W짜리 하나씩", 총 `ROWS*DATA_W` 비트로 선언한다.
    - 예컨대, ROWS=4, DATA_W=8이면 32bit가 된다. 마찬가지로 b_in_col도 "각 col마다 DATA_W짜리 하나씩", 총 `COLS*DATA_W` 비트로 선언한다.
     - 이러한 packed array는 a_data, b_data와 같이 unpacked array로 선언된 wire를 generate 구문으로 다시 맵핑하여 가독성을 높일 수 있다.

            wire [DATA_W-1:0] a_data         [0:ROWS-1];               // a_in_row -> row별 분리
            wire [DATA_W-1:0] b_data         [0:COLS-1];               // b_in_col -> col별 분
            .....
            //<1D Vector(a,b_in_row) → 2D Vector(a,b_data)>
            // +: DATA_W : Part-select(starting index + width)
            generate
              for (r=0; r<ROWS; r=r+1) begin : GEN_SPLIT_A
                assign a_data[r] = a_in_row[r*DATA_W +: DATA_W];
              end
              for (c=0; c<COLS; c=c+1) begin : GEN_SPLIT_B
                assign b_data[c] = b_in_col[c*DATA_W +: DATA_W];
              end
            endgenerate

     - 여기서 Systolic Array의 각 PE를 interconncect하기 위한 연결망을 위에서 선언한 a_data와 맵핑한다. 여기서 핵심은 a_data는 좌→우로 이동하므로 COLS+1, b_data는 상→하로 이동하기 때문에 ROWS+1로 물리적인 파이프 역할을 하는 신호선을 고려한다.

            generate
              for (r=0; r<ROWS; r=r+1) begin : GEN_A_EDGE
                assign a_conn[r][0] = a_data[r];
              end
              for (c=0; c<COLS; c=c+1) begin : GEN_B_EDGE
                assign b_conn[0][c] = b_data[c];
              end
            endgenerate

     - 이렇게 맵핑된 Systolic Array는 다음과 같이, 요약할 수 있다.

            // 2D Systolic Dataflow (A: left→right, B: top→bottom)
            //                          b_data[0] b_data[1]b_data[2]b_data[3]
            //                               ↓      ↓     ↓      ↓
            //                     b_conn[0][0] b_conn[0][1] b_conn[0][2] b_conn[0][3]
            //                               ↓     ↓      ↓      ↓
            // a_data[0] → a_conn[0][0] →  PE  →  PE  →  PE  →  PE
            //                               ↓     ↓      ↓      ↓
            // a_data[1] → a_conn[1][0] →  PE  →  PE  →  PE  →  PE
            //                               ↓     ↓      ↓      ↓
            // a_data[2] → a_conn[2][0] →  PE  →  PE  →  PE  →  PE
            //                               ↓     ↓      ↓      ↓
            // a_data[3] → a_conn[3][0] →  PE  →  PE  →  PE  →  PE

- Testbench

        `timescale 1ns / 1ps
        
        module tb_systolic_array_2d;
        
          localparam int K_DIM  = 4;
          localparam int DATA_W = 8;
          localparam int ACC_W  = 2*DATA_W + $clog2(K_DIM);
          localparam int ROWS   = 4;
          localparam int COLS   = 4;
        
          //==========================================================
          // Step 1) Interface Definition
          //==========================================================
          logic                    clk;
          logic                    rst_n;
          logic                    clr;
          logic                    en;
        
          logic [DATA_W-1:0]       a_in_row   [0:ROWS-1];
          logic [DATA_W-1:0]       b_in_col   [0:COLS-1];
        
          logic [ROWS*DATA_W-1:0]  a_in_row_flat;
          logic [COLS*DATA_W-1:0]  b_in_col_flat;
        
          logic [ROWS*COLS*ACC_W-1:0] pe_mul;
          logic [ROWS*COLS*ACC_W-1:0] pe_acc_sum;

     - Step 1) Parameter + Interface Definition
       - DUT를 4X4 Array로 잡았기 때문에, PE 내부에서는 총 4번의 누산(K_DIM)이 반복된다. 이에 따라 ACC_W 역시 단순히 2*DATA_W가 아닌, 그 반복되는 누산으로 커지는 비트폭을 고려해야 하므로, $clog2(K_DIM)을 추가적으로 더해준다.
       - Interface 정의에서의 가장 중요한 포인트는 가독성 및 Skew 주입 시 Indexing이 편한 Unpacked Array(a_in_row, b_in_col)과 DUT는 flat bus로 받아 TB에서 packing이 필요하므로 Packed Array(a_in_row_flat, b_in_col_flat, pe_mul, pe_acc_sum)을 동시에 선언한다. 

                  //==========================================================
                  // Step 2) Constrained Random Transaction
                  //==========================================================
                  class sa_2d_txn;
                    rand bit [DATA_W-1:0] A [0:ROWS-1][0:K_DIM-1];
                    rand bit [DATA_W-1:0] B [0:K_DIM-1][0:COLS-1];
                
                    rand bit              en;
                    rand bit              clr;
                
                    constraint c_clr { clr dist {1 := 5, 0 := 95}; }
                    constraint c_en  { en  dist {1 := 50, 0 := 50}; }
                  endclass
                
                  sa_2d_txn sa = new;
                
                //==========================================================
                // Pack TB unpacked inputs -> DUT flat ports
                //==========================================================
                  assign begin
                    a_in_row_flat = '0;
                    b_in_col_flat = '0;
                    for (int r=0; r<ROWS; r++) a_in_row_flat[r*DATA_W +: DATA_W] = a_in_row[r];
                    for (int c=0; c<COLS; c++) b_in_col_flat[c*DATA_W +: DATA_W] = b_in_col[c];
                  end

     - Step 2) Constrained Random Transaction
       - Golden Model 계산에 사용할 행렬 A(ROWS×K), B(K×COLS)를 랜덤 생성한다. 여기서 A,B는 한 testcase 동안 고정된 행렬이라고 보면 된다. (en, clr은 dist를 활용하여 가중치 랜덤 선언)
       - (이와는 별개로 그 아래에 TB unpacked array(a_in_row, b_in_col) 값을 DUT 포트 형식으로 재구성하기 위해 DUT packed array(a_in_row_flat, b_in_col_flat)을 맵핑해준다.)

                  //==========================================================
                  // Step 3) Top Module Instantiation
                  //==========================================================
                  systolic_array_2d #(
                    ...
                  ) dut (
                    ...
                    .a_in_row   (a_in_row_flat), //packed array로 맵핑
                    .b_in_col   (b_in_col_flat), //packed array로 맵핑
                    ...
                  );
                
                  //==========================================================
                  // Clock / Reset
                  //==========================================================
                  initial begin
                    clk = 1'b0;
                    forever #5 clk = ~clk; // 100MHz
                  end
                
                  initial begin
                    rst_n = 1'b0;
                    clr  = 1'b0;
                    en   = 1'b0;
                
                    for (int r=0; r<ROWS; r++) a_in_row[r] = '0;
                    for (int c=0; c<COLS; c++) b_in_col[c] = '0;
                
                    #30;
                    rst_n = 1'b1;
                    clr = 1'b1; //전역 초기화
                    #10;
                    clr = 1'b0;
                  end
                
                //==========================================================
                // Golden Model (C_ref) 
                //==========================================================
                  int unsigned C_ref [0:ROWS-1][0:COLS-1];
                
                  //automatic task란? 
                  //호출될 때마다 task 내부 변수들이 각 호출마다 task 내부 변수들이
                  //각 호출마다 독립적으로 생성되어 지역변수가 공유되지 않는다는 장점을 가지고 있다.
                
                  task automatic calc_golden_C_ref();
                    for (int r=0; r<ROWS; r++) begin
                      for (int c=0; c<COLS; c++) begin
                        int unsigned acc = 0;
                        for (int k=0; k<K_DIM; k++) begin
                          acc += (sa.A[r][k] * sa.B[k][c]);
                        end
                        C_ref[r][c] = acc;
                        end
                      end
                  endtask

     - Step 3) DUT Instantiation + Clock/Reset 생성 + Golden Model & Checker
       - Golden Model 정의에 대해서 주로 살펴보도록 하자. 기존에 mac.pe.v, pe_chain_1d.v에서는 golden model을 dut 그대로 작성하였지만, 2D Array에서는 task automatic으로 선언하여 golden model을 간소화 시켰다.
       - 우선, Golden Model의 결과를 저장하는 저장 배열 C_ref를 전역 선언한다.

                int unsigned C_ref [0:ROWS-1][0:COLS-1];

     - 이후, C_ref = A X B 행렬곱을 계산하기 위해, acc_local를 (r,c)마다 0으로 초기화하여, constraint random으로 생성한 sa.A, sa.B를 acc_local에 담아서 정답인 C_ref를 계산하도록 한다.
 

                  //==========================================================
                  // Step 4) Functional Coverage
                  //==========================================================
                  covergroup cg_sa_2d;
                    cp_en  : coverpoint sa.en  { bins b0={0}; bins b1={1}; }
                    cp_clr : coverpoint sa.clr { bins b0={0}; bins b1={1}; }
                
                    cp_A00 : coverpoint sa.A[0][0] {
                      bins zero = {0};
                      bins one  = {1};
                      bins max  = {(2**DATA_W)-1};
                      bins mid = {[2:(2**DATA_W)-2]};
                    }
                
                    cp_B00 : coverpoint sa.B[0][0] {
                      bins zero = {0};
                      bins one  = {1};
                      bins max  = {(2**DATA_W)-1};
                      bins mid = {[2:(2**DATA_W)-2]};
                    }
                  endgroup
                
                  cg_sa_2d cg = new;

     - Step 4) Functional Coverage
       - (기존 글에서 작성한 mac_pe.v, pe_chain_1d.v에 대한 검증 코드에서 정의한 coverage class와 동일) 
       - (최초 주입 입력 A[0][0], B[0][0]를 4개의 구간으로 cover(0, 1, max(255), mid(2~254))

                  //==========================================================
                  // Step 5) main
                  //==========================================================
                  
                //==========================================================
                // Main 돌입 직전, DUT에 입력을 어떻게 넣을지 정의하는 Driver
                //==========================================================
                
                  //Data Skew : a_in_row, b_in_col을 한 사이클 분량으로 세팅하는 입력 스케줄러
                  task automatic drive_skew_cycle(input int t);
                    for (int r=0; r<ROWS; r++) begin
                      int kA = t - r;
                      if ((kA >= 0) && (kA < K_DIM)) a_in_row[r] = sa.A[r][kA];
                      else                           a_in_row[r] = '0;
                    end
                
                    for (int c=0; c<COLS; c++) begin
                      int kB = t - c;
                      if ((kB >= 0) && (kB < K_DIM)) b_in_col[c] = sa.B[kB][c];
                      else                           b_in_col[c] = '0;
                    end
                  endtask
                
                  // testcase 경계에서 acc_sum 초기화용 clr 1-cycle 펄스
                  task automatic clr_pulse_1cycle();
                    @(negedge clk);
                    clr = 1'b1;
                    for (int r=0; r<ROWS; r++) a_in_row[r] = '0;
                    for (int c=0; c<COLS; c++) b_in_col[c] = '0;
                    @(posedge clk);
                    @(negedge clk);
                    clr = 1'b0;
                  endtask  

     - Step 5) Main (Driver)
       - drive_skew_cycle : Data Skew, a_in_row, b_in_col을 한 사이클 분량으로 세팅하는 입력 스케줄러
        
                    for (int r=0; r<ROWS; r++) begin
                      int kA = t - r;
                      if ((kA >= 0) && (kA < K_DIM)) a_in_row[r] = sa.A[r][kA];
                      //kA가 유효한 인덱스(0~K_DIM-1) 안에 들어오면 실제 A 값 주입하고, 아니면 0 주입
                      //이때, testcase에서 랜덤으로 생성된 행렬 A의 원소를 시간에 맞춰 보내게 됨
                      else                           a_in_row[r] = '0;
                    end
                //b_in_col도 마찬가지

       - 같은 시간 t에 대해 row(ROWS)가 커질수록 입력이 늦게 들어가도록 한다.
       - row 0: kA = t
       - row 1: kA = t-1 → 한 사이클 늦은 효과
       - row 2: kA = t-2 → 두 사이클 늦은 효과

     - 예시 : 랜덤으로 생성된 행렬이 다음과 같다고 하자.
       - A (ROWS×K_DIM)
         - row0: [1, 2, 3, 4]
         - row1: [5, 6, 7, 8]
         - row2: [9,10,11,12]
         - row3: [13,14,15,16]
         - sa.A[0][0]=1, sa.A[0][1]=2, … sa.A[3][3]=16

       - B (K_DIM×COLS)
         - k0: [1, 0, 0, 0]
         - k1: [0, 1, 0, 0]
         - k2: [0, 0, 1, 0]

       - 정답 : C = A×I = A

       - 1️⃣ t=0일 때
         - A 입력
           - r=0: kA=0-0=0 → a_in_row[0]=A[0][0]=1
           - r=1: kA=0-1=-1 → 범위 밖 → a_in_row[1]=0
           - r=2: kA=-2 → 0
           - r=3: kA=-3 → 0
           - ➡️ a_in_row = [1, 0, 0, 0]
             
         - B 입력
           - c=0: kB=0-0=0 → b_in_col[0]=B[0][0]=1
           - c=1: kB=-1 → 0
           - c=2: 0
           - c=3: 0
           - ➡️ b_in_col = [1, 0, 0, 0]

       - 2️⃣ t=1일 때
         - A 입력
           - r=0: kA=1 → A[0][1]=2
           - r=1: kA=0 → A[1][0]=5
           - r=2: kA=-1 → 0
           - r=3: kA=-2 → 0
           - ➡️ a_in_row = [2, 5, 0, 0]

         - B 입력
           - c=0: kB=1 → B[1][0]=0
           - c=1: kB=0 → B[0][1]=0
           - c=2: -1 → 0
           - c=3: -2 → 0
           - ➡️ b_in_col = [0, 0, 0, 0]

       - 3️⃣ t=2일 때
         - A 입력
           - r=0: kA=2 → A[0][2]=3
           - r=1: kA=1 → A[1][1]=6
           - r=2: kA=0 → A[2][0]=9
           - r=3: kA=-1 → 0
           - ➡️ a_in_row = [3, 6, 9, 0]

         - B 입력
           - c=0: kB=2 → B[2][0]=0
           - c=1: kB=1 → B[1][1]=1 
           - c=2: kB=0 → B[0][2]=0
           - c=3: -1 → 0
           - ➡️ b_in_col = [0, 1, 0, 0]

       - 4️⃣ t=3일 때
         - A 입력
           - r=0: kA=3 → A[0][3]=4
           - r=1: kA=2 → A[1][2]=7
           - r=2: kA=1 → A[2][1]=10
           - r=3: kA=0 → A[3][0]=13
           - ➡️ a_in_row = [4, 7, 10, 13]

         - B 입력
           - c=0: kB=3 → B[3][0]=0
           - c=1: kB=2 → B[2][1]=0
           - c=2: kB=1 → B[1][2]=0
           - c=3: kB=0 → B[0][3]=0
           - ➡️ b_in_col = [0, 0, 0, 0]

       - clr_pulse_1cycle : testcase 경계에서 acc_sum 초기화용 clr 1-cycle 펄스
       - DUT의 clr을 정확히 1clk 동안만 1로 만들어 누산을 초기화하려는 목적의 함수이다.

                  task automatic clr_pulse_1cycle();
                    @(negedge clk);
                    clr = 1'b1;

       - negedge에서 clr=1로 하여 setup time을 확보하고, 그 동안 a_in_row, b_in_col을 모두 0으로 만든다. 즉, DUT의 레지스터들은 보통 posedge에서 샘플링하기 때문에, negedge에서 clr=1로 올려두면, 다음 posedge까지 반 주기가 생기고, 이를 통해 setup time 관점에서 더 안전하게 클록 edge 전에 안정화 시킬 수 있다.

                  localparam int  NUM_TESTS    = 1000; //최대 testcase 수
                  localparam int  INJ_CYCLES   = K_DIM + ROWS + COLS - 2; //skew 주입 횟수
                  //쉽게 말해, 행/열마다 PE에 최초로 주입되는 입력 값이 딜레이가 생기는 만큼,
                  //Systolic Array의 PE가 전부 계산될 때 까지 주입해야 하는 시간을 말한다.
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
                  int cycles_checked = 0;
                  //전체 시뮬레이션 동안의 cycle 카운트

     - Step 5) Main (parameter + 변수 설정)
       
                   initial begin
                    @(posedge rst_n);
                    @(posedge clk);
                
                    $display("==================================================");
                    $display("[TB] systolic_array_2d Random Verification START");
                    $display("  ROWS=%0d, COLS=%0d, K_DIM=%0d, DATA_W=%0d, ACC_W=%0d",
                             ROWS, COLS, K_DIM, DATA_W, ACC_W);
                    $display("  NUM_TESTS=%0d, INJ_CYCLES=%0d", NUM_TESTS, INJ_CYCLES);
                    $display("==================================================");
                
                    for (int tc = 0; tc < NUM_TESTS; tc++) begin
                      // (1) 랜덤 생성
                      assert(sa.randomize());
                      cg.sample(); 
                     
                      // (2) testcase 시작마다 DUT 누산 상태를 강제 초기화
                      // 위에서 정의한 clr_pulse_1cycle 함수 driver 활용한다. 
                      clr_pulse_1cycle();
                      cycles_checked++; // driver에서 사이클 소모로 대략적인 사이클 체크 값 보정치 적용
                     
                      // (3) Golden Model 정답 계산 
                      // 위에서 정의한 calc_golden_C_ref로 정의된 golden model 함수를 활용한다.
                      calc_golden_C_ref();  
                
                      // (4) enable 신호 적용
                      en  = sa.en;
                      cycles_checked++; 
                
                      // (5) Skew Injection
                      // 위에서 정의한 drive_skew_cycle 함수 driver 활용한다.
                      for (int t=0; t<INJ_CYCLES; t++) begin
                        @(negedge clk); //negedge에서 입력 세팅하고 posedge에서 dut가 sampling
                        if (en) begin   //en=1이면 스케줄링에 맞춰 해당 t의 row/col 입력을 넣는다
                            drive_skew_cycle(t);         
                        end else begin
                          for (int r=0; r<ROWS; r++) a_in_row[r] = '0;
                          for (int c=0; c<COLS; c++) b_in_col[c] = '0;
                        end
                        @(posedge clk); //posedge에서 dut가 입력을 받아 레지스터 및 누산 업데이트
                        cycles_checked++;
                      end
                
                      // (6) Flush
                      // 입력을 더 이상 넣지 않고 0만 주입한다.
                      // 내부 파이프라인에 남아있는 값들이 끝까지 흘러가며 결과가 완성되도록 사이클을 확보한다.
                      for (int f=0; f<FLUSH_CYCLES; f++) begin
                        @(negedge clk);
                        for (int r=0; r<ROWS; r++) a_in_row[r] = '0;
                        for (int c=0; c<COLS; c++) b_in_col[c] = '0;
                        @(posedge clk);
                        cycles_checked++;
                      end
                
                      // (7) Result Check
                      $display("--------------------------------------------------");
                      $display("[TB] TestCase %0d (en=%0d, clr=%0d) Checking...", tc, en, 0);
                
                      if (en) begin
                        for (int r = 0; r < ROWS; r++) begin
                          for (int c = 0; c < COLS; c++) begin
                            if (pe_acc_sum[(r*COLS + c)*ACC_W +: ACC_W] !== C_ref[r][c]) begin
                              err_cnt++;
                              $display("ERROR C[%0d][%0d]: dut=%0d, ref=%0d, time=%0t",
                                       r, c,
                                       pe_acc_sum[(r*COLS + c)*ACC_W +: ACC_W],
                                       C_ref[r][c],
                                       $time);
                            end
                            else begin
                              $display("PASS  C[%0d][%0d]: dut=%0d, ref=%0d, time=%0t",
                                       r, c,
                                       pe_acc_sum[(r*COLS + c)*ACC_W +: ACC_W],
                                       C_ref[r][c],
                                       $time);
                            end
                          end
                        end
                
                        if (err_cnt == 0) $display("[TB] RESULT: PASS");
                        else               $display("[TB] RESULT: FAIL (err=%0d)", err_cnt);
                
                        err_cnt_total += err_cnt;
                      end
                      else begin
                        $display("[TB] en=0 -> skip checking this testcase");
                      end
                
                      // (8) Coverage 목표 도달 시 조기 종료
                      if (cg.get_inst_coverage() >= COV_TARGET) begin
                        $display("[TB] Coverage reached %0.2f%% at testcase=%0d, time=%0t",
                                 cg.get_inst_coverage(), tc, $time);
                        break;
                      end
                    end
                
                    //========================================================
                    // Summary
                    //========================================================
                    $display("==================================================");
                    $display("[TB] Simulation Summary");
                    $display("  Cycles checked : %0d", cycles_checked);
                    $display("  Total errors   : %0d", err_cnt_total);
                    if (err_cnt_total == 0) $display("  RESULT         : PASS");
                    else                    $display("  RESULT         : FAIL");
                    $display("--------------------------------------------------");
                    $display("[TB] Functional Coverage");
                    $display("  TOTAL      : %0.2f %%", cg.get_inst_coverage());
                    $display("  en         : %0.2f %%", cg.cp_en.get_inst_coverage());
                    $display("  clr        : %0.2f %%", cg.cp_clr.get_inst_coverage());
                    $display("  A[0][0]    : %0.2f %%", cg.cp_A00.get_inst_coverage());
                    $display("  B[0][0]    : %0.2f %%", cg.cp_B00.get_inst_coverage());
                    $display("==================================================");
                    $finish;
                  end
                endmodule

     - Step 5) Main (메인 initial 시작)
       - (1) Randomize → (2) Testcase 시작마다 DUT 누산 상태를 강제 초기화 → (3) Golden Model 정답 계산 → (4) enable 신호 적용 → (5) Skew Injection → (6) Flush → (7) Result Check → (8) Coverage 목표 도달 시 조기 종료

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_30.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_31.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_32.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_33.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_34.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_35.png" width="400"/>

Testcase 0th 손계산 추가 검증

<div align="left">











