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
 
- Testbench

        `timescale 1ns/1ps
        
        module tb_mac_pe;
        
            localparam int DATA_W = 8;
            localparam int ACC_W = 2*DATA_W;
        
            // Step 1) Interface Definition (생략)
            logic                  clk;
            logic                  rst_n;
            logic                  clr;
            logic                  en;
            logic [DATA_W-1:0]     a;
            logic [DATA_W-1:0]     b;
            logic [ACC_W-1:0]      mul;
            logic [ACC_W-1:0]      acc_sum;
            
            //==========================================================
            // Step 2) Constrained Random Transaction
            //==========================================================
            class mac_txn;
                rand bit [DATA_W-1:0] a;
                rand bit [DATA_W-1:0] b;
                rand bit              en;
                rand bit              clr;
                // clr 5% 정도 constraint random
                constraint c_clr { clr dist {1 := 5, 0 := 95}; }
                // en 50% 정도 constraint random
                constraint c_en  { en  dist {1 := 50, 0 := 50}; }
            endclass
            
            mac_txn ma = new; //constraint random class instantiation

    - Step 2) Constrained Random Transaction
      - dist로 확률 가중치를 두어 constraint random 값을 mac_txn class로 정의
      - clr dist {1 := 5, 0 := 95}; : 5% 가량 clr=1이 생성되도록 constraint
      - en  dist {1 := 50, 0 := 50}; : 50% 가량 en=1이 생성되도록 constraint
