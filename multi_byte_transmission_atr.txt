// Code your design here
`timescale 1ns / 1ps

`define N 3
module i2c_atr(sda_in,scl_in,en,ready,translation_byte,sda_out,scl_out);

parameter IDLE=9'b0_0000_0001;
parameter READ_ADDR=9'b0_0000_0010;
parameter ACK1=9'b0_0000_0100;
parameter ADDRESS_READ=9'b0_0000_1000;
parameter ACK2=9'b0_0001_0000;
parameter WRITE_DATA=9'b0_0010_0000;
parameter ACK3=9'b0_0100_0000;
parameter READ=9'b0_1000_0000;
parameter ACK4=9'b1_0000_0000;
(* dont_touch = "true" *) reg [8:0]state_0=IDLE,state_1=WRITE_DATA,prev_state=IDLE,state_temp;
always@(state_1) begin
state_temp<=state_1;
prev_state<=state_temp;
end

//parameter `ports=3;
inout sda_in;
inout scl_in;
inout [`N-1:0] sda_out,scl_out;
(* CLOCK_DEDICATED_ROUTE = "FALSE", IS_CLOCK = "FALSE" *) output [`N-1:0]ready;
input [`N-1:0]en;
input [6:0]translation_byte;

reg read=0;
wire scl_o=scl_in;

integer count=0,count_1;
wire [`N-1:0] enables;
assign enables=en;

reg start=0,stop=0,r_st=0;
wire flag=start?1'b1:1'b0;

reg start_bit=0;
reg r_st_bit=0;
always@(negedge sda_in)
begin
    if(scl_in==1 && state_0==IDLE && (state_1==IDLE || state_1==WRITE_DATA ||state_1==READ) && flag==0) begin
//           $display("DESIGN: %0t: negedge_start",$time);
      start_bit<=1;
    end
    else
      start_bit<=0;
      
      if(scl_in==1 && state_0==IDLE && flag==1) begin
      r_st_bit<=1;
      end
      else
      r_st_bit<=0;
  end

wire read_stop_bit=(state_1==ACK4 && sda_in==1 && scl_in==1)?1'b1:1'b0;
reg read_stop=0;
reg read_bit=0;
integer k=0,number_1;
(* dont_touch = "true" *) reg[31:0] number=0;
always@(*) begin
if(start_bit) begin
number<=number_1;
start<=1;
state_0<=READ_ADDR;
read_stop<=1'b0;
end
else if(state_1==ACK1)
state_0<=IDLE;
else if(r_st_bit)
r_st<=1;
else if(read_stop_bit)
read_stop<=1;
else if(stop) begin
start<=0;
r_st<=0;
end

if(read && read_stop==0 && start)
read_bit<=1;
else if(read_stop==1)
read_bit<=0;
end



always@(posedge sda_in) begin
if(scl_in==1 & sda_in==1 && ((prev_state==ACK3 && state_1==WRITE_DATA) || (state_1==READ && read_stop)) && count_1%8==0)
stop<=1;
else
stop<=0;
end

wire [7:0]buffer[`N-1:0];
reg [7:0]buffer_1=0;
reg xor_flag=0,xor_flag_1=0;

always@(*) begin
if(state_1==WRITE_DATA && r_st==1 && count_1<7 && scl_in==0) begin
xor_flag_1<=1;
buffer_1<=buffer[number];
end
else if(state_1==WRITE_DATA && count_1==8) begin
xor_flag_1<=0;
end
end



genvar j;
generate
for(j=0;j<`N;j=j+1) begin
assign buffer[j][6:0]=en[j]?translation_byte:0;
assign ready[j]=en[j];
end
endgenerate

always@(*) begin
for(k=0;k<`N;k=k+1) begin
if(en[k])
number_1=k;
end
end

reg ACK=0;


genvar i;
generate
for(i=0;i<`N;i=i+1) begin
// assign sda_out[i]=((en[i] && count==0 && xor_flag==0) || (en[i]==1 && xor_flag==0 && ACK==0))?sda_in:(xor_flag && count!=0 && en[i])?sda_in^buffer[number][7-count]:1'bz;
assign sda_out[i]=((en[i] && (xor_flag==0 && xor_flag_1==0 && ACK==0 && read==0)) ||(en[i] && (read_stop || start_bit) && xor_flag==0))?sda_in:(en[i] && (xor_flag ))?sda_in^buffer[number][7-count]:((xor_flag_1 && en[i]))?sda_in^buffer[number][7-count_1]:1'bz; 
assign scl_out[i]=(en[i])?scl_in:1'bz;
end
endgenerate

assign sda_in=((ACK || read) && read_stop==1'b0 && state_0==IDLE)?sda_out[number]:1'bz;

integer internal_count=0;
always@(negedge scl_o) begin

(* parallel_case *)
case(state_0)
READ_ADDR: begin
count<=count+1;
state_1<=IDLE;
count_1<=0;
if(count==7) begin
xor_flag=0;
end
else if(count==8) begin
xor_flag=0;
state_1<=ACK1;
ACK<=1;
count_1<=0;
end
else
xor_flag<=1;
end
IDLE: begin
xor_flag<=0;
count<=0;
end
endcase

(* parallel_case *)
case(state_1)
ACK1: begin
ACK<=0;
state_1<=ADDRESS_READ;
count_1<=0;
//$display("count=%0d, %0t",count,$time);
end

ADDRESS_READ: begin
count_1<=count_1+1;
if(count_1==7) begin
//$display("count=7, %0t",$time);
count<=0;
ACK<=1;
state_1<=ACK2;
end
end

ACK2: begin
ACK<=0;
state_1<=WRITE_DATA;
count_1<=0;
end

WRITE_DATA: begin
count_1<=count_1+1;
//$display("$0t: count=%0d,value=%0d",$time,count_1,buffer_1[7-count_1]);
if((count_1==(((internal_count+1)*8)-1) && r_st==0) || (count_1-1==(((internal_count+1)*8)-1) && r_st==1)) begin
ACK<=1;
state_1<=ACK3;
end
end

ACK3: begin
ACK<=0;
if(r_st==0) begin
state_1<=WRITE_DATA;
end
else if(r_st==1) begin
state_1<=READ;
count_1<=0;
read<=1;
end
internal_count<=internal_count+1;
end

READ: begin
count_1<=count_1+1;
if(count_1==7) begin
//$display("inside READ");
state_1<=ACK4;
read<=0;
end
else if(read_stop==1)
state_1<=IDLE;

end

ACK4: begin
count_1<=0;
//$display("inside ACK4");
state_1<=READ;
read<=1;
end
IDLE:begin
read<=0;
internal_count<=0;
count_1<=0;
end
endcase

end
endmodule


module i2c_slave(rst,sda,scl);
inout sda,scl;
input rst;

parameter SLAVE_ADDR=7'b1101000;

parameter IDLE=9'b0_0000_0001;
parameter READ_ADDR=9'b0_0000_0010;
parameter ACK1=9'b0_0000_0100;
parameter ADDRESS_READ=9'b0_0000_1000;
parameter ACK2=9'b0_0001_0000;
parameter WRITE_DATA=9'b0_0010_0000;
parameter ACK3=9'b0_0100_0000;
parameter READ=9'b0_1000_0000;
parameter ACK4=9'b1_0000_0000;
reg temp_sda_bit=0;

integer count=0,count_1=0;
(* keep = "true", dont_touch = "true" *) reg [8:0]state_0=IDLE,prev_st=IDLE,temp_state=IDLE,state_1=WRITE_DATA;

reg sda_en=0,scl_en=0,sda_design=0;
assign sda=(sda_en)?sda_design:1'bz;
reg start_bit=0,stop_bit=0;
reg SLAVE_CHK=1'b1;
reg [7:0] memory_data=8'b10010010;
reg [7:0]memory[255:0];
integer i;
initial begin
for(i=0;i<256;i=i+1) begin
  memory[i]=(~(i^memory_data));
end
end

reg flag=0;
always@(posedge rst, negedge sda)
begin
  if(rst) begin
    start_bit<=0;
    //        state<=IDLE;
  end
  else begin
    if(scl==1 && state_0==IDLE && (state_1==IDLE || state_1==WRITE_DATA) && flag==0) begin
//           $display("DESIGN: %0t: negedge_start",$time);
      start_bit<=1;
    end
    else
      start_bit<=0;
  end
end

always@(start_bit,state_1) begin
if(start_bit)
  state_0<=READ_ADDR;
else if(state_1==ACK1)
  state_0<=IDLE;
end

always@(*) begin
if(start_bit)
  flag=1;
else if(stop_bit)
  flag=0;
end

//   wire stop_value=(scl==1'b1 && sda==1'b1 && state_1==WRITE_DATA && state_0==IDLE)?1'b1:1'b0;

integer addr_count=0;
reg read_ack=0;
reg r_st_flag=0,temp_r_st_flag=0;
reg [7:0]SLAVE_ADDRESS; //includes the read
reg [7:0]ADDRESS; //ADRESS 
reg [8:0]DATA_BUFFER; //DATA, when write happens it takes the values from 8:1, if read happens with the rstart the 7:0, will taken in which it consitsts the slaveaddress and read bit
reg [7:0]READ_DATA_BUFFER; //DATA

always@(posedge sda) begin
if(scl==1 && state_0==IDLE && ((state_1==WRITE_DATA && r_st_flag==0) || (state_1==IDLE && prev_st==IDLE))) begin
//       $display("DESIGN: %0t: posedge_stop",$time);
  stop_bit<=1;
end
else if(state_0!=IDLE)
  stop_bit<=0;
end

always@(posedge rst or posedge scl)
begin
  if(rst) begin
    count<=0;
    //        sda_en<=0;
    count_1<=0;
    scl_en<=0;
    ADDRESS<=0;
    SLAVE_ADDRESS<=0;
    DATA_BUFFER<=0;
  end
  else begin
    count<=count+1;
    case(state_0)
      READ_ADDR: begin
        count_1<=count_1+1;
//             $display("DESIGN: %0t: count=%0d, addr_count=%0d, sda=%0d",$time,count_1,8-count_1-1,sda);
        SLAVE_ADDRESS[8-count_1-1]<=sda;
        if(count_1==7) begin
          temp_state<=ACK1;
          state_1<=ACK1;
          count<=0;
          addr_count<=0;
        end
        else
          state_1<=IDLE;
      end
      IDLE: begin
        count_1<=0;
      end
    endcase
    case(state_1)
      ACK1: begin
        if(SLAVE_CHK) begin
          count<=0;
          state_1<=ADDRESS_READ;
        end
        else
          state_1<=IDLE;
        temp_state<=IDLE;
      end
      ADDRESS_READ: begin
        ADDRESS[7-count]<=sda;
//             $display("DESIGN: %0t: addr_count=%0d, sda=%0d",$time,count,sda);
        if(count==7) begin
          state_1<=ACK2;
          temp_state<=ACK2;
        end
      end
      ACK2: begin
//             $display("DESIGN: %0t: ack2",$time);
//             $display("DESIGN: Address=%b",ADDRESS);
        state_1<=WRITE_DATA;
        temp_state<=IDLE;
        count<=0;
      end
      WRITE_DATA: begin //write also happens after detecting the r-start slave address also captures here
        //             if(flag==0)
        //               state_1<=IDLE;
        DATA_BUFFER[8- count%9]<=sda;
        if(count==1 && DATA_BUFFER[8] != temp_sda_bit) begin
//               $display("DESIGN: capturing");
          r_st_flag<=1; 
          temp_r_st_flag<=1;
//               $display("DESIGN: DATA_BUFFER=%b,temp_sda_b=%b",DATA_BUFFER[7],temp_sda_bit);
        end
//             $display("DESIGN: %0t: count=%0d, sda=%0d",$time,count,sda);
        if((count==((8*(addr_count+1)) +(addr_count-1)) && r_st_flag==0) || (count==8 && r_st_flag==1)) begin
          state_1<=ACK3;
          temp_state<=ACK3;
        end
      end
      ACK3: begin
        addr_count<=addr_count+1;
        if(r_st_flag==1) begin
          temp_state<=READ;
          state_1<=READ;
          READ_DATA_BUFFER<=memory[ADDRESS+addr_count];
//               $display("DESIGN: data for the address=%0h received=%b",ADDRESS+addr_count,memory[ADDRESS+addr_count]);
          count<=0;
        end
        else begin
          temp_state<=IDLE;
          state_1<=WRITE_DATA;          
//               $display("DESIGN: data=%b",DATA_BUFFER[8:1]);
          memory[ADDRESS+addr_count]=DATA_BUFFER[8:1];
          //               DATA_BUFFER=0;
           $display("DESIGN: %0t, ADDRESS=%b, memory stored data=%b",$time, ADDRESS+addr_count,memory[ADDRESS+addr_count]);
        end
      end
      READ: begin
        if(count==7) begin
          temp_state<=IDLE;
          state_1<=ACK4;
        end
      end
      ACK4: begin
        if(count==8) begin
//               $display("DESIGN %0t: ACK4_RECEIVED=%0b",$time,sda);
          if(sda==0) begin
            READ_DATA_BUFFER<=memory[ADDRESS+addr_count];
            state_1<=READ;
            temp_state<=READ;
            addr_count<=addr_count+1;
          end
          else begin
            state_1<=IDLE;
            temp_state<=IDLE;
          end
          count<=0;
        end
      end
      IDLE: begin
        addr_count<=0;
        ADDRESS<=0;
        DATA_BUFFER<=0;
        READ_DATA_BUFFER<=0;
        r_st_flag<=0;
        temp_r_st_flag<=0;
        if(read_ack==1 && temp_r_st_flag==1) begin
          read_ack<=0;
//               $display("-----------------DESIGN: Master read done---------------");
//               $display("DESIGN: READ_ACK=%0b",read_ack);
        end
        count<=0;
      end
      default: begin
        temp_state<=IDLE;
        state_1<=IDLE;
      end
    endcase
  end
end

always@(posedge scl) begin
prev_st<=state_1;
if(state_1==IDLE)
  prev_st<=IDLE;
end

always@(negedge scl, posedge rst)
begin
  if(rst) begin
    SLAVE_CHK<=1'b1;
    sda_en<=0;
    sda_design<=0;
  end
  else begin


    if(state_1==WRITE_DATA && count==1) begin
//           $display("DESIGN: %0t: (Applicable only for read, for write neglect can be neglected)negedge value=%b",$time,sda);
      temp_sda_bit=sda;
    end

    case(temp_state)
      ACK1: begin
        sda_en<=1;
        if(SLAVE_ADDR==SLAVE_ADDRESS[7:1]) begin
//               $display("DESIGN: sda_en=1");
          SLAVE_CHK=1'b1;
          sda_design<=1'b0; //ack
        end
        else begin
          SLAVE_CHK=1'b0;
          $error("DESIGN: SLAVE_ADDR=%b, read_addr_buffe=%b",SLAVE_ADDR,SLAVE_ADDRESS[7:1]);
          $error("DESIGN: SLAVE_ADDRESS_ERROR");
          sda_design<=1'b1; //ack
        end
      end
      ACK2: begin
        sda_en<=1'b1;
        if(ADDRESS <= 255)
          sda_design<=1'b0; //ack
        else begin
          sda_design<=1'b1; //ack
          $error("DESIGN: SLAVE_aDDRESS_ERROR");
          $error("DESIGN: ADDRESS=%b",ADDRESS);
        end
      end
      ACK3: begin
        //             temp_state<=IDLE;
        if(r_st_flag==1 && DATA_BUFFER[7:1]==SLAVE_ADDR && DATA_BUFFER[0]==1) begin
          sda_en<=1;
          sda_design<=1'b0; //ack
        end
        else if(r_st_flag==1 && (DATA_BUFFER[7:1]!=SLAVE_ADDR || DATA_BUFFER[0]!=1)) begin
          sda_en<=1;
//               $display("DESIGN: SLAVE_ADDR=%0b, DATA_BUFFER=%b",SLAVE_ADDR,DATA_BUFFER);
          $error("DESIGN: WRONG ADDRESS");
          sda_design<=1'b1; //ack
        end
        else begin //normal write ack
          sda_en<=1;
          sda_design<=1'b0; //ack
        end 
      end
      READ: begin          
        sda_en<=1;
        sda_design<=READ_DATA_BUFFER[7-count];
//             $display("DESIGN: %0t: negedge value=%0d count=%0d",$time,READ_DATA_BUFFER[7-count],count);
      end
      default: begin
        sda_en<=0;
      end
    endcase
  end
end
endmodule


//// Code your design here
module clk_divider(rst,clk,scl_clk);
input clk,rst;
output reg scl_clk=0;
integer count=0;

parameter in_clk=50_000_000; //50mhz
parameter i2c_clk=100_000; //10khz
parameter cycles=(in_clk/i2c_clk) -1;

always@(posedge clk) begin
if(rst) begin
scl_clk<=0;
count<=0;
end
else begin
if(cycles/2==count) begin
scl_clk<=~scl_clk;
count<=0;
end
else
count<=count+1;
end
end
endmodule

module tb;
reg rst,clk;
wire scl_clk;

integer match=0,mis_match=0;
//clock divider section
clk_divider dut1(rst,clk,scl_clk);
initial begin
rst=0;
clk=0;
@(posedge clk);
rst=1;
@(posedge clk);
@(posedge clk);
rst=0;
end
initial begin
#80000000;
$finish;
end

//50mhz_clk
always #10 clk=~clk;
//   always #1 clk=~clk; //to handle more transactions

parameter N=3;

reg [6:0]translation_byte;
reg [N-1:0]en=0;
wire [N-1:0]ready;

wire scl_in,sda_in;

wire [N-1:0]scl_out,sda_out;

i2c_atr dut(sda_in,scl_in,en,ready,translation_byte,sda_out,scl_out);

i2c_slave dut3(.rst(rst),.sda(sda_out[0]),.scl(scl_out[0]));
i2c_slave dut4(.rst(rst),.sda(sda_out[1]),.scl(scl_out[1]));
i2c_slave dut5(.rst(rst),.sda(sda_out[2]),.scl(scl_out[2]));


reg sda_en=0,scl_en=0;
reg sda_tb,scl_tb=1;


reg [6:0]slave_address=7'b1011001;
reg [7:0]address=8'b10011001;

assign sda_in=(sda_en)?sda_tb:1'bz;
assign scl_in=(scl_en)?scl_tb:scl_clk;


initial begin
repeat(2) begin
write(2);
 read(2);
 write(10);
 read(10);
 write(10);
 read(10);
 end
end
reg[1:0] en_1;
integer number_1;
reg [7:0]addr_write,data_write;
task write(input integer number);
repeat(number) begin
en_1=($random%3);
if(en_1>=3)
en_1=2'b01;

addr_write=$random;
en[en_1]=1'b1;
translation_byte=7'b1010100;
slave_address=translation_byte^7'b110_1000;
//  #100;
scl_en=1'b1;
scl_tb=1'b1;
sda_en=1'b1;
sda_tb=1'b1;
#100;
@(posedge scl_clk);
sda_tb=1'b0;
scl_en=1'b0;

for(integer j=0;j<7;j=j+1) begin
@(negedge scl_clk);
//         #1000;
sda_en=1'b1;
sda_tb=slave_address[6-j];
end

@(negedge scl_clk);
sda_en=1'b1;
sda_tb=1'b0;

@(negedge scl_clk);
sda_en=1'b0;

//     @(negedge scl_clk);
$display("TB:%0t sda_in=%0d",$time,sda_in);

for(integer i=0;i<8;i=i+1) begin
@(negedge scl_clk);
//         #1000;
sda_en=1'b1;
sda_tb=$random;
end

@(negedge scl_clk);
sda_en=1'b0;


number_1=$urandom_range(1,10);
repeat(number_1) begin
data_write=$random;
for(integer i=0;i<8;i=i+1) begin
@(negedge scl_clk);
//         #1000;
sda_en=1'b1;
sda_tb=data_write[7-i]; //temporaly we are considering it as a data
end
$display("data=%b",data_write);

@(negedge scl_clk);
if(dut3.DATA_BUFFER[8:1] == data_write || dut4.DATA_BUFFER[8:1] == data_write  || dut5.DATA_BUFFER[8:1] == data_write)
match=match+1;
else begin
$display("data=%b, dut3=%b, dut4=%b, dut.5=%b",data_write,dut3.DATA_BUFFER[8:1],dut4.DATA_BUFFER[8:1],dut5.DATA_BUFFER[8:1]);
mis_match=mis_match+1;
end
sda_en=1'b0;
sda_tb=1'b0;
end

@(negedge scl_clk);
sda_en=1'b1;
sda_tb=1'b0;

@(posedge scl_clk);
scl_en=1'b0;
scl_tb=1'b1;
#2000;
sda_en=1'b1;
sda_tb=1'b1;
#100;
en[en_1]=1'b0;
#1000;
end
endtask

reg [7:0] read_buffer;
reg [7:0] address_read;
task read(input integer n);
repeat(n) begin
en_1=($random%3);
if(en_1>=3)
en_1=2'b01;
address_read=$urandom_range(0,230);
en[en_1]=1'b1;
translation_byte=$random;
slave_address=translation_byte^7'b1101_000;
#100;
scl_en=1'b1;
scl_tb=1'b1;
sda_en=1'b1;
sda_tb=1'b1;
#100;
@(posedge scl_clk);
sda_tb=1'b0;
scl_en=1'b0;

for(integer j=0;j<7;j=j+1) begin
@(negedge scl_clk);
//#1000;
sda_en=1'b1;
sda_tb=slave_address[6-j];
end

//write bit
@(negedge scl_clk);
sda_en=1'b1;
sda_tb=1'b0;

@(negedge scl_clk);
sda_en=1'b0;

//     @(negedge scl_clk);
//  $display("TB:%0t sda_in=%0d",$time,sda_in);

//address
for(integer i=0;i<8;i=i+1) begin
@(negedge scl_clk);
//#1000;
sda_en=1'b1;
sda_tb=address_read[7-i];
end

//ack2
@(negedge scl_clk);
sda_en=1'b0;

//r-start
@(negedge scl_clk);
sda_en=1'b1;
sda_tb=1'b0;
#2000;
sda_en=1'b1;
sda_tb=1'b1;
@(posedge scl_clk);
#2000;
sda_tb=1'b0;

//slave_Address
for(integer j=0;j<7;j=j+1) begin
@(negedge scl_clk);
//#1000;
sda_en=1'b1;
sda_tb=slave_address[6-j];
end

//read_bit
@(negedge scl_clk);
sda_en=1'b1;
sda_tb=1'b1;

@(negedge scl_clk);
sda_en=1'b0;

number_1=$urandom_range(1,10);
for(integer j=0;j<number_1;j=j+1) begin
//read_data
@(negedge scl_clk);
sda_en=1'b0;
for(integer i=0;i<8;i=i+1) begin
@(posedge scl_clk);
//#1000;
sda_en=1'b0;
read_buffer[7-i]=sda_in;
end
$display("TB:%0t design data froms slave=%0h",$time,read_buffer);
//master ack
@(negedge scl_clk);
sda_en=1'b1;
sda_tb=(j==number_1-1)?1'b1:1'b0;

if(en==1) begin
if(dut3.READ_DATA_BUFFER == read_buffer)
match=match+1;
else
mis_match=mis_match+1;
end
else if(en==2) begin
if(dut4.READ_DATA_BUFFER == read_buffer)
match=match+1;
else
mis_match=mis_match+1;
end
else if(en==4) begin
if(dut5.READ_DATA_BUFFER == read_buffer)
match=match+1;
else
mis_match=mis_match+1;
end
end

read_buffer=0;
@(negedge scl_clk);
sda_en=1'b1;
sda_tb=0;
@(posedge scl_clk);
scl_en=1'b1;
scl_tb=1'b1;
#2000;
sda_en=1'b1;
sda_tb=1'b1;
#1000;
en[en_1]=1'b0;
#1000;
end
endtask

//flag==1 && count==27 && j==number && r_st_bit
//   always@(*) begin
//     if(dut.count>=26 && dut.count<=28) begin
//       $display("TB: %0t count=%0d, flag=%0d,j=%0d,r_st_bit=%0d",$time,dut.count,dut.flag,dut.number,dut.r_st_flag);
//     end
//   end

initial begin
$dumpfile("dump.vcd");
$dumpvars(0,tb);
end

endmodule
