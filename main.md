## Classes
- Virtual classes (base classes to be extended)
- Pure virtual functions get overriden
- Non-virtual functions call actual object not inherited one
Syntax:
```
class Packet;
  rand bit [7:0] addr;
  rand bit [3:0] data;
  rand bit burst_mode;

  // optional constructor if non-default
  function new(...);
    addr = ..
  endfunction: new

  function void checkSum(int a, int b, int c);
    return (a + b == c);
  endfunction: checkSum

  function void pre_randomize();
    if (burst_mode) begin
      addr.rand_mode(0); // turns off randomization
      addr = 8'h0;
    end else
      addr.rand_mode(1)
  endfunction: pre_randomize
```

## Interfaces and Modports
```
interface bus_if(input logic clk);
  logic valid, ready;
  logic [7:0] addr;
  logic [3:0] data;

  clocking drv_cb @(posedge clK);
    default input #1step output #1;
    input ready;
    output data, valid, addr;
  endclocking
endinterface: bus_if
```

FSM code 
```
module fsm (
  input  logic clk, rst_n,
  input  logic a, b, c
  output logic d, e, f
);
  typedef logic [1:0] {
    A = 2'd0, B = 2'd1, C = 2'd2, D = 2'd3 } state_t;
  state_t state, nextState;

  always_comb begin
    nextState = state;
    case (state)
      A: begin
        if (a) nextState = D;
      end
  end

  always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)
      state <= A;
    else
      state <= nextState;
  end
```
