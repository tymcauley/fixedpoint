# Fixed-Point User Library for Chisel [![Test](https://github.com/ucb-bar/fixedpoint/actions/workflows/test.yml/badge.svg?branch=master)](https://github.com/ucb-bar/fixedpoint/actions)

## Introduction

This is an attempt to add the FixedPoint type to Chisel as a user-level library. The main motivation behind this is the deprecation of FixedPoint starting from [Chisel v3.6](https://github.com/chipsalliance/chisel/releases/v3.6.0), where Chisel is transitioning from Scala for Chisel compiler (SFC) to MLIR FIRRTL compiler. In this transition, FixedPoint [was dropped at the FIRRTL level](https://github.com/chipsalliance/chisel/issues/3161), meaning that FixedPoint would need to be re-implemented as something that depends on the remaining existing Chisel types, and by using Chisel's public API. This library aims to provide that user-level implementation.

The main goal of this library is to faithfully reproduce Chisel's established FixedPoint interface, so that existing code that depends on it can just replace Chisel's FixedPoint imports with this library's FixedPoint imports and have the code
work just as it did before. To that end, the following classes/traits have been (re)implemented:
* FixedPoint
* BinaryPoint, KnownBinaryPoint, UnknownBinaryPoint
* HasBinaryPoint

Currently, this library works with [Chisel v6.5.0](https://github.com/chipsalliance/chisel/releases/v6.5.0).

## Usage

Here is an example module using this library's FixedPoint type:

```scala
import chisel3._
import circt.stage.ChiselStage
import fixedpoint._

class Example extends Module {
  val in = IO(Input(FixedPoint(8.W, 4.BP)))
  val out1 = IO(Output(FixedPoint(8.W, 5.BP)))
  val out2 = IO(Output(FixedPoint()))

  out1 := in
  out2 := WireDefault(3.14.F(8.BP))
}

object ExampleApp extends App {
  println(ChiselStage.emitSystemVerilog(new Example))
}
```

This outputs the following SystemVerilog code:

```systemverilog
// Generated by CIRCT firtool-1.43.0
module Example(	// <stdin>:3:10
  input         clock,	// <stdin>:4:11
                reset,	// <stdin>:5:11
  input  [7:0]  in,	// src/main/scala/Example.scala:6:14
  output [7:0]  out1,	// src/main/scala/Example.scala:7:16
  output [10:0] out2	// src/main/scala/Example.scala:8:16
);

  assign out1 = {in[6:0], 1'h0}; // <stdin>:3:10, src/main/scala/Example.scala:10:8
  assign out2 = 11'h324;         // <stdin>:3:10, src/main/scala/Example.scala:11:22
endmodule
```  

## How It Works

FixedPoint is implemented as an extension of `Record`, which has one *anonymous* data field of type `SInt`; it is also an [opaque type](https://github.com/chipsalliance/chisel/blob/v5.1.0/core/src/main/scala/chisel3/experimental/OpaqueType.scala). Most of the arithmetic involving FixedPoints has been delegated to the `SInt` arithmetic of the underlying data field, where shift operations are first used to align the data of FixedPoints that have different binary points. Connect methods have also been  overridden to account for data alignment of FixedPoints with different binary points, and to implement binary point inference.

## Limitations

It was challenging to implement FixedPoint using Chisel's public API as some of the needed functionality for FixedPoints was originally implemented in Chisel's package-private objects, which cannot be accessed or altered from a user-level library. Due to this issue, some of the original FixedPoint functionality could not be implemented without limited workarounds. Here is the current list of limitations of this implementation of FixedPoint:
* FixedPoints with different binary points are not aligned properly when used inside Chisel's Muxes (`Mux`, `Mux1H`, `PriorityMux`, `MuxLookup`, `MuxCase`). To that end, these objects have been redefined in the package `fixedpoint.shadow` to align FixedPoints by width and binary point before calling Chisel's corresponding Mux objects. In order to make FixedPoint work properly with Muxes, you have to import the new Mux definitions as follows:
  ```scala
  import fixedpoint.shadow.{Mux, Mux1H, PriorityMux, MuxLookup, MuxCase}
  ```
* Records with inferred widths cannot be used inside `Mux1H`. If you want to use FixedPoints inside a `Mux1H`, make sure that both width and binary point are specified in advance.
* FixedPoints do not connect properly if they are nested inside a `Bundle` or `Record`. If you have a bundle/record that has a FixedPoint field, you will have to extend it with the `ForceElementwiseConnect` trait. If you have a bundle `Foo` defined as:
  ```scala
  class Foo extends Bundle {
    val data = FixedPoint(8.W, 4.BP)
  }
  ```
  ...then you will have to redefine it as:
  ```scala
  class Foo(implicit val ct: ClassTag[Foo]) extends Bundle
    with ForceElementwiseConnect[Foo] {
    val data = FixedPoint(8.W, 4.BP)
  }
  ```
  If you have multiple levels of nesting inside Bundles, each bundle at every level
needs to extend `ForceElementwiseConnect`.
* FixedPoints do not connect properly if they are part of a `Vec`. **Currently, there is no solution available for this problem.**
