---
layout: post
comments: true
published: true
title: TinyGo Program Translation
description: 'Translate a Arduino C-Sketch (MAX7219 Driver) into Golang with TinyGO'
---
## MAX7219 Driver in TinyGo

Wow, I wonder what prompted me to start blogging again...

Anyway I've had some free time on my hands, and I've been learning Go for work. After a coworker mentioned [TinyGo](https://tinygo.org/), I figured I'd take a look at the microcontroller support library and translate one of my old [Arduino projects](https://samclane.github.io/Python-Arduino-PyFirmata/) into Go for fun. Since I've written, and rewritten a driver for the MAX7219 display driver chip, I figured I'd port it to TinyGo for their drivers repository. 

Again, here is the sketch in C:

```cpp
int dataIn = 2;
int load = 3;
int clock = 4;
 
int maxInUse = 4;    //change this variable to set how many MAX7219's you'll use
 
int e = 0;           // just a variable
 
// define max7219 registers
byte max7219_reg_noop        = 0x00;
byte max7219_reg_digit0      = 0x01;
byte max7219_reg_digit1      = 0x02;
byte max7219_reg_digit2      = 0x03;
byte max7219_reg_digit3      = 0x04;
byte max7219_reg_digit4      = 0x05;
byte max7219_reg_digit5      = 0x06;
byte max7219_reg_digit6      = 0x07;
byte max7219_reg_digit7      = 0x08;
byte max7219_reg_decodeMode  = 0x09;
byte max7219_reg_intensity   = 0x0a;
byte max7219_reg_scanLimit   = 0x0b;
byte max7219_reg_shutdown    = 0x0c;
byte max7219_reg_displayTest = 0x0f;
 
void putByte(byte data) {
  byte i = 8;
  byte mask;
  while(i > 0) {
    mask = 0x01 << (i - 1);      // get bitmask
    digitalWrite( clock, LOW);   // tick
    if (data & mask){            // choose bit
      digitalWrite(dataIn, HIGH);// send 1
    }else{
      digitalWrite(dataIn, LOW); // send 0
    }
    digitalWrite(clock, HIGH);   // tock
    --i;                         // move to lesser bit
  }
}
 
void maxSingle( byte reg, byte col) {    
//maxSingle is the "easy"  function to use for a single max7219
 
  digitalWrite(load, LOW);       // begin    
  putByte(reg);                  // specify register
  putByte(col);//((data & 0x01) * 256) + data >> 1); // put data  
  digitalWrite(load, LOW);       // and load da stuff
  digitalWrite(load,HIGH);
}
 
void maxAll (byte reg, byte col) {    // initialize  all  MAX7219's in the system
  int c = 0;
  digitalWrite(load, LOW);  // begin    
  for ( c =1; c<= maxInUse; c++) {
  putByte(reg);  // specify register
  putByte(col);//((data & 0x01) * 256) + data >> 1); // put data
    }
  digitalWrite(load, LOW);
  digitalWrite(load,HIGH);
}
 
void maxOne(byte maxNr, byte reg, byte col) {    
//maxOne is for addressing different MAX7219's,
//while having a couple of them cascaded
 
  int c = 0;
  digitalWrite(load, LOW);  // begin    
 
  for ( c = maxInUse; c > maxNr; c--) {
    putByte(0);    // means no operation
    putByte(0);    // means no operation
  }
 
  putByte(reg);  // specify register
  putByte(col);//((data & 0x01) * 256) + data >> 1); // put data
 
  for ( c =maxNr-1; c >= 1; c--) {
    putByte(0);    // means no operation
    putByte(0);    // means no operation
  }
 
  digitalWrite(load, LOW); // and load da stuff
  digitalWrite(load,HIGH);
}
 
 
void setup () {
 
  pinMode(dataIn, OUTPUT);
  pinMode(clock,  OUTPUT);
  pinMode(load,   OUTPUT);
 
  digitalWrite(13, HIGH);  
 
//initiation of the max 7219
  maxAll(max7219_reg_scanLimit, 0x07);      
  maxAll(max7219_reg_decodeMode, 0x00);  // using an led matrix (not digits)
  maxAll(max7219_reg_shutdown, 0x01);    // not in shutdown mode
  maxAll(max7219_reg_displayTest, 0x00); // no display test
   for (e=1; e<=8; e++) {    // empty registers, turn all LEDs off
    maxAll(e,0);
  }
  maxAll(max7219_reg_intensity, 0x0f & 0x0f);    // the first 0x0f is the value you can set
                                                  // range: 0x00 to 0x0f
}
```


Go's interface pattern will allow us to make this a lot more succinct, and allow us to better encapsulate the function of the MAX, henceforth called `Device` to follow `tinygo/driver`'s nameing convention.

We start by declaring our package name, and a basic definition of a `MAX7219 Struct`:

```go
package max7219

import (
	"github.com/tinygo-org/tinygo/src/machine"
)

// Uses a 3-wire serial interface
type Device struct {
	Data     machine.Pin // DIN
	Load     machine.Pin // Can also be labeled CS
	Clock    machine.Pin // CLK
	MaxInUse int         // Number of daisy chained MAX units
}
```

We require use of the TinyGo `machine` library to interface with Arduino's pins. Machine is a hardware-abstraction layer, similar to the Arduino library in the Arduino IDE. It will allow us to run this code on a variety of hardware, provided the pins to the MAX7219 configured similarly in the code.

The Factory Method pattern is very popular in Go, and taught as idiomatic. Also, the `drivers` library seems to use that pattern, so we'll follow suit.

```go
// Initialize the pins for a MAX7219 device as output pins
func New(pd machine.Pin, pl machine.Pin, pc machine.Pin, n ...int) *Device {

	pl.Configure(machine.PinConfig{Mode: machine.PinOutput})
	pd.Configure(machine.PinConfig{Mode: machine.PinOutput})
	pc.Configure(machine.PinConfig{Mode: machine.PinOutput})

	inUse := 1
	if len(n) > 0 {
		inUse = n[0]
	}

	dev := Device{Data: pd, Load: pl, Clock: pc, MaxInUse: inUse}

	return &dev
}

// Initialize the matrix for input
func (d Device) Configure() {
	d.MaxSingle(REG_SCANLIMIT, 0x07)
	d.MaxSingle(REG_DECODE_MODE, 0x00)
	d.MaxSingle(REG_SHUTDOWN, 0x01)
	d.MaxSingle(REG_DISPLAY_TEST, 0x00)
	// Wipe all rows of the matrix
	for r := 0; r <= 8; r++ {
		d.MaxSingle(byte(r), 0)
	}
	d.MaxSingle(REG_INTENSITY, 0x0F&0x0F)
}
```

Notice we split our function into two parts, removing the implementation details from the "constructor" method `New()` and into the `Configure()` method. This allows us to instantiate a `Device` before connecting to hardware, which is useful for testing. 

---

NOTE: If you're wondering where the registeres `REG_*` are coming from, they're in a separate file in the same directory called `registers.go`

```go
package max7219

const (
	REG_NOOP         = 0x00
	REG_DIGIT0       = 0x01
	REG_DIGIT1       = 0x02
	REG_DIGIT2       = 0x03
	REG_DIGIT3       = 0x04
	REG_DIGIT4       = 0x05
	REG_DIGIT5       = 0x06
	REG_DIGIT6       = 0x07
	REG_DIGIT7       = 0x08
	REG_DECODE_MODE  = 0x09
	REG_INTENSITY    = 0x0A
	REG_SCANLIMIT    = 0x0B
	REG_SHUTDOWN     = 0x0C
	REG_DISPLAY_TEST = 0x0F
)
```

This saves space in the main file, and makes the hardware more configurable. 

---

We then almost copy the code from `main.c` verbatim, only changing some formatting where Go differs from C:

```go
// Helper function to send a single byte to the matrix
func (d Device) putByte(data byte) {
	for i := 0x08; i > 0; i-- {
		mask := byte(0x01 << (i - 1))
		d.Clock.Low() // tick
		if (data & mask) > 0 {
			d.Data.High()
		} else {
			d.Data.Low()
		}
		d.Clock.High() // tock
	}
}

// Write the bitstring `col` to `row`
// Note: Row is indexed at 1
func (d Device) MaxSingle(row byte, col byte) {
	d.Load.Low()
	d.putByte(row)
	d.putByte(col)
	d.Load.Low()
	d.Load.High()
}

func (d Device) MaxAll(row byte, col byte) {
	d.Load.Low() // begin
	for c := 1; c <= d.MaxInUse; c++ {
		d.putByte(row)
		d.putByte(col)
	}
	d.Load.Low()
	d.Load.High()
}

// Address different MAX chips while having a couple cascaded
func (d Device) MaxOne(index int, row byte, col byte) {
	d.Load.Low()

	for c := d.MaxInUse; c > index; c-- {
		d.putByte(0) // NOP
		d.putByte(0) // NOP
	}

	d.putByte(row)
	d.putByte(col)

	for c := index - 1; c >= 1; c-- {
		d.putByte(0) // NOP
		d.putByte(0) // NOP
	}
}
```

A note of caution: you cannot currently pass pointers through function parameters in TinyGo. The compiler will throw very cryptic errors and dump a stacktrace to the console if you try and feed it a file where this occurs. This is why I had to scrap a `WriteMatrix(mat [8]byte)`, as it passes a pointer to the byte array `mat`. Otherwise, this could be even more useful. However, it's much shorter and more readable to use a `Device` struct for a matrix, then having to iterate over a byte array every time you want to write to the entire matrix. However, it's a small price to pay for Go's interface-driven function signatures, which are very handy and make the code much more reasonable, without all the cruft of a fully OOP language. It's much more fun to write for microcontrollers than, say, a heftier language like C++. Still, the TinyGo library has a little ways to go before it's ready for use in any serious application. But when it is, it will surely take a large portion of C developers with it. 
