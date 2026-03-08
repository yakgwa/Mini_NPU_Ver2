## [AI Accelerator 기초] - Systolic Array와 FSM 구현3 (2D 4X4 Systolic Array +  Controller + ReLU)

### Systolic Array  + Controller + Activation Function(ReLU)

- DUT
1️⃣2️⃣3️⃣4️⃣5️⃣6️⃣7️⃣8️⃣9️⃣🔟
    - 0️⃣ Parameter & Port 정의

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
