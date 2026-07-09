## Classes (and constraints)
- Virtual classes (base classes to be extended)
- Pure virtual functions get overriden
- Non-virtual functions call actual object not inherited one
Syntax:
```
class Packet;
  rand bit [7:0] addr;
  rand bit [3:0] data;
  rand bit burst_mode;

  constraint c_addr { addr[0] == 0; }
  constraint c_order { solve addr before data; }

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

To instantiate:
```
Packet pkt = new();
if (!pkt.randomize() with { data == 4'hF; }) $error("Randomization failed");
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

  clocking mon_cb @(negedge clk);
    default input #1step output #1;
    input ready, data, valid, addr;
  endclocking

endinterface: bus_if
```

## Data structures
1) Queue
```
int q[$]; // int q[$:5] bounded at 6 elements
q.push_back(1);
q.delete(0)
```

2) Fixed array
```
int arr[];
arr = new[4]; // size 4
```

3) Associative array
```
int mem[string]; // string as keys, int as values
mem["test"] = 5;
if (mem.exists("test)) // returns 1
```

4) Structs (packed and unpacked)
```
typedef struct packed {
  bit [7:0] addr, bit [7:0] data } pkt_t; // can do wire[15:0] w = b; because packed
  
typedef enum logic {A, B} state_t;
```

## Concurrency
1) Fork..join (waits for all)
2) Fork..join_any (waits for one)
3) Fork..join_none (waits for none)

### Semaphores
```
semaphore sem = new(1); // only 1 key

sem.get(1);
drive_bus(0;
sem.put(1);
```

### Mailbox
```
mailbox #(packet) mbx = new(0); // unbounded if 0, bounded if > 0

// Producer does this
mbx.put(pkt); // blocks if full

// Consumer does this
packet p;
mbx.get(p); // blocks if empty
```

### Event
```
event done;

-> done; // triggers event
wait(done.triggered); // waits for triggered done in same timestep
```

## FIFO
```
module fifo #(
  parameter DEPTH=8,
  parameter DW=32,
  localparam PW=$clog2(DEPTH)
)(
  input  logic clk, rst_n,
  input  logic re, we,
  output logic full, empty,
  input  logic [DW-1:0] w_data,
  output logic [DW-1:0] r_data
);
  logic [DW-1:0] Q [DEPTH];
  logic [PW-1:0] r_ptr, w_ptr;
  logic [PW:0] capacity;

  logic do_write, do_read;

  assign full = (capacity == DEPTH);
  assign empty = (capacity == '0);
  assign do_write = (we && ~full);
  assign do_read = (re && ~empty);

  always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n) begin
	capacity <= '0;
	r_ptr <= '0;
        w_ptr <= '0;
    end else begin
	if (do_write) begin
	  w_ptr <= w_ptr + 1;
	  Q[w_ptr] <= w_data;
	end
    	if (do_read) begin
	  r_ptr <= r_ptr + 1;
	  r_data <= Q[r_ptr];
        end
        case ({do_write, do_read})
	 2'b10: capacity <= capacity + 1;
         2'b01: capacity <= capacity - 1;
         default: ;
    end
  end
endmodule: fifo
```

For asynchronous FIFO: add gray-code pointers and 2 FF synchronizers across clock boundaries. <br>
gray code: grey = (bin >> 1)*bin;

## Round-robin arbiter
```
module rr_arbiter #(parameter int N=4) (
  input  logic clk, rst_n,
  input  logic [N-1:0] req,
  output logic [N-1:0] grant
);

  logic [$clog2(N)-1:0] ptr;
  int idx;

  always_comb begin
    grant = '0;
    if (|req) begin
      for (int i=0; i<N; i++) begin
        idx = (ptr + i) % N;
        if (req[idx]) begin
          grant[idx] = 1'b1; break;
        end
      end
    end
  end

  always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)
      ptr <= '0;
    else if (|grant)
      ptr <= (1 + idx) % N;
endmodule: rr_arbiter
```

## Valid-ready handshake
```
module skid_stage #(parameter int W=8)
( input logic clk, rst_n,
  input  logic         in_valid,  output logic in_ready,
  input  logic [W-1:0] in_data,
  output logic         out_valid, input  logic out_ready,
  output logic [W-1:0] out_data );

  assign in_ready = out_ready || !out_valid; // draining or empty
  
  always_ff @(posedge clk, negedge rst_n) begin
  	if (~rst_n) begin
		out_valid <= '0;
		out_data <= '0;
	end
	else if (in_valid && in_ready) begin
		out_data <= in_data;
		out_valid <= 1'b1;
	end
	else if (out_valid && out_ready) begin
		out_data <= '0; // not required
		out_valid <= '0;
	end
  end
endmodule: skid_stage
```
## FSM 
```
module fsm (
  input  logic clk, rst_n,
  input  logic a, b, c
  output logic d, e, f
);
  typedef enum logic [1:0] {
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

## SVA
Immediate and concurrent
### Concurrent assertions
Example:
```
property p_req_ack;
	@(posedge clk) disable iff (~rst_n) $rose(req) |-> ##[1:5] ack;
endproperty

a_req_ack: assert property (p_req_ack) else $error();
```

```
// operators
|->; |=>

// Delay
##1; ##[1:3];

// Repetition
[*2] // consecutive
[=2] // non-consecutive

// example
assert property ((@(posedge clk) req[=2] |-> ack)); // ack is true on 2nd non-consecutive req

// System functions
$past(); $fell(); $stable();
```

## Coverage 
Syntax:
```
covergroup <covergroup_name> (<input signal/class>) <@(event)> // event is optional, input is optional if covergroup defined inside class
	cp_op: coverpoint <signal> {
		bins arith[] = {[0:3]}; // creates 1 bin each
		bins logic_op = {[4:7]}; // creates 1 total bin
		bins mid[4] = {[1:254]}; // split into 4
		illegal_bins bad = {[8:15]}; //  error if hit
	}

	//cross
	x_op_zero: cross <signalA>, <signalB> { // can also be coverpoints
		ignore_bins n = binsof(cp_op) intersect {[4:7]};
	}
```
		
## Verification plan
1) Datapath correctness: test using simulation, randomized stimulus and scoreboards using a model.
2) Control correctness: test using assertions
3) Protocol correctness: test using assertions
4) Corner cases: directed tests
5) Collect functional coverage
6) Use formal verification to verify control correctness, no illegal states and FSM behavior

> Control correctness = "Is the design's internal behavior/state machine doing the right thing?" <br>
> Protocol correctness = "Is the design obeying the rules of an interface/communication contract?"
	
