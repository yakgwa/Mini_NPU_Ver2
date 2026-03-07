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









