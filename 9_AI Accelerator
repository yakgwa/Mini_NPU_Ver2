## [AI Accelerator 기초] - Systolic Array와 FSM 구현1 (PE, 1D PE Chain) 
AI 가속기 하드웨어를 이해하기 위해서는, 연산 구조 자체를 어떻게 설계하고 제어하는지를 단계적으로 뜯어보는 과정이 필요하다. 이전까지는 Google TPU, Gemmini 논문을 살펴보며 Systolic Array 기반 가속기의 철학과 데이터플로우(OS/WS/IS)를 정리했고, 동시에 ZyNet으로 자동 생성된 RTL DUT와 Testbench를 분석하면서 실제 뉴럴넷 하드웨어 구조를 확인했다. 그 과정에서 ZyNet 구조가 가지는 PPA 관점의 한계와, 데이터 재사용 및 제어 방식 측면에서의 개선 여지도 함께 정리하였다. 이러한 분석을 바탕으로 최초의 개선 방향 모델을 개념적으로 제시했다. 다만, 곧바로 최종 형태의 가속기를 설계하기보다는, 이번 글부터는 실제로 가속기를 설계하는 것이기 때문에 보다 기초적으로 접근함으로써 단일 PE(Processing Element)부터 시작하여 1D 1×4 PE Chain으로 확장하고, 최종적으로 OS(Output-Stationary) 기반 4×4 Systolic Array로 단계적으로 확장하는 흐름으로 설계를 진행한다. 따라서 이번 단계에서의 목표는 “잘 동작하는 구조를 만드는 것”이다. 따라서 타이밍 최적화, 면적 절감, 전력 분석과 같은 요소는 배제하고, 기능 검증 수준에서만 설계와 검증을 수행한다. 또한, 복잡한 스케줄러 대신 FSM 기반의 단순 컨트롤러, 그리고 ReLU를 결합하여, 단순 연산이 하드웨어 상에서 어떻게 흘러가고 제어되는지에 집중한다.

*참고 : tb는 항상 

step1) interface definition ▶ step2) Constrained Random Transaction ▶ Step 3) Top Module Instantiation ▶ Step 4) Functional Coverage ▶ Step 5) Main 
순서로 작성하였다
