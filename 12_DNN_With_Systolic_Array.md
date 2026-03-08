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

    - 기존 mac_pe.v와 비교한 개선 포인트
      1) signed : fixed-point를 고려한 SIGNED 연산
            - 기존 mac_pe.v는 입력과 연산 경로가 unsigned로 정의되어 있다. 하지만 실제 DNN 연산에서는 activation이나 weight가 음수를 포함하는 경우가 일반적이며, fixed-point 환경에서는 부호 해석이 잘못되면 곱셈 단계부터 결과가 왜곡될 수 있다. 따라서 signed로 정의함으로써 음수 입력 및 가중치가 포함된 경우에도 하드웨어 동작과 수치 해석이 직관적으로 유지되도록 한다.
      2) DEBUG 매크로를 이용한 디버그 포트 옵션화
            - PE.v에서는 DEBUG 매크로를 사용해 디버그 포트를 조건부로 노출한다.

                        `ifdef DEBUG ... `endif
        
            - 디버그 빌드에서는 곱 결과를 외부에서 관측할 수 있어 검증이 용이하고, 릴리즈 빌드에서는 해당 포트 자체가 제거되어 합성 및 배치 관점에서 더 깔끔한 구조를 유지할 수 있다.
      3) Saturation 로직 추가
            - PE.v에서는 오버플로우와 언더플로우 상황을 정확하게 처리하기 위해 포화Saturation 로직을 추가한다. 예컨대, 양수와 양수를 더했을 때 결과가 음수로 해석될 경우(즉, 부호 비트가 1로 바뀔 경우), 최대 양수 값(0x7FFF)으로, 음수와 음수를 더했을 때 결과가 양수로 해석될 경우(부호 비트가 0으로 바뀔 때), 최소 음수 값(0x8000)으로 saturation시킨다. 이러한 saturation 처리는 다음과 같은 방식으로 동작시킨다.
        
                        if (!sign_mul && !sign_sum && sign_res) sum <= 0x7FFF; 
                        // 양수 + 양수가 음수로 해석될 때 
                        if (sign_mul && sign_sum && !sign_res) sum <= 0x8000; 
                        // 음수 + 음수가 양수로 해석될 때
                        
                        //이 로직을 통해 더 이상 부호 비트의 오류로 연c산이 잘못되거나 왜곡되는 일이 없도록 보장

      4) 추후 PPA 개선 포인트 ①: 파이프라이닝(Pipelining)
            - 현재 PE 구조에서는 한 클럭 사이클 내에 덧셈 반영은 한 클럭 이후에 반영될 뿐, 곱셈과 덧셈이 연속으로 수행된다. 특히 곱셈기는 덧셈기에 비해 지연이 크고 면적이 큰 회로인데, 이어 덧셈기까지 직렬로 연결되면 Critical Path가 훨씬 더 길어질 수 밖에 없다. 기능 검증 목적에서는 문제가 되지 않지만, 이후 성능 목표에 따라 가장 먼저 개선 대상이 될 수 있다. 이에 따라 타이밍 최적화를 위한 가장 대표적인 방법은 곱셈과 덧셈 사이에 레지스터를 추가하여 pipeline을 구성하는 것이다.

                        현재 구조(단일 스테이지):
                        Reg → [Mul] → [Add] → Reg 지연 = T_mul + T_add
                        
                        파이프라인 적용 시(2-stage):
                        Stage 1: Reg → [Mul] → Pipe_Reg 
                        Stage 2: Pipe_Reg → [Add] → Reg 
                        "이에 따라 지연 시간 = max(T_mul, T_add)"

            - 이렇게 하면 클럭 주기가 단축되어 Fmax를 높일 수 있다. 다만, 데이터가 1사이클 늦게 전달되므로 i_en과 같은 제어 신호도 동일하게 1클럭 지연시켜야 하며, 전체 MAC 연산의 latency가 1 증가한다는 trade-off가 존재한다.
            - ​최종적으로 mac_pe.v용으로 작성된 tb_mac_pe.sv를 PE.v에 맞게 최소한의 수정만 적용하여 재사용하였으며, 입력 데이터(0, 최대 양수, 최대 음수, 일반 값)와 제어 신호(i_en, i_rst)의 모든 유효 조합에 대해 coverpoint를 구성하였다. 검증 결과 functional coverage 100%를 만족하였고, PE.v는 기능적으로 정상 동작함을 확인하였다.

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_41.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_42.png" width="400"/>

<div align="left">

### pe_systolic_cell.v / Systolic_Array.v

- DUT

        module pe_systolic_cell #(
            parameter integer dataWidth = 8
            )(
            //본인 연산용 데이터 (왼쪽/위쪽에서 옴)
            input  wire signed [dataWidth-1:0]    i_a,
            input  wire signed [dataWidth-1:0]    i_b,
            //옆/아래 PE로 전달할 데이터 (Registering -> Data Skew 용)
            output wire signed [dataWidth-1:0]    o_a,
            output wire signed [dataWidth-1:0]    o_b,
            //Control
            input  wire                           i_clk,
            input  wire                           i_rst,    
            input  wire                           i_en,
            //Result
            output wire signed [2*dataWidth-1:0]  o_debugmul, 
            output wire signed [2*dataWidth-1:0]  o_psum  
        );
        
            // 1. Data Passing을 위한 Input Registering
            // 입력을 1클럭 잡고 있다가 옆으로 넘겨줌
            reg signed [dataWidth-1:0] a_reg;
            reg signed [dataWidth-1:0] b_reg;
        
            assign o_a = a_reg;
            assign o_b = b_reg;
        
            always @(posedge i_clk) begin
                if (i_rst) begin
                    a_reg <= 0;
                    b_reg <= 0;
                end else if (i_en) begin
                    a_reg <= i_a;
                    b_reg <= i_b;
                end
            end
        
            // PE Instantiation
            // 실제 MAC 수행. 입력으로 Register된 값을 넣음.
            PE #(
                .dataWidth      (dataWidth)
            ) dut (
                .i_clk   (i_clk),
                .i_rst   (i_rst),  
                .i_en    (i_en),      
                .i_a     (a_reg),   // Register된 값을 연산에 사용
                .i_b     (b_reg),   // Register된 값을 연산에 사용
                .o_psum  (o_psum)
                `ifdef DEBUG
                , .o_debugmul (o_debugmul) 
                `endif
            );
        endmodule

        module Systolic_Array #(
            parameter integer dataWidth = 8,
            parameter integer ARRAY_ROW = 4,
            parameter integer ARRAY_COL = 4
        )(
            input  signed [ARRAY_ROW*dataWidth-1:0]              i_row_data,
            input  signed [ARRAY_COL*dataWidth-1:0]              i_col_data,
            input                                                i_clk,
            input                                                i_rst, 
            input                                                i_en,
        
            output signed [ARRAY_ROW*ARRAY_COL*2*dataWidth-1:0]  o_array_out,  
            output signed [ARRAY_ROW*ARRAY_COL*2*dataWidth-1:0]  pe_acc_sum 
            `ifdef DEBUG
          , output signed [2*dataWidth-1:0]                    o_debug_pe00_mul
            `endif
        );
        
            // ---------------------------------------------------------
            // 1. Unpacking Inputs (Packed -> Unpacked Matching)
            // ---------------------------------------------------------
            wire signed [dataWidth-1:0] a_data [0:ARRAY_ROW-1];
            wire signed [dataWidth-1:0] b_data [0:ARRAY_COL-1];
        
            genvar r, c;
            generate
                for (r=0; r<ARRAY_ROW; r=r+1) begin : GEN_SPLIT_A
                    assign a_data[r] = i_row_data[r*dataWidth +: dataWidth];
                end
                for (c=0; c<ARRAY_COL; c=c+1) begin : GEN_SPLIT_B
                    assign b_data[c] = i_col_data[c*dataWidth +: dataWidth];
                end
            endgenerate
        
            // ---------------------------------------------------------
            // 2. Interconnect Wires (Systolic Data Flow)
            // ---------------------------------------------------------
            // a_conn[r][c]   : PE[r][c]의 입력 (Input)
            // a_conn[r][c+1] : PE[r][c]의 출력 (Output -> Next PE Input)
            wire signed [dataWidth-1:0]     a_conn [0:ARRAY_ROW-1][0:ARRAY_COL]; 
            wire signed [dataWidth-1:0]     b_conn [0:ARRAY_ROW][0:ARRAY_COL-1];
            
            // Debug & Result Wiring
            wire signed [2*dataWidth-1:0]   mul_debug [0:ARRAY_ROW-1][0:ARRAY_COL-1];
            wire signed [2*dataWidth-1:0]   acc_debug [0:ARRAY_ROW-1][0:ARRAY_COL-1];
        
            // ---------------------------------------------------------
            // 3. Edge Connection (Array의 왼쪽/위쪽 끝 연결)
            // ---------------------------------------------------------
            generate
                for (r=0; r<ARRAY_ROW; r=r+1) begin : GEN_A_EDGE
                    assign a_conn[r][0] = a_data[r]; 
                end
                for (c=0; c<ARRAY_COL; c=c+1) begin : GEN_B_EDGE
                    assign b_conn[0][c] = b_data[c]; 
                end
            endgenerate
        
            // ---------------------------------------------------------
            // 4. PE Instantiation
            // ---------------------------------------------------------
            generate
                for (r=0; r<ARRAY_ROW; r=r+1) begin : GEN_ROW
                    for (c=0; c<ARRAY_COL; c=c+1) begin : GEN_COL
                        
                        // Result Flattening (2D -> 1D Output Bus)
                        assign o_array_out[(r*ARRAY_COL + c)*2*dataWidth +: 2*dataWidth] 
                               = acc_debug[r][c];
                        assign pe_acc_sum [(r*ARRAY_COL + c)*2*dataWidth +: 2*dataWidth] 
                               = acc_debug[r][c];
                        pe_systolic_cell #(
                            .dataWidth (dataWidth)
                        ) dut (
                            // Data Passing Connections
                            .i_a        (a_conn[r][c]),      
                            .i_b        (b_conn[r][c]),      
                            .o_a        (a_conn[r][c+1]),    
                            .o_b        (b_conn[r+1][c]),    
                            // Control Signals
                            .i_clk      (i_clk),
                            .i_rst      (i_rst), 
                            .i_en       (i_en),
                            // Results
                            .o_debugmul (mul_debug[r][c]),
                            .o_psum     (acc_debug[r][c])
                        );
                    end
                end
            endgenerate
        
            `ifdef DEBUG
                assign o_debug_pe00_mul = mul_debug[0][0];
            `endif
        endmodule

    - 기존 Systolic_Array의 개선은 PE.v에서 대부분 반영하였으며, pe_systolic_cell과 systolic_array의 로직은 기존 DUT 로직과 동일하게 Data Skewing 기반으로 동작하며, 입력 데이터를 레지스터에 저장하여 데이터를 안정적으로 전달하는 방식으로 설계되었다.
    -  최종적으로 Systolic_Array.v용으로 작성된 tb를 최소한의 수정만 적용하여 재사용하였으며, 마찬가지로 입력 데이터(0, 최대 양수, 최대 음수, 일반 값)와 제어 신호(i_en, i_rst)의 모든 유효 조합에 대해 coverpoint를 구성하였다. 검증 결과 functional coverage 100%를 만족하였고, 기능적으로 정상 동작함을 확인하였다.

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_43.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_44.png" width="400"/>

<div align="left">

<div align="center"><img src="https://github.com/yakgwa/Mini_NPU_Ver2/blob/main/Picture/image_45.png" width="400"/>

<div align="left">

### Data_Skewing.v

        module Data_Skewing #(
            parameter integer dataWidth  = 8,      // 데이터 비트폭
            parameter integer ARRAY_ROW  = 4,      // Systolic Array 행 수
            parameter integer ARRAY_COL  = 4       // Systolic Array 열 수
        )(
            input  wire                               i_clk,       
            input  wire                               i_rst,      
            input  wire                               i_en,        
        
            input  wire [ARRAY_ROW*dataWidth-1:0]     i_row_data,  // Raw Row [Row3|Row2|Row1|Row0]
            input  wire [ARRAY_COL*dataWidth-1:0]     i_col_data,  // Raw Col [Col3|Col2|Col1|Col0]
        
            output wire [ARRAY_ROW*dataWidth-1:0]     o_row_data,  // Skewed Row
            output wire [ARRAY_COL*dataWidth-1:0]     o_col_data   // Skewed Col
        );
        
            // ===== 1. Packed → Unpacked Bit Slicing(Matching) =====
            wire signed [dataWidth-1:0] row_in [0:ARRAY_ROW-1];
            wire signed [dataWidth-1:0] col_in [0:ARRAY_COL-1];
        
            genvar r;
            generate
                for (r = 0; r < ARRAY_ROW; r = r + 1) begin : UNPACK_ROW
                    assign row_in[r] = i_row_data[r*dataWidth +: dataWidth];
                end
                for (r = 0; r < ARRAY_COL; r = r + 1) begin : UNPACK_COL
                    assign col_in[r] = i_col_data[r*dataWidth +: dataWidth];
                end
            endgenerate
        
        /*
        row_in, col_in : packed data를 쪼개서 다루기 위해 선언한 inner wire array
        +: : Indexed Vector Part Select를 통해 i_row_data[r*dataWidth +: dataWidth]로 비트를 잘라,
             32bit bus를 8bit x 4개 (row_in[0] ~ row_in[3])로 분리한다.
        */
        
            // ===== 2. Row Skewing Shift Registers =====
            // Row 1: 1단 SR
            reg signed [dataWidth-1:0] row_sr_1_0;
            // Row 2: 2단 SR chain
            reg signed [dataWidth-1:0] row_sr_2_0, row_sr_2_1;
            // Row 3: 3단 SR chain
            reg signed [dataWidth-1:0] row_sr_3_0, row_sr_3_1, row_sr_3_2;
        
            always @(posedge i_clk) begin
                if (i_rst) begin
                    row_sr_1_0 <= 0;
                    row_sr_2_0 <= 0; row_sr_2_1 <= 0;
                    row_sr_3_0 <= 0; row_sr_3_1 <= 0; row_sr_3_2 <= 0;
                end else if (i_en) begin
        // 1-stage
                    row_sr_1_0 <= row_in[1];    
        // 2-stage                      
                    row_sr_2_0 <= row_in[2]; row_sr_2_1 <= row_sr_2_0; 
        // 3-stage
                    row_sr_3_0 <= row_in[3]; row_sr_3_1 <= row_sr_3_0; row_sr_3_2 <= row_sr_3_1; 
                end
            end
        
            // ===== 3. Column Skewing Shift Registers =====
            reg signed [dataWidth-1:0] col_sr_1_0;
            reg signed [dataWidth-1:0] col_sr_2_0, col_sr_2_1;
            reg signed [dataWidth-1:0] col_sr_3_0, col_sr_3_1, col_sr_3_2;
        
            always @(posedge i_clk) begin
                if (i_rst) begin
                    col_sr_1_0 <= 0;
                    col_sr_2_0 <= 0; col_sr_2_1 <= 0;
                    col_sr_3_0 <= 0; col_sr_3_1 <= 0; col_sr_3_2 <= 0;
                end else if (i_en) begin
        // 1-stage
                    col_sr_1_0 <= col_in[1];
        // 2-stage
                    col_sr_2_0 <= col_in[2]; col_sr_2_1 <= col_sr_2_0;
        // 3-stage
                    col_sr_3_0 <= col_in[3]; col_sr_3_1 <= col_sr_3_0; col_sr_3_2 <= col_sr_3_1;
                end
            end
        /*
        Row 번호, Col 번호 (r, c)만큼 Cycle delay를 준다.
        */
        
            // ===== 4. Output Packing =====
            assign o_row_data = {row_sr_3_2, row_sr_2_1, row_sr_1_0, row_in[0]};
            assign o_col_data = {col_sr_3_2, col_sr_2_1, col_sr_1_0, col_in[0]};
        /*
        {...} (Concatenation): 지연된 각각의 신호들을 다시 하나의 큰 버스로 합침
        순서: [Row 3(3 delay), Row 2(2 delay), Row 1(1 delay), Row 0(0 delay)]
        Row 0 / Col 0: row_in[0], col_in[0]이 바로 사용된다. 
        (레지스터를 거치지 않은 0 사이클 지연 값)
        나머지는 각 파이프라인의 마지막 레지스터 값(row_sr_3_2 등)을 출력으로 내보냄.
        */
        endmodule


- DUT


















