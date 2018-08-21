Author: erfur
Date: Tue Aug 21 08:30:46 +03 2018

# Hackcon 2018 - Anti RE

Analyzing the binary, what we get is a main loop with several similar 
branches. After looking at all the branches I realized that they each do 
one simple operation. Only after getting the flag I understood the fact 
that this was a virtual machine implementation.


The virtual code is fixed, meaning that the same procedure is executed 
each time the app is run. Instructions are in one of the following two 
formats:

```
1.	instruction opcode  +  dstaddr  +  srcaddr
	     (1 byte)         (1 byte)	  (1 byte)
```

or

```
2.	instruction opcode  +  dstaddr  +  value
	     (1 byte)         (1 byte)    (4 bytes)
```

Some things to consider:
- Memory is accessed as 4 bytes of blocks.
- Virtual memory starts at [rsp+188h].
- Real memory being accessed is [rsp+188h+4*addr].
- Our input is stored at the first 16 bytes of virtual memory.

There are 7 different instructions, each assigned to different 
opcodes:

```
OPCODE	FUNC	FORMAT
----------------------
0x0 	XOR		1
0x2 	OR		1
0x3 	ADD		1
0x4 	SUB		1
0x5 	SHL		1
0x10 	STORE	2
0x11 	COPY	1
0xFF	HALT
```

The running procedure is as follows:
```
STORE	0x6, 0x41
STORE	0x7, 0x50
STORE	0xA, 0x8
SHL		0x7, 0xA
STORE	0x8, 0x56
STORE	0xA, 0x10
SHL		0x8, 0xA
STORE	0x9, 0x45
STORE	0xA, 0x18
SHL		0x9, 0xA
OR		0x6, 0x7
OR		0x6, 0x8
OR		0x6, 0x9
COPY	0x7, 0x3
XOR		0x7, 0x6
COPY	0x6, 0x2
XOR		0x6, 0x3
STORE	0x8, 0x102730E
XOR		0x6, 0x8
COPY	0x5, 0x310AA04
ADD		0x5, 0x1
XOR		0x5, 0x2
COPY	0x8, 0x0
STORE	0x9, 0x4F0D301
SUB		0x8, 0x9
XOR		0x8, 0x1
OR		0x5, 0x6
OR		0x5, 0x7
OR		0x5, 0x8
COPY	0x0, 0x50
HALT
```

After the procedure is run, if the first 4 bytes of virtual memory is 
zero, then we get the flag. Simplifying the above procedure, we get an 
easy system to solve:

```
(MEM[0x3]^0x45565041) | (MEM[0x2]^MEM[0x3]^0x102730E) | 
(MEM[0x1]^(MEM[0x0]-0x4F0D301)) | (MEM[0x2]^(MEM[0x1]+0x310AA04)) == 0
```

And we get the correct input:
```
MEM[0x0] = 0x46344C4C = "LL4F"
MEM[0x1] = 0x4143794B = "KyCA"
MEM[0x2] = 0x4454234F = "O#TD"
MEM[0x3] = 0x45565041 = "APVE"

"LL4FKyCAO#TDAPVE"
```
