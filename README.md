# IEEE-754-Multiplier-Project
Designed and implemented IEEE-754 compliant floating-point multiplier in HDL

Design code:
`timescale 1ns / 1ps

module mult_comb(
    
    output logic [31:0] c,
    input logic [31:0] a,
    input logic [31:0] b
);
logic sign_prod;
logic inf_flag;
logic nan_flag;
logic exp_add;
logic [8:0] expon_prod_pre;
logic [8:0] expon_prod_post;
logic [47:0] mult_res;
logic [23:0] mant_prod_post;
logic [23:0] layer_0;
logic [11:0] layer_1;
logic [5:0]  layer_2;
logic [2:0]  layer_3;
logic [1:0]  layer_4;
logic [5:0]  position [0:6];
logic [47:0] inter_mantissa;

always_comb begin
    expon_prod_pre = a[30:23] + b[30:23];
    sign_prod = a[31] ^ b[31];
    nan_flag = ((&a[30:23]) && (|a[22:0])) || ((&b[30:23]) && (|b[22:0]));
    inf_flag = ((&a[30:23]) && (~|a[22:0])) || ((&b[30:23]) && (~|b[22:0]));
    exp_add = (~|a[30:23])||(~|b[30:23]);
    position[0] = 47;
    if (mult_res[47:24]) begin
        layer_0 = mult_res[47:24];
        position[1] = position[0];
    end else begin
        layer_0 = mult_res[23:0];
        position[1] = position[0] - 24;
    end
    if (layer_0[23:12]) begin
        layer_1 = layer_0[23:12];
        position[2] = position[1];
    end else begin
        layer_1 = layer_0[11:0];
        position[2] = position[1] - 12;
    end
    if (layer_1[11:5]) begin
        layer_2 = layer_1[11:6];
        position[3] = position[2];
    end else begin
        layer_2 = layer_1[5:0];
        position[3] = position[2] - 6;
    end
    if (layer_2[5:3]) begin
        layer_3 = layer_2[5:3];
        position[4] = position[3];
    end else begin
        layer_3 = layer_2[2:0];
        position[4] = position[3] - 3;
    end
    if (layer_3[2:1]) begin
        layer_4 = layer_3[2:1];
        position[5] = position[4];
    end else begin
        layer_4 = {layer_3[0],1'b0};
        position[5] = position[4] - 2;
    end
    if (layer_4[1]) position[6] = position[5];
    else position[6] = position[5] - 1;
    if (expon_prod_pre < 47 - position[6]) inter_mantissa = mult_res << expon_prod_pre;
    else inter_mantissa = mult_res << (47 - position[6]);
    if (expon_prod_pre < 47 - position[6]) expon_prod_post = 0;
    else expon_prod_post = expon_prod_pre - 47 + position[6];
    if (expon_prod_post < 127) inter_mantissa = inter_mantissa >> (127 - expon_prod_post);
    if (expon_prod_post < 127) expon_prod_post = 0;
    else expon_prod_post = expon_prod_post - 127;
    mant_prod_post = inter_mantissa[47:24] + inter_mantissa[23];
    if (!mant_prod_post) expon_prod_post = 0;
    if (expon_prod_post > 253) expon_prod_post = 255;
    if (expon_prod_post == 255) mant_prod_post = {1'b0, {23{1'b0}}};
    c = {sign_prod, expon_prod_post[7:0] + mant_prod_post[23] + exp_add, mant_prod_post[22:0]};
    if(nan_flag) c[30:0] = {8'd255, 23'd1};
    if(inf_flag) c[30:0] = {8'd255, 23'd0};
end

karatsuba_comb #(.WIDTH(24)) mult (
    .a({(|a[30:23]), a[22:0]}),
    .b({(|b[30:23]), b[22:0]}),
    .c(mult_res)
);

endmodule




module karatsuba_comb#(
    parameter WIDTH = 24
)(
    input logic [WIDTH - 1 : 0] a,
    input logic [WIDTH - 1 : 0] b,
    output logic [2*WIDTH - 1 : 0] c
);
logic [WIDTH - 1 : 0] c11;
logic [WIDTH - 1 : 0] c12;
logic [WIDTH - 1 : 0] c21;
logic [WIDTH - 1 : 0] c22;
logic [2*WIDTH - 1 : 0] f0;
logic [2*WIDTH - 1 : 0] f1;
logic [2*WIDTH - 1 : 0] f2;

mult #(.WIDTH(WIDTH/2)) m11 (.a(a[0 +: WIDTH/2]), .b(b[0 +: WIDTH/2]), .c(c11));
mult #(.WIDTH(WIDTH/2)) m12 (.a(a[0 +: WIDTH/2]), .b(b[WIDTH/2 +: WIDTH/2]), .c(c12));
mult #(.WIDTH(WIDTH/2)) m21 (.a(a[WIDTH/2 +: WIDTH/2]), .b(b[0 +: WIDTH/2]), .c(c21));
mult #(.WIDTH(WIDTH/2)) m22 (.a(a[WIDTH/2 +: WIDTH/2]), .b(b[WIDTH/2 +: WIDTH/2]), .c(c22));

always_comb begin
    f0 = {c22, c11};
    f1 = c12 << WIDTH/2;
    f2 = c21 << WIDTH/2;
    c = f0 + f1 + f2;
end

endmodule



module mult#(
    parameter WIDTH = 12
)(
    
    output logic [2*WIDTH - 1 : 0] c,
    input logic [WIDTH - 1 : 0] a,
    input logic [WIDTH - 1 : 0] b
);
localparam LAYERS = $clog2(WIDTH);
logic [2*WIDTH - 1 : 0] intermediate [2**LAYERS][LAYERS];
integer i, j;

always_comb begin
    for (i = 0; i < 2**LAYERS; i = i + 1) intermediate[i][0] = 0;
    for (i = 0; i < WIDTH; i = i + 1) intermediate[i][0] = (a & {WIDTH{b[i]}}) << i;
    for (i = 0; i < LAYERS-1; i = i + 1) begin
        for (j = 0; j < 2**LAYERS; j = j + 2**(i+1)) begin
            intermediate[j][i+1] = intermediate[j][i] + intermediate[j+2**i][i];
        end
    end
    c = intermediate[0][LAYERS-1] + intermediate[2**(LAYERS-1)][LAYERS-1];
end

endmodule

