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


























<div align="left">


