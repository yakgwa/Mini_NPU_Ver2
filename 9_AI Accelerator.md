## [AI Accelerator 기초] - Systolic Array와 FSM 구현1 (PE, 1D PE Chain) 
AI 가속기 하드웨어를 이해하기 위해서는, 연산 구조 자체를 어떻게 설계하고 제어하는지를 단계적으로 뜯어보는 과정이 필요하다. 이전까지는 Google TPU, Gemmini 논문을 살펴보며 Systolic Array 기반 가속기의 철학과 데이터플로우(OS/WS/IS)를 정리했고, 동시에 ZyNet으로 자동 생성된 RTL DUT와 Testbench를 분석하면서 실제 뉴럴넷 하드웨어 구조를 확인했다. 그 과정에서 ZyNet 구조가 가지는 PPA 관점의 한계와, 데이터 재사용 및 제어 방식 측면에서의 개선 여지도 함께 정리하였다. 이러한 분석을 바탕으로 최초의 개선 방향 모델을 개념적으로 제시했다. 다만, 곧바로 최종 형태의 가속기를 설계하기보다는, 이번 글부터는 실제로 가속기를 설계하는 것이기 때문에 보다 기초적으로 접근함으로써 단일 PE(Processing Element)부터 시작하여 1D 1×4 PE Chain으로 확장하고, 최종적으로 OS(Output-Stationary) 기반 4×4 Systolic Array로 단계적으로 확장하는 흐름으로 설계를 진행한다. 따라서 이번 단계에서의 목표는 “잘 동작하는 구조를 만드는 것”이다. 따라서 타이밍 최적화, 면적 절감, 전력 분석과 같은 요소는 배제하고, 기능 검증 수준에서만 설계와 검증을 수행한다. 또한, 복잡한 스케줄러 대신 FSM 기반의 단순 컨트롤러, 그리고 ReLU를 결합하여, 단순 연산이 하드웨어 상에서 어떻게 흘러가고 제어되는지에 집중한다.

*참고 : tb는 항상 

step1) interface definition ▶ step2) Constrained Random Transaction ▶ Step 3) Top Module Instantiation ▶ Step 4) Functional Coverage ▶ Step 5) Main 
순서로 작성하였다.

### PE(Processing Element)
- DUT

        module mac_pe #(
            parameter integer DATA_W = 8,
            parameter integer ACC_W = 2*DATA_W
        )(
            input wire [DATA_W-1:0] a,       //op1
            input wire [DATA_W-1:0] b,       //op2
            
            input wire              clk,     //clock
            input wire              rst_n,   //reset (neg sensitive)
            input wire              clr,     //first power on -> register state definition 
            input wire              en,      //mul + add enable
            
            output wire [ACC_W-1:0] mul,
            output reg  [ACC_W-1:0] acc_sum  //mul + add 
            );
        
            //mul
            assign mul = a * b;
            
            //adder
            always @(posedge clk or negedge rst_n) begin
                if(!rst_n) begin
                    acc_sum <= {ACC_W{1'b0}};
                end else if (clr) begin
                    acc_sum <= {ACC_W{1'b0}};
                end else if (en) begin 
                    acc_sum <= acc_sum + mul; 
                end
            end
        endmodule

    - mac_pe는 Systolic Array의 가장 작은 MAC 수행 최소 연산 단위인 PE 역할을 한다. 입력 a, b를 곱해(mul) 필요할 때만 누적(acc_sum <= acc_sum + mul)한다.
    - PE는 의도적으로 mul(comb logic)과 acc_sum(seq logic)을 분리하여, 입력이 바뀌면 즉시 mul을 진행하고, acc_sum은 clock edge에서만 상태가 갱신되는 register이다. 이때 input을 모두 unsigned로 두었는데, 실제 fixed-point까지 진행하게 되면 signed로 두어 연산을 처리해야 한다.
 
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




