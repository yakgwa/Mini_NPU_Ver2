## [Reference Model 분석] - CNN-Handwritten-Digit-MNIST (Code Main)
    
    `define pretrained
    `define numLayers 4
    `define dataWidth 8
    `define numNeuronLayer1 30
    `define numWeightLayer1 784
    `define Layer1ActType "sigmoid"
    `define numNeuronLayer2 20
    `define numWeightLayer2 30
    `define Layer2ActType "sigmoid"
    `define numNeuronLayer3 10
    `define numWeightLayer3 20
    `define Layer3ActType "sigmoid"
    `define numNeuronLayer4 10
    `define numWeightLayer4 10
    `define Layer4ActType "hardmax"
    `define sigmoidSize 10
    `define weightIntWidth 4

✅ Activation Function

- ReLU

		module ReLU  #(parameter dataWidth=16,weightIntWidth=4) ( 
		//include.v에 의해 module instantiation 시, dataWidth=8
		    input           clk,
		    input   [2*dataWidth-1:0]   x,  
		    output  reg [dataWidth-1:0]  out
		);

		always @(posedge clk)
		begin
		    if($signed(x) >= 0) //x>=0이면 그대로 전달 (x를 2's complement signed값으로 해석)
		    begin
		        if(|x[2*dataWidth-1-:weightIntWidth+1]) //over flow to sign bit of integer part
		            out <= {1'b0,{(dataWidth-1){1'b1}}}; //positive saturate
		        else
		            out <= x[2*dataWidth-1-weightIntWidth-:dataWidth];
		    end
		    else                //x<0이면 무조건 0
		        out <= 0;      
		end
		endmodule
  
- 1️⃣ 왜 ReLU 입력 폭이 2×dataWidth인가?
	- 뉴런 내부에서는 input × weight (곱셈), sum + bias (누산)
	- 이 과정에서 정수부가 확장되므로 dataWidth=8이라도 결과는 16bit 필요
	- 따라서 ReLU 입력 x는 input [2*dataWidth-1:0] x // ex) [15:0]

- 2️⃣ ReLU 출력은 왜 다시 dataWidth(8bit)인가?
	- 다음 layer의 입력 인터페이스 폭은 항상 dataWidth로 고정
	- 만약 16bit를 그대로 넘기면, 다음 layer의 비트 폭이 선형적으로 증가
	- 그래서 ReLU에서 scaled-down (re-quantization) 수행

​

- 3️⃣ ReLU 입력 x의 비트 구간 의미
	-dataWidth=8, weightIntWidth=4 기준:

			x[15:12] | x[11:4] | x[3:0]
			x[11:4] : 다음 layer로 전달할 실제 output (8bit)
			x[3:0] : 소수부 LSB → truncation으로 제거
			x[15:12]: 출력 폭으로 표현 불가능한 상위 비트 → overflow 영역

​
- 4️⃣ Overflow 판단과 Saturation

		if(|x[2*dataWidth-1-:weightIntWidth+1]) out <= {1'b0,{(dataWidth-1){1'b1}}};
		dataWidth=8 → |x[15-:5]=|x[15:11]와 같은 의미

	- 상위 비트 중 하나라도 1이면, 출력 8bit로 표현 불가하므로, 최대 양수값(0x7F)으로 saturation
	- ⚠️ 단, overflow는 x[15:12]가 기준이지만, 현재 구현은 x[11]를 포함한 보수적 saturation 정책을 사용​

- 5️⃣ Overflow가 없을 때 출력 비트 선택
		
		out <= x[2*dataWidth-1-weightIntWidth-:dataWidth]; // dataWidth=8 → out = x[11:4] 

	- 정수부 위치를 기준으로 비트 정렬(alignment)

	- 단순 상/하위 비트 선택이 아니라 → fixed-point 스케일 유지

- 6️⃣ LSB(x[3:0])는 버려짐
	- LSB는 fixed-point의 소수부에 해당
	- 그러나, 다음 layer는 8bit 입력만 사용하므로, 이를 비트 폭이 늘어나는 것을 막고, inference 과정은 weight 양자화 노이즈/activation 비선형성/상대적 크기 비교(hardmax)라는 점에서 근사화한다.

- Sigmoid

		module Sig_ROM #(parameter inWidth=10, dataWidth=16) (
		    input           clk,
		    input   [inWidth-1:0]   x,
		    output  [dataWidth-1:0]  out
		    );
		    
		    reg [dataWidth-1:0] mem [2**inWidth-1:0];
		    reg [inWidth-1:0] y;
			
			initial
			begin
				$readmemb("sigContent.mif",mem);
			end
		    
		    always @(posedge clk)
		    begin
		        if($signed(x) >= 0)
		            y <= x+(2**(inWidth-1));
		        else 
		            y <= x-(2**(inWidth-1));      
		    end
		    
		    assign out = mem[y];
		    
		endmodule
  
- 1️⃣ 실시간 연산이 너무 무겁다

		sigmoid(x) = 1 / (1 + e^{-x}) 

	- 즉, 지수함수와 나눗셈이 필요해서 FPGA에서 실시간 연산으로 구현하면 자원/지연이 매우 큼
	- 따라서 ZyNet은 ROM(LUT) 기반으로 근사

- 2️⃣ sigContent.mif의 역할: “sigmoid 함수표”
	- sigmoid LUT의 값들을 미리 계산해 파일로 저장해둔 것이 sigContent.mif
	- Sig_ROM 내부에서 sigContent.mif를 $readmemb로 읽어 ROM 초기화
	- 즉, sigContent.mif = sigmoid 값을 근사화한 값이 들어있는 LUT

- 3️⃣ inWidth(=sigmoidSize=10)의 의미: 입력 폭이 아닌, LUT 해상도
	- include.v에서 sigmoidSize = 10
	- 이 값은 “sigmoid 입력 데이터 폭”이라기보다, LUT 주소 폭에 해당하므로 이를 통해 해상도를 결정
	- inWidth가 10이면, 주소 개수 = 2^10 = 1024이므로, ROM 엔트리 = 1024개가 됨

			Sig_ROM 내부 구조
			ROM 배열 : reg [dataWidth-1:0] mem [2**inWidth-1:0]; 
			주소 레지스터 : reg [inWidth-1:0] y; 
​
- 4️⃣ Sig_ROM에서 주소(y) 만드는 방식 (signed 입력 처리)
	- Sig_ROM은 입력 x(signed)를 ROM 주소 y (unsigned)로 매핑한다.

			always @(posedge clk) begin 
			
			        if($signed(x) >= 0) y <= x + (2**(inWidth-1)); 
			
			        else y <= x - (2**(inWidth-1)); end assign out = mem[y]; 

	- ROM 주소 공간은 0 ~ 2^inWidth-1
	- signed 입력 x를 그대로 쓰면 음수 주소가 나오므로, ± 2^(inWidth-1) 만큼 이동시켜 중앙을 0 근처로 맞추는(offset) 방식을 쓴다.
	- 최종 출력은 out = mem[y] 

- 5️⃣ 그럼 sigmoid의 “실제 입력 x”는 어디서 오나?
	- Neuron.v에서 Sig_ROM Instantiation 확인

			Sig_ROM #(.inWidth(sigmoidSize), .dataWidth(dataWidth)) s1(
			
			     .clk(clk), 
			
			     .x(sum[2*dataWidth-1-:sigmoidSize]), 
			
			     .out(out) ); 

	- 즉, sigmoid 입력으로 들어가는 x는 MAC 결과 sum의 상위 비트 sigmoidSize(=10)개만 잘라서 넣는다.

✅ Weight_Memory.v - 각 뉴런이 사용하는 가중치 저장/제공 전용 메모리 블록

		`include "include.v"
		
		module Weight_Memory #(parameter numWeight = 3, neuronNo=5,layerNo=1,
		                       addressWidth=10,dataWidth=16,weightFile="w_1_15.mif") 
		    ( 
		    input clk,
		    input wen,
		    input ren,
		    input [addressWidth-1:0] wadd,
		    input [addressWidth-1:0] radd,
		    input [dataWidth-1:0] win,
		    output reg [dataWidth-1:0] wout);
		    
		    reg [dataWidth-1:0] mem [numWeight-1:0];
		
		    `ifdef pretrained
		        initial begin
			        $readmemb(weightFile, mem);
			    end
			`else
				always @(posedge clk)
				begin
					if (wen)
					begin
						mem[wadd] <= win;
					end
				end 
		    `endif
		    
		    always @(posedge clk)
		    begin
		        if (ren)
		        begin
		            wout <= mem[radd];
		        end
		    end 
		endmodule
		
- 1️⃣ 파라미터 구조

		parameter
		  numWeight   = 3,     // 해당 뉴런이 가지는 가중치 개수 (layer 1이면 784개)
		  neuronNo    = 5,      // 뉴런 번호 (파일 구분용)
		  layerNo     = 1,        // 레이어 번호 (파일 구분용)
		  addressWidth= 10, // 주소 폭 (메모리에서 '몇 번째 칸을 읽을기' 지정하기 위한 주소 비트 수=주소 수)
		  dataWidth   = 16,    // 가중치 비트 폭
		  weightFile  = "w_1_15.mif"

- 2️⃣ 내부 메모리 구조
	- 1차원 메모리 배열로, 각 엔트리에 하나의 weight가 들어간다.

			reg [dataWidth-1:0] mem [numWeight-1:0];

			mem[0]   → weight 0
			mem[1]   → weight 1
			...
			mem[n]   → weight n
   
- 3️⃣ Pre-Trained vs Trained 모드

		initial begin
		
		  $readmemb(weightFile, mem);
		
		end

	- Weight를 ROM처럼 사용하여, 시뮬레이션/합성 시작 시, *.mif 파일로 초기화한다.

​

- 4️⃣ 뉴런의 weight 요청

		always @(posedge clk)
		
		begin
		
		  if (ren)
		
		    wout <= mem[radd];
		
		end

	- ren=1 일 때만, wout으로 출력을 갱신한다.
	- 즉, 뉴런 내부에서는 Input data valid (ren=1) → next rising clock에서 wout 수신 → MAC 연산 수행

3. Neuron.v - FC 뉴런 1개를 구현

module neuron #(parameter layerNo=0,neuronNo=0,numWeight=784,dataWidth=16,
sigmoidSize=5,weightIntWidth=1,actType="relu",biasFile="",weightFile="")(
    input           clk,
    input           rst,
    input [dataWidth-1:0]    myinput,
    input           myinputValid,
    input           weightValid,
    input           biasValid,
    input [31:0]    weightValue,
    input [31:0]    biasValue,
    input [31:0]    config_layer_num,
    input [31:0]    config_neuron_num,
    output[dataWidth-1:0]    out,
    output reg      outvalid   
    );
    parameter addressWidth = $clog2(numWeight);
    
    reg         wen; //weight 메모리 write enable
    wire        ren; //weight 메모리 read enable
    reg [addressWidth-1:0] w_addr; //write를 위한 weight 메모리 주소
    reg [addressWidth:0]   r_addr; //read를 위한 weight 메모리 주소
    reg [dataWidth-1:0]  w_in; //메모리에 쓸 weight
    wire [dataWidth-1:0] w_out; //메모리에서 읽은 weight 값
    reg [2*dataWidth-1:0]  mul; 
    reg [2*dataWidth-1:0]  sum;
    reg [2*dataWidth-1:0]  bias; //곱셈/누산/bias 결과는 dataWidth 폭의 두 배
    reg [31:0]    biasReg[0:0]; //pre-trained 모드에서 biasfile에 읽어올 저장용 레지스터
    reg         weight_valid; 
    reg         mult_valid;
    wire        mux_valid;
    reg         sigValid; //valid 파이프라인용 플래그으로, 유효한 결과가 나오는 타이밍 알려줌
    wire [2*dataWidth:0] comboAdd; //mul+sum 결과
    wire [2*dataWidth:0] BiasAdd; //bias+sum 결과
    reg  [dataWidth-1:0] myinputd; //입력을 1클록 지연(파이프라인 정렬)
    reg muxValid_d; //mux_valid의 엣지 검출용
    reg muxValid_f; //''
    reg addr=0; //biasReg 인덱스로 사용하는데, 여기선 상수 0처럼 동작

   //Loading weight values into the momory
    always @(posedge clk)
    begin
        if(rst) begin //reset(초기화)
            w_addr <= {addressWidth{1'b1}};
            wen <=0;
        end
        else if(weightValid & (config_layer_num==layerNo) & (config_neuron_num==neuronNo))
        begin
            w_in <= weightValue;
            w_addr <= w_addr + 1;
            wen <= 1;
        end
        else
            wen <= 0;
    end
	
    assign mux_valid = mult_valid;
    assign comboAdd = mul + sum;
    assign BiasAdd = bias + sum;
    assign ren = myinputValid;
    
	`ifdef pretrained
		initial begin
			$readmemb(biasFile,biasReg);
		end
		always @(posedge clk)
		begin
            bias <= {biasReg[addr][dataWidth-1:0],{dataWidth{1'b0}}};
        end
	`else
		//생략
	`endif   
    
    always @(posedge clk)
    begin
        if(rst|outvalid)
            r_addr <= 0;
        else if(myinputValid)
            r_addr <= r_addr + 1;
    end
    
    always @(posedge clk)
    begin
        mul  <= $signed(myinputd) * $signed(w_out);
    end 
    
    always @(posedge clk)
    begin
        if(rst|outvalid)
            sum <= 0;
        else if((r_addr == numWeight) & muxValid_f)
        begin
            if(!bias[2*dataWidth-1] &!sum[2*dataWidth-1] & BiasAdd[2*dataWidth-1]) begin            begin
                sum[2*dataWidth-1] <= 1'b0;
                sum[2*dataWidth-2:0] <= {2*dataWidth-1{1'b1}};
            end
            else if(bias[2*dataWidth-1] & sum[2*dataWidth-1] &  !BiasAdd[2*dataWidth-1]) begin
                sum[2*dataWidth-1] <= 1'b1;
                sum[2*dataWidth-2:0] <= {2*dataWidth-1{1'b0}};
            end
            else
                sum <= BiasAdd; 
        end
        else if(mux_valid)
        begin
            if(!mul[2*dataWidth-1] & !sum[2*dataWidth-1] & comboAdd[2*dataWidth-1])
            begin
                sum[2*dataWidth-1] <= 1'b0;
                sum[2*dataWidth-2:0] <= {2*dataWidth-1{1'b1}};
            end
            else if(mul[2*dataWidth-1] & sum[2*dataWidth-1] & !comboAdd[2*dataWidth-1])
            begin
                sum[2*dataWidth-1] <= 1'b1;
                sum[2*dataWidth-2:0] <= {2*dataWidth-1{1'b0}};
            end
            else
                sum <= comboAdd; 
        end
    end
    
    always @(posedge clk)
    begin
        myinputd <= myinput;
        weight_valid <= myinputValid;
        mult_valid <= weight_valid;
        sigValid <= ((r_addr == numWeight) & muxValid_f) ? 1'b1 : 1'b0;
        outvalid <= sigValid;
        muxValid_d <= mux_valid;
        muxValid_f <= !mux_valid & muxValid_d;
    end
    
    //Instantiation of Memory for Weights
    Weight_Memory #(.numWeight(numWeight),.neuronNo(neuronNo),.layerNo(layerNo),
.addressWidth(addressWidth),.dataWidth(dataWidth),.weightFile(weightFile)) WM(
        .clk(clk),
        .wen(wen),
        .ren(ren),
        .wadd(w_addr),
        .radd(r_addr[addressWidth-1:0]),
        .win(w_in),
        .wout(w_out)
    );
    
	generate
		if(actType == "sigmoid")
		begin:siginst
		//Instantiation of ROM for sigmoid
			Sig_ROM #(.inWidth(sigmoidSize),.dataWidth(dataWidth)) s1(
			.clk(clk),
			.x(sum[2*dataWidth-1-:sigmoidSize]),
			.out(out)
		);
		end
		else
		begin:ReLUinst
			ReLU #(.dataWidth(dataWidth),.weightIntWidth(weightIntWidth)) s1 (
			.clk(clk),
			.x(sum),
			.out(out)
		);
		end
	endgenerate
endmodule
    
1. 파라미터 / 입/출력 구조

parameter

     layerNo=0,                          //몇 번째 레이어인가?

     neuronNo=0,                      //해당 레이어에서 몇 번째 뉴런인가?

     numWeight=784,               //이 뉴런이 곱해야 하는 입력 개수

     dataWidth=16, sigmoidSize=5, weightIntWidth=1, actType="relu", biasFile="", weightFile=""

​

    input           clk,

    input           rst,

    input [dataWidth-1:0]    myinput,              //입력 데이터 스트림

    input           myinputValid,                         //유효 신호

    input           weightValid,                           

    input           biasValid,

    input [31:0]    weightValue,

    input [31:0]    biasValue,

    input [31:0]    config_layer_num,             //모든 뉴런이 같은 weightValue/biasValue 버스 공유

    input [31:0]    config_neuron_num,         //axi 버스가 특정 뉴런을 선택하기 위해 사용하는 설정용

    output[dataWidth-1:0]    out,                    //주소 입력이며, weight/bias 로딩을 위한 HW 셀렉터 역할

    output reg      outvalid                              //config_layer/neuron_num input이 필요하다.

​

2. 주소폭/레지스터/와이어 선언

parameter addressWidth = $clog2(numWeight);

weight 메모리 주소에 필요한 비트 수를 계산해야 한다. 예컨대, numWeight=784라면, $clog2(784)=10으로, 784개의 weight를 넣기 위해 1024개의 entry에 대한 메모리 주소를 만든다.

​

3. weight 값을 메모리에 write하는 always 구문 중 일부

        else if(weightValid & (config_layer_num==layerNo) & (config_neuron_num==neuronNo))

        begin

            w_in <= weightValue;

            w_addr <= w_addr + 1;

            wen <= 1;

        end

weightValid=1이고, 외부가 지정한 config_layer/neuron_num이 해당 뉴런의 layerNo/neuronNo와 같으면, w_in에 weightValue를 넣고, w_addr 증가, wen=1로 메모리에 write

​

4. pre-trained 모드에서 Bias

initial begin

$readmemb(biasFile,biasReg);

end

always @(posedge clk)

begin

            bias <= {biasReg[addr][dataWidth-1:0],{dataWidth{1'b0}}};

        end

include.v에서 입력 비트 폭을 8bit로 정의하였고, 이때, Q.format의 정수부를 4bit로 정의하였다. 이에 대해 곱셈 결과 정수부는 MSB 8bit, 실수부는 LSB 8bit이므로, 정수인 bias를 더해주기 위해서는 8bit만큼 shift left해줄 필요가 있다. {biasReg[addr][dataWidth-1:0],{dataWidth{1'b0}}};의 의미는 곧, {biasValue[7:0],8'b0}이므로 biasValue << 8와 같은 효과를 준다. 예컨대, biasValue=0000_0011이라면, 실제 bias=0000_0011_0000_0000이다.

​

5. r_addr (read address) 카운터

always @(posedge clk)

begin

  if(rst|outvalid)

    r_addr <= 0;

  else if(myinputValid)

    r_addr <= r_addr + 1;

end

rst(초기화) 또는 outvalid(출력완료)되면, 다음 입력 벡터 처리를 위해 r_addr=0으로 초기화하고, myinputValid(입력)가 유효할 때마다 weight 주소를 1씩 증가한다.

​

6. Mul pipeline

always @(posedge clk)

begin

  mul <= $signed(myinputd) * $signed(w_out);

end

myinputd(지연 입력)과 w_out(메모리의 weight)를 곱한다.

​

7. sum + saturation + bias add

always @(posedge clk)

begin

  if(rst|outvalid)

    sum <= 0;

reset 또는 outvalid(한 벡터 처리 완료)일 때 sum=0으로 초기화

  else if((r_addr == numWeight) & muxValid_f) begin

    ...

    sum <= BiasAdd;

  end

(r_addr == numWeight)은 r_addr(입력 개수 카운터)이 numWeight에 도달한 상황과, 이때 마지막 mul이 실제로 sum에 반영된 '직후'를 의미한 muxValid_f=1은 모든 입력을 다 처리한 상황이다. 따라서 마지막으로 bias를 더할 차례에 해당한다. begin-end 구문을 자세히 살펴보자.

            if(!bias[2*dataWidth-1] &!sum[2*dataWidth-1] & BiasAdd[2*dataWidth-1]) begin                                begin

                sum[2*dataWidth-1] <= 1'b0;

                sum[2*dataWidth-2:0] <= {2*dataWidth-1{1'b1}};

            end

            else if(bias[2*dataWidth-1] & sum[2*dataWidth-1] &  !BiasAdd[2*dataWidth-1]) begin

                sum[2*dataWidth-1] <= 1'b1;

                sum[2*dataWidth-2:0] <= {2*dataWidth-1{1'b0}};

            end

            else

                sum <= BiasAdd;

end

if(!bias[MSB] & !sum[MSB] & BiasAdd[MSB]) : bias와 sum 모두 양수인데, assign BiasAdd = bias + sum;에 대한 BiasAdd는 2's complement에 의해 음수가 되면서 overflow된 상황이다. 이때는 sum을 MAX로 saturation시킨다.

else if(bias[2*dataWidth-1] & sum[2*dataWidth-1] &  !BiasAdd[2*dataWidth-1]) : 이는 반대로 bias와 sum 모두 음수인데, 이 두 값 모두 음수여서 BiasAdd가 양수로 overflow된 상황이므로, 이때는 sum을 MIN로 saturation시킨다.

else : 만약 위와 같은 두 상황이 모두 발생하지 않은, overflow가 발생하지 않은 상황이라면, 정상적으로 더한 값을 sum에 사용한다.

​

이어서 그 다음 구문을 확인해보자.

  else if(mux_valid)

  begin

    ...

    sum <= comboAdd;

  end

end

이는 단순 일반 단계에서의 mul + sum 수행인데, 이 역시 위에서 알아보았던 overflow 체크 후 필요한 상황에 대해서는 sum을 saturation시킬 수 있도록 한다.

​

8. 입력/Valid Pipeline 정렬 + outvalid 생성

    always @(posedge clk)

    begin

        myinputd <= myinput;  //입력 1클록 지연

        weight_valid <= myinputValid; //myinputValid 1클록 지연

        mult_valid <= weight_valid; //weight_valid 1클록 더 지연

        sigValid <= ((r_addr == numWeight) & muxValid_f) ? 1'b1 : 1'b0; //마지막 입력 처리 완료 시 1

        outvalid <= sigValid; //sigValid를 바로 다음 줄에서 outvalid로 보냄

        muxValid_d <= mux_valid; //일반 mul + sum에 대한 것

        muxValid_f <= !mux_valid & muxValid_d; //마지막 누산이 끝나는 순간에 대한 것

    end

​

*입력은 왜 1 클록 지연이 필요한가?

ren = myinputValid로서, inputvalid가 들어온 사이클에 weight가 바로 나오지 않고,(if (ren) wout <= mem[radd];), 다음 클록에 w_out이 유효하므로, 1 클록 지연 필요 

*mult_valid는 왜 2 클록 필요한가?

Cycle N : myinputValid=1 //weight memory에 read 요청(ren=1)

Cycle N+1 : w_out, myinputd 값 유효, 이 둘로 mul 계산이 진행되지만, mul은 레지스터이므로 결과는 N+1의 엣지에서 저장된다.(mul  <= $signed(myinputd) * $signed(w_out);)

Cycle N+2 : 이제서야 레지스터된 mul이 안정적으로 존재한다.

​

9. Weight_Memory 인스턴스 및 Activation 선택

Weight_Memory #(..) WM(..);

generate

if(actType == "sigmoid") begin:siginst

//Instantiation of ROM for sigmoid

Sig_ROM #(..) s1(..);

end

else

begin:ReLUinst

ReLU #(..) s1(..);

end

endgenerate

​

4. MaxFinder.v

module maxFinder #(parameter numInput=10,parameter inputWidth=16)(
input           i_clk,
input [(numInput*inputWidth)-1:0]   i_data,
input           i_valid,
output reg [31:0]o_data,
output  reg     o_data_valid
);

reg [inputWidth-1:0] maxValue;
reg [(numInput*inputWidth)-1:0] inDataBuffer;
integer counter;

always @(posedge i_clk)
begin
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
1. 파라미터 / 입/출력 구조

parameter 

    numInput=10,     //입력 개수

    inputWidth=16   //입력 비트 폭

​

input           i_clk,

input [(numInput*inputWidth)-1:0]   i_data, //numInput개 입력을 한 번에 묶어 들어오는 버스

input           i_valid,                                         //i_data 묶음의 유효함 트리거

output reg [31:0]o_data,                               //최댓값을 가진 원소의 인덱스(0~numInput-1)를 출력

output  reg     o_data_valid                          //결과 유효 펄스

​

2. 레지스터 선언

reg [inputWidth-1:0] maxValue;                                  //현재까지 발견한 최댓값

reg [(numInput*inputWidth)-1:0] inDataBuffer;         //i_valid 순간의 i_data를 저장해 두는 버퍼

integer counter;                                                           //몇 번째 원소를 검사 중인지 나타내는 카운터

​

3. 메인 블록

always @(posedge i_clk)

begin

    o_data_valid <= 1'b0;

매 클록마다 일단 o_data_valid를 0으로 내려. 결과가 나올 때만 아래에서 1로 올려서 1 클록짜리 valid 펄스를 만들겠다는 구조.

​

    if(i_valid)                                                   //새로운 계산 시작

    begin

        maxValue <= i_data[inputWidth-1:0]; //하위 16비트 즉, 0번째 원소 값을 maxValue에 대입

        counter <= 1;                                        //1번째 원소 비교하기 위한 카운터

        inDataBuffer <= i_data;                       //입력 묶음을 내부에 저장(비교를 inDataBuffer로 함)

        o_data <= 0;                                         //argmax 인덱스 0으로 시작

    end

​

    else if(counter == numInput)                  //모든 원소 검사 완료 시

    begin

        counter <= 0;                                       //카운터를 idle 상태로 복귀

        o_data_valid <= 1'b1;                         //결과 유효를 1로

    end

​

    else if(counter != 0)                                //검사가 진행 중일때

    begin

        counter <= counter + 1;                     //다음 인덱스 준비

        if(inDataBuffer[counter*inputWidth+:inputWidth] > maxValue)

        begin

            maxValue <= inDataBuffer[counter*inputWidth+:inputWidth];  //maxValue 업데이트

            o_data <= counter; //o_data를 현재 인덱스로 업데이트

        end

    end

end

inDataBuffer[counter*inputWidth+:inputWidth] :

counter=1 → [16 +: 16] = bits [31:16] = 1번 원소

counter=2 → bits [47:32] = 2번 원소

...

counter=0은 원래 초기화에서 이미 처리했기 때문에 여기서는 1부터 진행

5. Layer1.v (2, 3 생략)

`timescale 1 ns / 1 ps
module Layer_1 #(parameter NN = 30,numWeight=784,dataWidth=16,layerNum=1,
                 sigmoidSize=10,weightIntWidth=4,actType="relu")
    (
    input           clk,
    input           rst,
    input           weightValid,
    input           biasValid,
    input [31:0]    weightValue,
    input [31:0]    biasValue,
    input [31:0]    config_layer_num,
    input [31:0]    config_neuron_num,
    input           x_valid,
    input [dataWidth-1:0]    x_in,
    output [NN-1:0]     o_valid,
    output [NN*dataWidth-1:0]  x_out
    );
neuron #(.numWeight(numWeight),.layerNo(layerNum),.neuronNo(0),.dataWidth(dataWidth),
         .sigmoidSize(sigmoidSize),.weightIntWidth(weightIntWidth),.actType(actType),
          .weightFile("w_1_0.mif"),.biasFile("b_1_0.mif"))n_0(
        .clk(clk),
        .rst(rst),
        .myinput(x_in),
        .weightValid(weightValid),
        .biasValid(biasValid),
        .weightValue(weightValue),
        .biasValue(biasValue),
        .config_layer_num(config_layer_num),
        .config_neuron_num(config_neuron_num),
        .myinputValid(x_valid),
        .out(x_out[0*dataWidth+:dataWidth]),
        .outvalid(o_valid[0])
        );
.
.
.
neuron #(.numWeight(numWeight),.layerNo(layerNum),.neuronNo(29),.dataWidth(dataWidth),
         .sigmoidSize(sigmoidSize),.weightIntWidth(weightIntWidth),.actType(actType),
         .weightFile("w_1_29.mif"),.biasFile("b_1_29.mif"))n_29(
        .clk(clk),
        .rst(rst),
        .myinput(x_in),
        .weightValid(weightValid),
        .biasValid(biasValid),
        .weightValue(weightValue),
        .biasValue(biasValue),
        .config_layer_num(config_layer_num),
        .config_neuron_num(config_neuron_num),
        .myinputValid(x_valid),
        .out(x_out[29*dataWidth+:dataWidth]),
        .outvalid(o_valid[29])
        );
endmodule
6. ZyNet.v

module zyNet #(...) (...);

.
.
.

localparam IDLE = 'd0,
           SEND = 'd1;
wire [`numNeuronLayer1-1:0] o1_valid;
wire [`numNeuronLayer1*`dataWidth-1:0] x1_out;
reg [`numNeuronLayer1*`dataWidth-1:0] holdData_1;
reg [`dataWidth-1:0] out_data_1;
reg data_out_valid_1;

Layer_1 #(...) l1(...);

//State machine for data pipelining
reg       state_1;
integer   count_1;
always @(posedge s_axi_aclk)
begin
    if(reset)
    begin
        state_1 <= IDLE;
        count_1 <= 0;
        data_out_valid_1 <=0;
    end
    else
    begin
        case(state_1)
            IDLE: begin
                count_1 <= 0;
                data_out_valid_1 <=0;
                if (o1_valid[0] == 1'b1)
                begin
                    holdData_1 <= x1_out;
                    state_1 <= SEND;
                end
            end
            SEND: begin
                out_data_1 <= holdData_1[`dataWidth-1:0];
                holdData_1 <= holdData_1>>`dataWidth;
                count_1 <= count_1 +1;
                data_out_valid_1 <= 1;
                if (count_1 == `numNeuronLayer1)
                begin
                    state_1 <= IDLE;
                    data_out_valid_1 <= 0;
                end
            end
        endcase
    end
end
.
.
.
1. FSM : IDLE 상태

            IDLE: begin

                count_1 <= 0;

                data_out_valid_1 <=0;

                if (o1_valid[0] == 1'b1)

                begin

                    holdData_1 <= x1_out;

                    state_1 <= SEND;

                end

            end

매 클록 count_1=0, valid=0, o1_valid[0]==1이 되는 순간, holdData_1 <= x1_out로 병렬 출력 벡터를 버퍼에 캡처해서 상태를 SEND로 전환

2. FSM : SEND 상태

SEND: begin

    out_data_1 <= holdData_1[`dataWidth-1:0];

    holdData_1 <= holdData_1 >> `dataWidth;

    count_1 <= count_1 +1;

    data_out_valid_1 <= 1;

    if (count_1 == `numNeuronLayer1)

    begin

        state_1 <= IDLE;

        data_out_valid_1 <= 0;

    end

end

out_data_1 <= holdData_1[dataWidth-1:0] : holdData의 LSB쪽 1개 원소를 꺼내서 출력

holdData_1 <= holdData_1 >> dataWidth : 한 원소만큼 오른쪽 shift, 다음 클록에는 “다음 원소”가 LSB로 내려오게 됨

count_1 <= count_1 + 1 : 몇 개 보냈는지 증가

data_out_valid_1 <= 1 : SEND 동안은 매 클록 유효

즉, x1_out의 구성 순서가 x1_out[dataWidth-1:0] = 0번 뉴런 출력, x1_out[2*dataWidth-1:dataWidth] = 1번 뉴런 출력 ... 이런 식으로 “LSB부터 뉴런0,1,2…” 라는 가정
