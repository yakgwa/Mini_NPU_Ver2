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

                    //==========================================================
                    // Step 3) Top Module Instantiation
                    //==========================================================
                     mac_pe #(.DATA_W(DATA_W), .ACC_W(ACC_W)) dut (
                        .clk     (clk),
                        .rst_n   (rst_n),
                        .clr     (clr),
                        .en      (en),
                        .a       (a),
                        .b       (b),
                        .mul     (mul),
                        .acc_sum (acc_sum)
                     );
                //========================================================
                // Clock / Reset
                //========================================================
                    initial begin
                        clk = 1'b0;
                        forever #5 clk=~clk; //100MHz
                    end
                    
                    initial begin
                        rst_n = 1'b0;
                        clr = 1'b0;
                        en = 1'b0;
                        a = '0;
                        b = '0;
                        
                        #30;
                        rst_n = 1'b1; //after 30ns -> rst_n=1, start
                     end
                //==========================================================
                // Golden Model & Checker (Watchpoint)
                //==========================================================
                    logic [ACC_W-1:0] ref_mul;
                    logic [ACC_W-1:0] ref_sum;
                
                    int cycles_checked;
                    int err_acc_cnt;
                    int err_mul_cnt;
                    
                // golden model: DUT와 동일한 동작을 testbench 안에서 구현
                     assign ref_mul = a * b; //always -> assign (data-mismatch debugging point)
                     always @(posedge clk or negedge rst_n) begin
                        if (!rst_n) begin
                            ref_sum <= {ACC_W{1'b0}};
                        end else if (clr) begin
                            ref_sum <= {ACC_W{1'b0}};
                        end else if (en) begin
                            ref_sum <= ref_sum + ref_mul;
                        end
                      end
                // Checker : DUT vs golden model
                    always @(posedge clk or negedge rst_n) begin
                        if(!rst_n) begin
                            cycles_checked <= 0;
                            err_acc_cnt <= 0;
                            err_mul_cnt <= 0;
                        end else begin
                            cycles_checked++;
                
                      // mul 비교
                      if (mul !== ref_mul) begin
                        err_mul_cnt++;
                        $display("ERROR! MUL mismatch: dut=%0d, ref=%0d, time=%0d ns",
                                 mul, ref_mul, $time);
                      end else begin
                        $display("PASS!  MUL match   : dut=%0d, ref=%0d, time=%0d ns",
                                 mul, ref_mul, $time);
                      end
                        
                      // acc_sum 비교
                      if (acc_sum !== ref_sum) begin
                        err_acc_cnt++;
                        $display("ERROR! ACC_SUM mismatch: dut=%0d, ref=%0d, time=%0d ns",
                                 acc_sum, ref_sum, $time);
                      end else begin
                        $display("PASS!  ACC_SUM match   : dut=%0d, ref=%0d, time=%0d ns",
                                 acc_sum, ref_sum, $time);
                      end
                      end
                    end

    - Step 3) DUT Instantiation + Clock/Reset 생성 + Golden Model & Checker
      - step 3의 핵심 golden model에 대한 내용을 자세히 설명하면 다음과 같다.
      - instantiation된 dut와 동일한 reference model(=golden model)을 생성하기 위해 ref_mul, ref_sum 등의 reference signal 및 마지막 summary에서 PASS/FAIL 판단을 위한 err_acc(mul)_cnt signal을 생성한다.
      - 이후, golden model을 구현하기 위해 dut와 동일한 논리로 코드를 만들고, rst_n=0이면 카운터는 0으로 세팅되고, rst_n=1일 경우에만 앞서 정의한 err_mul(acc)_cnt 비교를 수행한다.

                if(!rst_n) begin
                   ...
                end else begin
                   cycles_checked++;
                   // 비교는 여기서만 수행
                end

      - 이때, 비교는 'mul mismatch'와 'acc_sum mismatch'를 분리함으로써 mul mismatch면 comb multiplier부터 의심할 수 있도록, acc mismatch면 en/clr/rst_n/누산 순서를 의심할 수 있도록 테스트를 수행한다.
     
            //========================================================
            // Step 4) Functional Coverage
            //========================================================
            
            covergroup cg_mac;
                cp_en  : coverpoint ma.en  { bins b0={0}; bins b1={1}; }
                cp_clr : coverpoint ma.clr { bins b0={0}; bins b1={1}; }            
        
                cp_a : coverpoint ma.a {
                bins zero   = {0};
                bins one = {1};
                bins max = {(2**DATA_W)-1};
                bins mid = default;
                }
        
                cp_b : coverpoint ma.b {
                bins z   = {0};
                bins one = {1};
                bins max = {(2**DATA_W)-1};
                bins mid = default;
                }
            endgroup
            
            cg_mac cg = new; //coverage class instantiation

    - Step 4) Functional Coverage
      - functional coverage 정도를 확인하기 위해 cg_mac에 대한 covergroup class를 구성하였다. 
      - 기본적으로 en/clr이 0/1 둘 다 나왔는지 확인하며, a, b에 대해서는 0, 1, max 같은 edge 값은 별도 bin으로 강제 관찰 및 나머지는 default로 뭉쳐서 나머지 값에 대해서도 나온 정도로만 확인한다.
      - 검증 코드 작성 시, TB stimulus(ma.a/ma.b)를 coverage 대상으로 잡음으로써 dut 입력이 실제로 어떤 분포로 들어갔는지 수치적으로 확인하였다. (단, 정교한 cross까지 수행하진 않았고, edge 값 중심으로 구성함)
     
            //========================================================
            // Step 5) main
            //========================================================
            localparam int  MAX_CYCLES  = 2000;   // 무한 repeat 대신 안전하게 제한
            localparam real COV_TARGET  = 100.0;  // 목표 커버리지(%)
        
            initial begin
            //waiting 'reset off'
            @(posedge rst_n);
            @(posedge clk);
            
            $display("[TB] Time-based Random Stress START, duration = %0d ns", $time);
                   
            repeat(MAX_CYCLES) begin
                @(negedge clk);
                assert(ma.randomize());
                    
                en = ma.en;
                clr = ma.clr;
                a = ma.a;
                b = ma.b;
                    
                cg.sample();
                    
                @(posedge clk); //next clk
                    
                if (cg.get_inst_coverage() >= COV_TARGET) begin
                     $display("[TB] Coverage reached %0.2f%%, time=%0t",
                           cg.get_inst_coverage(), $time);
                     break;
                end          
            end
        
            //========================================================
            // Summary
            //========================================================
            $display("==================================================");
            $display("[TB] Simulation Summary");
            $display("  Cycles checked     : %0d",    cycles_checked);
            $display("  MUL mismatches     : %0d",    err_mul_cnt);
            $display("  ACC_SUM mismatches : %0d",    err_acc_cnt);
            if (err_mul_cnt == 0 && err_acc_cnt == 0) begin
              $display("  RESULT             : PASS");
            end else begin
              $display("  RESULT             : FAIL");
            end
            $display("--------------------------------------------------");
            $display("[TB] Functional Coverage");
            $display("  TOTAL      : %0.2f %%", cg.get_inst_coverage());
            $display("  en         : %0.2f %%", cg.cp_en.get_inst_coverage());
            $display("  clr        : %0.2f %%", cg.cp_clr.get_inst_coverage());
            $display("==================================================");
        
            $finish;
          end
        endmodule

    - Step 5) Main : Stimulus loop
      - simulation이 끝나지 않는 것을 방지하기 위해 MAX_CYCLES를 정의하고, 만약 coverage 목표를 조기에 달성하면 조기 종료할 수 있게 끔 COV_TARGET을 정의하였다. 최초 rst_n=1으로 변경한 다음부터, 루프를 돌면서 randomize stimulus를 진행한다.

            repeat(MAX_CYCLES) begin
                @(negedge clk);
                ..
                @(posedge clk); //posedge에서 값을 실제로 반영
                    
                if (cg.get_inst_coverage() >= COV_TARGET) begin
                     $display("[TB] Coverage reached %0.2f%%, time=%0t",
                           cg.get_inst_coverage(), $time);
                     break;
                end          
            end








