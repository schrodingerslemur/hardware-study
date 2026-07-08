1) FIFO code
```
module fifo #(
  parameter DEPTH=8 // power of two,
  parameter WIDTH=8
)(
  input  logic clk, rst,
  input  logic we, re,
  input  logic [DEPTH-1:0] w_data
  output logic [DEPTH-1:0] r_data
);
  logic [WIDTH-1:0] q [DEPTH-1:0];
  logic [$clog2(DEPTH)-1:0] r_ptr, w_ptr;
  logic [$clog2(DEPTH):0] capacity;

  assign full = (capacity == DEPTH);
  assign empty = (capacity == '0);

  always_ff @(posedge clk, posedge rst) begin
    if (rst) begin
      r_data <= '0
      r_ptr <= '0;
      w_ptr <= '0;
      capacity <= '0;
    end
    else if (re && we) begin
      r_data <= q[r_ptr];
      r_ptr <= r_ptr + 1;
      q[w_ptr] <= w_data;
      w_ptr <= w_ptr + 1;
    end
    else if (re && ~empty) begin
      r_data <= q[r_ptr];
      r_ptr <= r_ptr + 1;
      capacity <= capacity - 1;
    end
    else if (we && ~full) begin
      q[w_ptr] <= w_data;
      w_ptr <= w_ptr + 1;
      capacity <= capacity + 1;
    end
```
