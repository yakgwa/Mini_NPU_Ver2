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


- 기존 systolic_controller_relu.v 모듈은 후처리 로직과 데이터 정렬 로직이 혼재되어 있어 복잡도가 높았다. 이에 따라 설계의 가독성과 모듈화를 위해, Systolic Array 입력단의 핵심인 Data Skew Logic을 별도 모듈로 분리하였다. 해당 모듈은 메모리에서 읽어온 Flat(Packed) Row/Col 데이터를, Systolic Array 구조에 맞는 대각선 형태의 Wavefront로 변환하기 위해 Temporal Skewing을 적용하는 역할을 수행한다.
- 올바른 연산(Input × Weight)이 수행되려면, 데이터가 전파되는 하드웨어적 특성을 고려해야 한다​
    - 데이터 전파 지연 (Propagation Delay): 각 PE 내부에는 데이터를 잠시 저장했다가 옆으로 전달하는 1-cycle 레지스터(a_reg, b_reg)가 존재한다. 즉, 데이터는 한 번에 모든 PE로 퍼지는 것이 아니라, 매 클럭마다 한 칸씩 이동한다.(예: PE[0][0]를 통과한 Input 데이터는 1 cycle 뒤에야 PE[0][1]에 도달)
    - 타이밍 동기화 (Synchronization): Input 데이터가 옆으로 이동하는 동안, 짝이 되는 Weight 데이터도 그 자리에서 기다리거나 늦게 도착해야만 정확한 곱셈(Input[k] × Weight[k])이 이루어집니다. 따라서 Input이 이동하는 거리만큼 Weight 데이터 투입 시점도 지연시켜야 두 데이터가 정확한 위치에서 만날 수 있다.
- 동작 원리 : 대각선 Wavefront 형성
    - 이 모듈은 Shift Register를 사용하여 인덱스가 증가할수록 더 많은 지연을 주는 방식으로 데이터를 밀어넣어서 입력한다. Skewing Pattern은 인덱스와 동일한 수의 클럭 사이클만큼 지연시킨다.
      - Row/Col 0: 0 Cycle Delay (즉시 진입)
      - Row/Col 1: 1 Cycle Delay (레지스터 1개 통과)​
      - Row/Col 2: 2 Cycle Delay (레지스터 2개 통과)
      - Row/Col 3: 3 Cycle Delay (레지스터 3개 통과)
- 타이밍 예시 (Timing Example)
    - 메모리에서 읽어온 원본 데이터(Raw)가 Skew Logic을 통과하면(Skewed), 아래와 같이 대각선 형태로 줄을 서서 Array에 진입한다. 
      - Cycle 0: [ A0, 0, 0, 0 ] → 첫 번째 데이터만 진입
      - Cycle 1: [ A1, B0, 0, 0 ] → 두 번째 줄 진입 시작​
      - Cycle 2: [ A2, B1, C0, 0 ] → 세 번째 줄 진입 시작
      - Cycle 3: [ A3, B2, C1, D0 ] → 전체 4x4 PE에 유효 데이터 도달 (Wavefront 완성)
- 데이터 정의 (Terminology)
    - Raw Data: NPU Top의 FSM이 메모리에서 가져와 MUX를 통해 내보낸 직후의 데이터로, 아직 정렬되지 않은 상태
    - Skewed Data: 본 모듈의 Shift Register 체인을 통과하여 시간차(Delay)가 적용된 데이터로, 실제 Systolic Array의 포트(i_row, i_col)로 연결

### ReLU.v / Sig_ROM.v - Reference Model에서 작성한 DUT와 동일

- DUT

        module ReLU  #(parameter dataWidth=8,weightIntWidth=4) ( 
            //include.v에 의해 module instantiation 시, dataWidth=8
            input                          i_clk,
            input      [2*dataWidth-1:0]   i_in,  
            output reg [dataWidth-1:0]     o_out
            );
        
            always @(posedge i_clk)
            begin
                if($signed(i_in) >= 0) //i_in>=0이면 그대로 전달 (i_in 2의 보수로 해석)
                begin
                    if(|i_in[2*dataWidth-1-:weightIntWidth+1]) //over flow to sign bit of integer part
                        o_out <= {1'b0,{(dataWidth-1){1'b1}}}; //positive saturate
                    else
                        o_out <= i_in[2*dataWidth-1-weightIntWidth-:dataWidth];
                end
                else                //x<0이면 무조건 0
                    o_out <= 0;      
            end
        endmodule

    - Reference Model 코드 분석 시 꼼꼼하게 분석한 부분들을 다시 한 번 짚어보면, 뉴런 내부에서 input × weight (곱셈), sum + bias (누산)에 따른 확장된 16bit가 입력으로 들어오며, ReLU 출력은 다시 다음 layer의 입력 인터페이스 폭을 고정시키기 위해 dataWidth(8bit)로  scaled-down (re-quantization)된다.
    - ReLU 입력 i_in의 비트 구간 의미는 dataWidth=8, weightIntWidth=4 기준으로 다음과 같다.
      - i_in [15:12] | [11:4] | [3:0]
      - i_in[11:4] : 다음 layer로 전달할 실제 output (8bit)
      - i_in[3:0] : 소수부 LSB → truncation으로 제거
      - i_in[15:12]: 출력 폭으로 표현 불가능한 상위 비트 → overflow 영역
    - 한편, Overflow 판단과 Saturation은 다음과 같다.
      - if(|i_in[2*dataWidth-1-:weightIntWidth+1]) o_out <= {1'b0,{(dataWidth-1){1'b1}}};
      - dataWidth=8 → |i_in[15-:5]=|x[15:11]와 같은 의미
    - 상위 비트 중 하나라도 1이면, 출력 8bit로 표현 불가하므로, 최대 양수값(0x7F)으로 saturation
    - ⚠️ 단, overflow는 i_in[15:12]가 기준이지만, i_in[11]를 포함한 보수적 saturation 정책을 사용
    - Overflow가 없을 때는 출력 비트를 선택한다.
      - o_out <= i_in[2*dataWidth-1-weightIntWidth-:dataWidth]; // dataWidth=8 → o_out = i_in[11:4]
    - 정수부 위치를 기준으로 비트 정렬(alignment) 및 단순 상/하위 비트 선택이 아니라 → fixed-point 스케일 유지한다. LSB(i_in[3:0])는 fixed-point의 소수부에 해당하므로 버린다.
 
        module Sig_ROM #(parameter inWidth=10, dataWidth=8) (
            input                   i_clk,
            input   [inWidth-1:0]   i_in,
            output  [dataWidth-1:0] o_out
            );
            
            reg [dataWidth-1:0] mem [2**inWidth-1:0];
            reg [inWidth-1:0] y;
        	
        	initial
        	begin
        		$readmemb("sigContent.mif",mem);
        	end
            
            always @(posedge i_clk)
            begin
                if($signed(i_in) >= 0)
                    y <= i_in+(2**(inWidth-1));
                else 
                    y <= i_in-(2**(inWidth-1));      
            end
            
            assign o_out = mem[y];
        endmodule      

    - 마찬가지로 Reference Model 코드 분석 시 꼼꼼하게 분석한 부분들을 다시 한 번 짚어보면 sigmoid 함수 자체는 지수함수 및 나눗셈이 필요하기 때문에 다음과 같이 ROM(LUT) 기반으로 근사한다.
    - $readmemb로 읽어 ROM을 초기화 할 때, LUT의 값들을 미리 계산해 파일로 저장해둔 sigContent.mif을 사용한다. 여기서 새롭게 등장하는 parameter인 inWidth(=sigmoidSize=10)은 입력 폭이 아닌, LUT 해상도를 의미한다. 예컨대, sigmoidSize = 10이라면, 주소 개수 = 2^10 = 1024이므로, ROM 엔트리 = 1024개가 되는 것이다.
    - Sig_ROM 내부 구조는 다음과 같다.
      - ROM 배열 : reg [dataWidth-1:0] mem [2**inWidth-1:0];
      - 주소 레지스터 : reg [inWidth-1:0] y;     
    - Sig_ROM에서 주소(y) 만드는 방식은 우선 입력 i_in(signed)를 ROM 주소 y (unsigned)로 매핑한다.

           if($signed(i_in) >= 0) y <= i_in + (2**(inWidth-1)); 
           else y <= i_in - (2**(inWidth-1)); end assign out = mem[y]; 

    - signed 입력 i_in를 그대로 쓰면 음수 주소가 나오므로, ± 2^(inWidth-1) 만큼 이동시켜 중앙을 0 근처로 맞추는(offset) 방식을 쓴다.

### maxFinder.v - Reference Model에서 작성한 DUT와 동일

- DUT

        module maxFinder #(parameter numInput=10,parameter inputWidth=16)(
            input                               i_clk,
            input [(numInput*inputWidth)-1:0]   i_data,
            input                               i_valid,
            output reg [31:0]                   o_data,
            output reg                          o_data_valid
            );
        
            reg [inputWidth-1:0] maxValue;
            reg [(numInput*inputWidth)-1:0] inDataBuffer;
            integer counter;
        
            always @(posedge i_clk) begin
                o_data_valid <= 1'b0;
                if(i_valid)
                begin
                    maxValue <= i_data[inputWidth-1:0];
                    counter <= 1;
                    inDataBuffer <= i_data;
                    o_data <= 0;
                end
                else if(counter == numInput)
                begin
                    counter <= 0;
                    o_data_valid <= 1'b1;
                end
                else if(counter != 0)
                begin
                    counter <= counter + 1;
                    if(inDataBuffer[counter*inputWidth+:inputWidth] > maxValue)
                    begin
                        maxValue <= inDataBuffer[counter*inputWidth+:inputWidth];
                        o_data <= counter;
                    end
                end
            end
        endmodule

    - maxFinder는 최종적으로 inputWidth=16의 입력 비트 폭을 가진, Layer3의 뉴런 개수인 numInput=10이 packed array [(numInput*inputWidth)-1:0] i_data로 numInput개 입력이 한 번에 버스 형태로 들어오게 된다. 자세한 설명은 이전 Reference Model 코드 분석에서 확인할 수 있다

### Activation_Unit.v

- DUT

        `timescale 1ns / 1ps
        `include "include.v"
        
        module Activation_Unit #(
            parameter dataWidth = 8
            )(
            input                        i_clk,
            input      [2*dataWidth-1:0] i_psum,   //MUL + ADD (가중합)
            input      [2*dataWidth-1:0] i_bias,   //bias
            input                        i_actsel, //Act Func Select Signal (0:sig, 1:ReLU)
            output reg [dataWidth-1:0]   o_out     //Act Func Output
            );
        /*
        주의!!
        Activation_Unit에서 사용할 Bias는 결국 추후 설명할 Bias_Bank에서 가져오므로,
        그 모듈에서 8-bit bias를 16-bit로 확장할 때 좌측 시프트를 수행함.
        따라서 사용할 Bias는 이미 {bias_val, 8'b0} 형태이므로
        여기서 다시 좌측 시프트를 수행하면,
        {i_bias[15:0], 8'b0} = 24-bit → 하위 16-bit 잘림 → bias = 0
        항상 Bias=0이 되어버린다.
        따라서 bias = i_bias (이미 Bias_Bank에서 시프트 완료)
        */
            //bias : fixed-point Quantization 반영은
            wire [2*dataWidth-1:0] bias;
            assign bias = i_bias; 
            
            //bias_add_res  : bias를 더한 결과를 저장할 신호 (guard bit +1)
            //sum_saturated : Act Func의 입력 값이 저장될 레지스터 
            wire [2*dataWidth:0]   bias_add_res;
            reg  [2*dataWidth-1:0] sum_saturated;
            
            //i_psum + i_bias
            assign bias_add_res = $signed(i_psum) + $signed(bias);
            
            //MSB guard bit를 버리고 나머지 비트만 기재하여 sum_saturated에 저장
            always @(*) begin
                // 포화 처리: BiasAdd가 양수일 경우 포화 처리, 음수일 경우 포화 처리
                if(!bias[2*dataWidth-1] & !i_psum[2*dataWidth-1] & bias_add_res[2*dataWidth-1]) begin
                    // 양수 오버플로우: 더한 결과가 양수에서 오버플로우 되면, sum을 최댓값으로 설정
                    sum_saturated[2*dataWidth-1]   <= 1'b0;
                    sum_saturated[2*dataWidth-2:0] <= {2*dataWidth-1{1'b1}};
                end else if (bias[2*dataWidth-1] & i_psum[2*dataWidth-1] & !bias_add_res[2*dataWidth-1]) begin
                    // 음수 오버플로우: 더한 결과가 음수에서 오버플로우 되면, sum을 최솟값으로 설정
                    sum_saturated[2*dataWidth-1]   <= 1'b1;
                    sum_saturated[2*dataWidth-2:0] <= {2*dataWidth-1{1'b0}};
                end else begin
                    // 오버플로우가 발생하지 않으면 정상적으로 더한 값을 저장
                    sum_saturated <= bias_add_res;
                end
            end
            
            //sig_out  : Sigmoid Output Store
            //relu_out : ReLU Output Store
            wire [dataWidth-1:0] sig_out;
            wire [dataWidth-1:0] relu_out;
            
            //Sigmoid Instantiation
            Sig_ROM #(
                .inWidth(`sigmoidSize), 
                .dataWidth(dataWidth)
            ) sig_inst (
                .i_clk(i_clk),
                .i_in(sum_saturated[2*dataWidth-1 -: `sigmoidSize]), 
                .o_out(sig_out)
            );
            
            //ReLU Instantiation
            ReLU #(
                .dataWidth(dataWidth),
                .weightIntWidth(`weightIntWidth)
            ) relu_inst (
                .i_clk(i_clk),
                .i_in(sum_saturated),
                .o_out(relu_out)
            );
        
            always @(*) begin
                if (i_actsel == 1'b0) 
                    o_out = sig_out;
                else 
                    o_out = relu_out;
            end
        endmodule

    - Activation Function은 주어진 가중합과 바이어스를 더한 뒤, 선택된 활성화 함수(Sigmoid/ReLU)를 적용하여 최종 출력을 생성하는 기능을 수행한다. 입력으로 들어오는 psum_in (MUL + ADD된 가중합)과 bias_in (바이어스)을 더한 후, Activation Function에 적용하기 전에 이를 saturated value로 변환한다.

                1. bias_add_res 계산:
                assign bias_add_res = $signed(i_psum_in) + $signed(i_bias);
                문제점: bias_add_res를 계산할 때, Q4.4 형식으로 Fixed-Point 연산 시,
                        곱셈 결과는 정수부 8bit, 소수부 8bit로 나누어진다.
                        이때, i_bias를 그대로 적용해버리면, 소수부 8bit에 bias가 적용될 위험이 있다.
                해결  : i_bias를 left shift하여 8bit로 확장하고, 나머지 하위 8bit는 0으로 padding함으로써,
                        정수부 8bit와 bias가 더해지도록 한다.
                
                2. sum_saturated 값 처리:
                always @(*) begin
                    sum_saturated = bias_add_res[2*dataWidth-1:0];
                end
                문제점: sum_saturated에 할당되는 값은 bias_add_res의 하위 dataWidth 비트만 저장하게 된다. 
                        이때, 상위 비트가 잘리게 되며, 그로 인해 정보 손실이 발생할 수 있다. 
                해결  : saturation 처리를 추가하여, 범위 내에서 값을 유지시켜야 한다. 
                        따라서 최댓값과 최솟값으로 클리핑하여 오버플로우 및 정보 손실을 방지한다.
