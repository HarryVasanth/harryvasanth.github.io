---
layout: post
title: "Misc - Pong for TI-84 (ti-basic)"
date: 2016-12-11 14:39:29 +00:01
categories: misc
tags: ti-basic
---

## Introduction

In this tutorial, I'll guide you through the process of creating a simple Pong game for the TI-84 calculator using TI-BASIC. Pong is a classic arcade game that involves two paddles and a ball. The objective is to bounce the ball back and forth between the paddles, scoring points when the opponent misses the ball.

## Prerequisites

Ensure you have a TI-84 calculator and the cable to connect it to your computer. You'll also need TI Connect CE software to transfer programs to the calculator.

## Getting Started

```
ClrHome
Disp "","   Welcome to","      Pong!
Pause 
8->B
4->A
4->C
4->D
randInt(0,1->E
randInt(0,1->F
0->Z
0->Y
Repeat B=1 or B=16
	ClrHome
	Output(A,B,"o
	Output(C,1,"!
	Output(D,16,"!
	getKey
	If Ans=73 and C>1
	C-1->C
	If Ans=93 and C<8
	C+1->C
	If Ans=25 and D>1
	D-1->D
	If Ans=34 and D<8
	D+1->D
	If A=1
	0->E
	If A=8
	1->E
	If E=1
	A-1->A
	If E=0
	A+1->A
	If F=1
	B-1->B
	If F=0
	B+1->B
	If A=C and B=2
	0->F
	If A=D and B=15
	1->F
	Y+randInt(3,5->Y
	Z+randInt(3,5->Z
End
If B=1
Then
	Disp "Right wins!
	Y->Z
Else
	Disp "Left wins!
End
Pause 
prgmSCORE
```

## Conclusion

Congratulations! You've successfully created a basic Pong game for the TI-84 calculator. Feel free to customize and enhance the game with additional features, such as sound effects or advanced gameplay mechanics.
