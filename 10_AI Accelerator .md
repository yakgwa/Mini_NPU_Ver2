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


















