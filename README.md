# Servo and 2d canvas

## Introduction

The [HTML standard](https://html.spec.whatwg.org) states that: "the [canvas element](https://html.spec.whatwg.org/multipage/#the-canvas-element) provides scripts with a resolution-dependent bitmap canvas, which can be used for rendering graphs, game graphics, art, or other visual images on the fly." Drawing on the canvas happen by way of a [rendering context](https://html.spec.whatwg.org/multipage/#renderingcontext), which comes in various flavors, including [2d](https://html.spec.whatwg.org/multipage/#2dcontext) and [WebGPU](https://gpuweb.github.io/gpuweb/#canvas-rendering).

[Vello](https://github.com/linebender/vello) is an experimental GPU compute-centric 2D renderer written in Rust. 

Servo currently implements the [canvas element](https://github.com/servo/servo/blob/291fbce434f804c2ce51d2b966bab4c472d9ecf3/components/script/dom/htmlcanvaselement.rs), and supports various rendering contexts, including a [2d one](https://github.com/servo/servo/blob/291fbce434f804c2ce51d2b966bab4c472d9ecf3/components/script/dom/canvasrenderingcontext2d.rs), using a Rust library named [Raquote](https://docs.rs/raqote/latest/raqote/), and a [WebGPU context](https://github.com/servo/servo/blob/291fbce434f804c2ce51d2b966bab4c472d9ecf3/components/script/dom/gpucanvascontext.rs) which uses our internal WebGPU capabilities. This report discusses the pros and cons of replacing Raquote with Vello for our 2d context implementation. 

## The problem
As described in a [Servo issue](https://github.com/servo/servo/issues/30636), Raquote was meant as a temporary solution until another project, [Pathfinder](https://github.com/servo/pathfinder), could replace it. As it currently stands, both Raquote and Pathfinder do not see active development, hence Servo is currently relying on legacy projects. A solution would be to switch to a project with an active community of developers and a bright future ahead of itself. 

## The solution
Vello is actively developed. As an added bonus, Vello relies on WebGPU for it's actual GPU capabilities, making it a good potential fit for a web engine like Servo(which needs to implement WebGPU anyway).

For its WebGPU integration, Vello uses [wgpu](https://github.com/gfx-rs/wgpu/), which is also used by Servo, albeit at a different level: Servo relies on `wgpu-core` inside `components/webgpu`, our WebGPU "backend", whereas Vello uses the higher-level `wgpu`, which corresponds to the DOM "front-end" that Servo implements in `components/script`. Therefore, Vello does not do anything directly with the GPU, rather it acts more like a front-end library that uses WebGPU to build and submit a list of GPU commands using its own shaders. As a result, a Vello based canvas rendering context could run entirely inside `components/script` and use the WebGPU constructs found there--this is already how servo's canvas WebGPU context works. 

The pros and cons of using Vello can be summed up as:

Pros:
- Library under active development and with a vibrant community; involvement of that community in the Servo implementation.
- A simplification of Servo's architecture: the removal of the canvas rendering thread, replaced by renderer logic running in `components/script`(the canvas component remains necessary for the WebGL rendering backends).

Cons: 
- Large amount of work. 
- The state of Servo's current WebGPU capabilities is unclear(more work?).

This "large amount of work" would consist of two parts:

1. Implement what Vello calls a [`Renderer`](https://github.com/linebender/vello/blob/399a7237ff7d7ff3d36d0a3930f9e6af10129f4f/src/lib.rs#L189) using Servo's WebGPU DOM. This would come down to a straightforward translation of what Vello currently does with `wgpu`.
2. Implement a canvas 2d rendering context using 1. This would come down to re-implementing what is currently done by the [Raquote backend](https://github.com/servo/servo/blob/291fbce434f804c2ce51d2b966bab4c472d9ecf3/components/canvas/raqote_backend.rs).

## Alternative solution
A user-agent like Servo does not have to support all rendering contexts: the [`getContext`](https://html.spec.whatwg.org/multipage/#dom-canvas-getcontext) method can return null for unsupported context types. Servo could therefore focus itself entirely on the WebGPU rendering context: it would be a rather thin wrapper around the existing WebGPU capabilities, which need to be supported anyway. Since Vello uses WebGPU, a canvas 2d rendering context could be implemented as a Wasm library(that would not be native to Servo) that would internally use Vello.  
