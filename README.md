---
layout: post
title: "三阶魔方色彩识别与复原算法"
description: " "
categories: [code]
tags: [opencv]
redirect_from:
  - /2018/01/23/
---

> Color-recognition-of-Rubik-s-Cube

* Kramdown table of contents
{:toc .toc}
# 三阶魔方色彩识别与复原算法

Created 2018.01.23 by William Yu; Last modified: 2018.07.21-V1.0.5

Contact :[windmillyucong@163.com](mailto:windmillyucong@163.com)

Copylift! 2018 William Yu. Some Rights Reserved.  

------

## Description

编译环境：VS2017+opencv3.3.0

解决魔方颜色识别问题，并提供魔方复原算法。

说明：
程序分为两部分：
debug模式和运行模式
debug模式下读取进行程序参数调试，包括
```c++
摄像头测试
check_value函数：魔方颜色hsv参数检测
```
运行模式流程：

```c++
get_color.cpp：
		摄像头读取依次读取6面颜色，每获得一张图片，通知下位机转动魔方，将54色存入数组
		position->|0 |1 |2 |
		          |3 |4 |5 |
		          |6 |7 |8 |
		meaning-> 0 1 2 3 4 5
		          W R G B O Y 
deviation.cpp：检查魔方54面颜色是否齐全，查错
standardize_color.cpp：将数组转化为标准魔方描述顺序
mySolveCube.cpp：将魔方状态投入魔方还原函数，输入魔方还原步骤
move函数.cpp：解读还原描述，转化为机械步骤，发送给下位机
```

### Cubielet

The names of the cubielet positions of the cube.

```c++
              | ************ |
              | *U1**U2**U3* |
              | ************ |
              | *U4**U5**U6* |
              | ************ |
              | *U7**U8**U9* |
|************ | ************ | ************ | ************|
|*L1**L2**L3* | *F1**F2**F3* | *R1**R2**F3* | *B1**B2**B3*|
|************ | ************ | ************ | ************|
|*L4**L5**L6* | *F4**F5**F6* | *R4**R5**R6* | *B4**B5**B6*|
|************ | ************ | ************ | ************|
|*L7**L8**L9* | *F7**F8**F9* | *R7**R8**R9* | *B7**B8**B9*|
|************ | ************ | ************ | ************|
              | *D1**D2**D3* |
              | ************ |
              | *D4**D5**D6* |
              | ************ |
              | *D7**D8**D9* |
              | ************ |
      
   |		  | 0 1 2
   |		  | 3 4 5
   |		  | 6 7 8
   | 36 37 38 | 18 19 20 | 9  10 11 | 45 46 47 
   | 39 40 41 | 21 22 23 | 12 13 14 | 48 49 50 
   | 42 43 44 | 24 25 26 | 15 16 17 | 51 52 53 
   |		  | 27 28 29
   |		  | 30 31 32
   |		  | 33 34 35
    

string goal[] = { 
		"UF", "UR", "UB", "UL", "DF", "DR", "DB", "DL", "FR", "FL", "BR", "BL",
		"UFR", "URB", "UBL", "ULF", "DRF", "DFL", "DLB", "DBR" };


   |		         | 15.1   3.1   14.1  |
   |		         | 4.1     U    2.1   |
   |		         | 16.1   1.1   13.1  |
   | 15.3  4.2  16.2 | 16.3   1.2   13.2  | 13.3  2.2  14.2 | 14.3  3.2  15.2
   | 12.2   L   10.2 | 10.1    F    9.1   | 9.2    R   11.2 | 11.1   B   12.1
   | 19.2  8.2  18.3 | 18.2   5.2   17.3  | 17.2  6.2  20.3 | 20.2  7.2  19.3
   |		         | 18.1   5.1   17.1
   |		         | 8.1     D    6.1
   |		         | 19.1   7.1   20.1
```

### Facelet

The names of the facelet positions of the cube.

```c++
string goal[] = { 
		"UF", "UR", "UB", "UL", "DF", "DR", "DB", "DL", "FR", "FL", "BR", "BL",
		"UFR", "URB", "UBL", "ULF", "DRF", "DFL", "DLB", "DBR" };

Facelet:
	The names of the facelet positions of the cube
				  | ************ |
			  	  | *U1**U2**U3* |
				  | ************ |
			  	  | *U4**U5**U6* |
				  | ************ |
				  | *U7**U8**U9* |
	|************ | ************ | ************ | ************|
	|*L1**L2**L3* | *F1**F2**F3* | *R1**R2**F3* | *B1**B2**B3*|
	|************ | ************ | ************ | ************|
	|*L4**L5**L6* | *F4**F5**F6* | *R4**R5**R6* | *B4**B5**B6*|
	|************ | ************ | ************ | ************|
	|*L7**L8**L9* | *F7**F8**F9* | *R7**R8**R9* | *B7**B8**B9*|
	|************ | ************ | ************ | ************|
				  | *D1**D2**D3* |
				  | ************ |
				  | *D4**D5**D6* |
				  | ************ |
				  | *D7**D8**D9* |
				  | ************ |

	A cube definition string "UBL..." means for example: In position U1 we have the U - color, in position U2 we have the
	B - color, in position U3 we have the L color etc.according to the order U1, U2, U3, U4, U5, U6, U7, U8, U9, R1, R2,
	R3, R4, R5, R6, R7, R8, R9, F1, F2, F3, F4, F5, F6, F7, F8, F9, D1, D2, D3, D4, D5, D6, D7, D8, D9, L1, L2, L3, L4,
	L5, L6, L7, L8, L9, B1, B2, B3, B4, B5, B6, B7, B8, B9 of the enum constants.
	"""
	U1 = 0
	U2 = 1
	U3 = 2
	U4 = 3
	U5 = 4
	U6 = 5
	U7 = 6
	U8 = 7
	U9 = 8
	R1 = 9
	R2 = 10
	R3 = 11
	R4 = 12
	R5 = 13
	R6 = 14
	R7 = 15
	R8 = 16
	R9 = 17
	F1 = 18
	F2 = 19
	F3 = 20
	F4 = 21
	F5 = 22
	F6 = 23
	F7 = 24
	F8 = 25
	F9 = 26
	D1 = 27
	D2 = 28
	D3 = 29
	D4 = 30
	D5 = 31
	D6 = 32
	D7 = 33
	D8 = 34
	D9 = 35
	L1 = 36
	L2 = 37
	L3 = 38
	L4 = 39
	L5 = 40
	L6 = 41
	L7 = 42
	L8 = 43
	L9 = 44
	B1 = 45
	B2 = 46
	B3 = 47
	B4 = 48
	B5 = 49
	B6 = 50
	B7 = 51
	B8 = 52
	B9 = 53

Color:
	""" The possible colors of the cube facelets. Color U refers to the color of the U(p)-face etc. 
	Also used to name the faces itself."""
	U = 0
	R = 1
	F = 2
	D = 3
	L = 4
	B = 5
```

## Img

![cube.jpg](https://github.com/YuYuCong/YuYuCong.github.io/blob/master/img/cube.jpg?raw=true)

## See Also

- <http://tomas.rokicki.com/cubecontest/winners.html>
- <http://williamyu.top/blog/2018/02/05/solve-cube/>
- <http://www.zunny.com/RUBIK.HTM>

## Contributing / Contact

Have anything in mind that you think is awesome and would fit in this blog? Feel free to send a pull request.

Feel free to [contact me](mailto:windmillyucong@163.com) anytime for anything.

-----



## License

[![CC0](http://i.creativecommons.org/p/zero/1.0/88x31.png)](http://creativecommons.org/publicdomain/zero/1.0/)

