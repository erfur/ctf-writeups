Author: erfur

Date: Wed Aug 22 23:40:25 +03 2018

# Hackcon 2018 - CryptoRevSalad

We are given a pcap file with lots of noise. After checking out many of 
the streams in wireshark, I came across two TCP streams that were 
interesting, first of which is this message:

```
From: Hacker Man <nothacker@hackcon.in>
Date: Wed, Aug 6, 2018 at 8:55 PM
Subject: Re: The Code
To: Not Hacker Man <thehacker@hackcon.in>

I'll take your struggle as a compliment to my hard wark. Also I fear 
that you are 'Hacker Man' so it might be above your capabilities to 
decode the message.    
PS: Please Find Attached(don't really) the encoding binary.

Regards
Not ! Hacker Man

>       From: Not Hacker Man <thehacker@hackcon.in>
>       Date: Wed, Aug 5, 2018 at 5:55 PM
>       Subject: The Code
>       To: Hacker Man <nothacker@hackcon.in>
>
>       Hey, I'm not doing well since your last e-mail. I recieved the 
encoded super secret code but I'm unable to decode it. Please help me.
>
>       Regards
>       ! (Hacker Man)


        >       From: Hacker Man <nothacker@hackcon.in>
        >       Date: Wed, Aug 1, 2018 at 8:55 PM
        >       Subject: Re: The Code
        >       To: Not Hacker Man <thehacker@hackcon.in>
        >
        >       Hey, how you not !(doin')?
        >       Thanks for asking the encoded super secret message is enclosed in the following hash box.
        >
        >       ########################################################################################################################
        >       #`0vo&jm1[`F.^t.RGG..#`2.;c).lvZk{|.~B...[II..D..#
        >       ########################################################################################################################
        >
        >       Regards
        >       Not ! Hacker Man


                >       From: Not Hacker Man <thehacker@hackcon.in>
                >       Date: Wed, Aug 1, 2018 at 5:55 PM
                >       Subject: The Code
                >       To: Hacker Man <nothacker@hackcon.in>
                >
                >
                >       Hey, how you doin'?
                >       Can you send super secret codes that we wrote. But I think our mail is not secure enough so don't sent them in 
paintext.
                >
                >       Regards
                >       Not Hacker Man
```

And the second one is this hexdump (shortened version):

```
0000000 4b50 0403 0014 0000 0008 bb4c 4d01 dde1
0000010 906a 126d 0000 3ad8 0000 0006 0000 6e65
0000020 6f63 6564 5bed 707d d753 bf95 f0fa 6007
0000030 996c 4a18 8020 1f08 490b 2451 18db b093
0000040 db20 783c 0306 3f8e 2508 3ceb 92cb 2b6c
0000050 92c8 3d2b cd81 0286 606b 71a2 f59c 4da4
0000060 3fea bbb2 204c b49d 36c3 974e 0ca6 829b
0000070 338d dd38 b6d9 bb6e b2cd 3a69 32e3 6b4d
0000080 6487 29d7 f129 8212 9cf7 eefb de95 7a7b
0000090 b0ca 266e 717f f23d e779 cf77 f739 8fdc
...
0001250 62f1 b2d7 2e64 df6b c3a9 feb9 1d5a d26e
0001260 7cd1 eaa2 aabb 779c 7429 6eb1 e9c1 9fd7
0001270 ed7a cbaa a5fc 1d0a 1d63 e9c6 2228 fba9
0001280 de1f 46df f3ed e3c0 fe79 1c05 fa2b fff6
0001290 500f 014b 1402 1400 0000 0800 4c00 01bb
00012a0 e14d 6add 6d90 0012 d800 003a 0600 0000
00012b0 0000 0000 0000 2000 0000 0000 0000 6500
00012c0 636e 646f 5065 054b 0006 0000 0100 0100
00012d0 3400 0000 9100 0012 0000 0000          
00012db
```

After transforming the hexdump back to raw data with:

```
$ xxd -r > file
```

We get a zip file with an executable called "encode" in it. When 
executed, the app will as for a message to encode, then print the 
encoded message. Since we know the encoded message

```
00000502  20 20 20 20 20 3e 20 20 20 20 20 20 20 23 60 30        >       #`0
00000512  76 6f 26 6a 6d 31 5b 60 46 0f 5e 74 1e 52 47 47   vo&jm1[`F.^t.RGG
00000522  19 1d 23 60 32 0e 3b 63 29 0f 6c 76 5a 6b 7b 7c   ..#`2.;c).lvZk{|
00000532  14 7e 42 12 04 03 5b 49 49 15 17 44 10 11 23 0d   .~B...[II..D..#.
```

we just need to reverse the binary and decode this message.

## ./encode

App starts by creating a string:

```
"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}[]()_1234567890!
@#$%^&*<>?"
```

Which will be used as a key for encoding. The input is taken right after.
At this point, the length of the input is calculated (rax), then 

```
0x004012b9      48f7d8         neg rax
0x004012bc      83e00f         and eax, 0xf
```

is executed on the length, after that the result is compared to a counter
which is incremented every time a character is added to our input. On a
high level, this is done to make our input's length a multiple of 16.
For every character missing, a random character from the key is added to
out input, except that the selected character isn't random at all. Since
rand() is used without properly setting a seed value with srand(), 
srand(1) is used by default, meaning that the following stream will form
with each rand(!)omly selected char from the key:

```
xg)JHpAmj3QBY1d
```

After forming the string with a length multiple of 16, the 'encode'
function is called. In short, for each 16-byte block:

	- XOR every 4 bytes (a)
	- XOR previous four values (b)
	- XOR the result in b with four results in a (c)
	- For every 4 bytes, XOR the byte with the corresponding value in c
	
To illustrate:

```
(⊕):XOR
 -------------------------------
|0|1|2|3|4|5|6|7|8|9|a|b|c|d|e|f| 16-byte block from input
 -------------------------------
 \ | | / \ | | / \ | | / \ | | /
   ⊕        ⊕       ⊕       ⊕
    \       |       |       /
	  \      |     |      /
	    \     |   |     /
	      \    | |    /
			\  | |  /
			  \| |/
			    ⊕ --------------------.
			                          |
									  |
 -------------------------------      |
|0|1|2|3|4|5|6|7|8|9|a|b|c|d|e|f| 16-byte block from input
 -------------------------------      |
  \ | | / \ | | / \ | | / \ | | /     |
    ⊕       ⊕       ⊕       ⊕         |
    |   .---|---.---|---.---|---------|
    v   |   v   |   v   |   v         |
    ⊕  <'   ⊕  <'   ⊕  <'   ⊕  <------'
 / | | \ / | | \ / | | \ / | | \
 ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕ ⊕
 -------------------------------
|0|1|2|3|4|5|6|7|8|9|a|b|c|d|e|f| 16-byte block from input
 -------------------------------
 | | | | | | | | | | | | | | | |
 -------------------------------
|0|1|2|3|4|5|6|7|8|9|a|b|c|d|e|f| encoded string
 -------------------------------
```

In the end, every byte will be XORed with the rest of the 4-byte blocks
in its respective 16-byte block. So, to get any character, we only need
to XOR the rest of its block and XOR the result with it. The following 
code decodes the string and prints the flag:

```
a = "`0vo&jm1[`F\x0f^t\x1eRGG\x19\x1d#`2\x0e;c)\x0flvZk{|\x14~B\x12\x04\x03[II\x15\x17D\x10\x11"

for i in range(0, len(a), 16):
	for j in range(4):
		xorsum = 0
		for k in range(4):
			if k!=j:
				for l in range(i+k*4, i+k*4+4):
					xorsum ^= ord(a[l])
		for m in range(i+j*4, i+j*4+4):
			print(chr(ord(a[m])^xorsum), end='')
```

```
d4rk{70ld_y0u_5ymm37r1c_k3y_is_n07_53cur3!!}c0de
```
