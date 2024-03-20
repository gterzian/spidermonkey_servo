# Servo and SpiderMonkey


## Introduction

[Servo](https://github.com/servo/servo) is a Web engine--a piece of software written in Rust and implementing a host of standards which together form the Web platform--which relies on [SpiderMonkey](https://spidermonkey.dev/), itself written in C++, for its script execution capabilities. 
This report will provide an overview of Servo's integration of SpiderMonkey, and provide an outlook on improving the modularity of that integration. 

## Script execution and the Web

SpiderMonkey in and of itself is unrelated to the Web platform; rather, it is an implementation of the [ECMAScript specification](https://tc39.es/ecma262/), as well as of Wasm related specifications such as the [Wasm JS interface](https://webassembly.github.io/spec/js-api/index.html), [core Wasm](https://www.w3.org/TR/wasm-core/), and the [Wasm Web API](https://www.w3.org/TR/wasm-web-api/). Those capabilities are then integrated into Servo, the engine implementing the Web platform as specified by core standards such as [HTML](https://html.spec.whatwg.org/), specifications providing various types of infrastructure shared across the Web platform, such as [Web IDL](https://webidl.spec.whatwg.org/), and a host of additional specifications providing peripheral capabilities to the Web platform, such as [WebGPU](https://gpuweb.github.io/gpuweb/). 

The integration of SpiderMonkey into Servo takes places at various levels of granularity, resulting in an API surface that does not map to the various specifications which Servo and SpiderMonkey are meant to implement, and resulting in a tight-coupling between the two. A good example of such tight-coupling is Servo's implementation of the [Streams standard](https://streams.spec.whatwg.org), which, while not being part of EcmaScript but rather part of the Web platform, relies on an implementation provided(and now [deprecated](https://spidermonkey.dev/blog/2022/01/14/newsletter-firefox-96-97.html)) by SpiderMonkey itself. 

## SpiderMonkey: a modern script engine

SpiderMonkey is the Javascript and Wasm implementation [used in FireFox](https://firefox-source-docs.mozilla.org/js/index.html#spidermonkey). As FireFox is a modern browser with a feature-set similar to Chrome([differences between Chrome 124 and FireFox 125](https://caniuse.com/?compare=chrome+124,firefox+125&compareCats=all)), we can deduct that SpiderMonkey is a modern script execution engine with a feature-set similar to [V8](https://v8.dev/), the script engine used in Chrome. 

Online forums often contain comments about V8 outperforming SpiderMonkey, but a more in-depth look appears to relegate this idea to the urban myth. The often cited performance testing results are based on older so-called synthetic test suites which have since been discredited. An example is Octane, a [retired](https://v8.dev/blog/retiring-octane) test-suite for which v8 was particularly tuned. The problem however was that this tuning was a form of over-optimization which often would slow-down real-word browsing use-cases. Modern performance test-suites such as [Speedometer](https://browserbench.org/Speedometer2.0/) are aimed at a holistic measurement of real-world browsing use-cases, and the SpiderMonkey team targets the results of these benchmarks([example for React](https://bugzilla.mozilla.org/show_bug.cgi?id=1867359) and [Google Docs](https://hacks.mozilla.org/2020/11/warp-improved-js-performance-in-firefox-83/)). 

While performance and feature-sets are bound to differ when comparing specific releases, generally speaking SpiderMonkey appears as capable and well-maintained as V8. 

## Servo's integration of SpiderMonkey

Servo being written in Rust, integration with the C++ codebase of SpiderMonkey happens via a crate of [Rust bindings](https://github.com/servo/mozjs). These bindings consists of two parts:
- `mozjs-sys`: low-level Rust bindings to the C++ API. In Servo these are used as `use js::jsapi`, `use js::jsval`, and `use js::glue`.
- `mozjs`: Higher-level bindings that hide the SpiderMonkey API behind idiomatic Rust API. In Servo these are used as `use js::rust`.

Servo uses both parts of these bindings, additional code found in Servo itself at `components/script/dom/bindings`, and other code which is not part of the repository but rather [generated as part of the build process](https://github.com/servo/servo/blob/b1c9378af457093fa1fcee83de29736dc49f7c0f/components/script/dom/bindings/codegen/CodegenRust.py). The generated code deals with mundane low-level matters that makes Rust objects integrate with the SpiderMonkey runtime(in JS and Wasm) and vice-versa, while the manually written code found in `components/script/dom/bindings` consists of utilities used across Servo's `script` crate that make it easier to integrate the Web Platform with SpiderMonkey. In particular, integration with the SpiderMonkey garbage collector is an important concern, which is best dealt with through shared utilities to avoid programming mistakes. Servo's large set of utilities and code generation capabilities result from the initial investment made by Mozilla. 

One issue with these capabilities, probably because it was unthinkable within Mozilla to use anything but SpiderMonkey, is their tight-coupling to SpiderMonkey. Another issue is that, despite the utilities covering much ground, the Web Platform code in Servo's script crate often still uses the low-level SpiderMonkey API directly(via `js::jsapi`)--defeating some of the benefits in terms of avoiding programming errors provided by the more general utilities, and resulting in further tight-coupling. 

## The problem of a lack of modularity

Servo's implementation of the Web platform is peppered with unsafe calls to `js::jsapi`: unsafe, non-idiomatic, and tightly-coupled to the SpiderMonkey API. The utilities and generated code are also tightly-coupled, but that is at least hidden from their point of use in the rest of the `script` crate--making it easier to reduce and eventually remove that coupling.

On the bright side, we have: 

- An existing Web standard that defines a clean interface between script execution and the rest of the Web platform: [Web IDL](https://webidl.spec.whatwg.org/). 
- Some integration with SpiderMonkey that is safe, idiomatic, and hides the SpiderMonkey API: see `js::rust` and the various utilities in `components/script/dom/bindings`. 
- Large part of the integration with SpiderMonkey that is generated automatically, through Rust code generated by a Python script found in `components/script/dom/bindings/codegen`.
- Servo's integration with SpiderMonkey being the only "browser grade"(in terms of security and features) integration with a script execution engine in Rust land. It can be compare with the problem experienced by the [Deno project](https://deno.com/) with their own script integration: for example the absense of [host objects](https://github.com/denoland/deno/issues/11118), or the challenges encountered to support important features such as [ExternalArrayBuffer](https://github.com/denoland/deno/issues/9799).

## The solution to the problem of a lack of modularity

### Prior art
An existing effort to provide a common interface to all JS engines is React-Native's [JavaScript Interface (JSI)](https://github.com/facebook/react-native/blob/d97741af60e3af598d71894800d928b61c5782b4/packages/react-native/ReactCommon/jsi/jsi/jsi.h), which is [described as](https://reactnative.dev/blog/2021/10/26/toward-hermes-being-the-default#pioneering-at-vertical-integrations) "a lightweight, general-purposed API for embedding a JavaScript engine into a C++ program". It allows React-Native to run JS and call into JS engines across platforms: using Facebook's [Hermes](https://reactnative.dev/blog/2021/10/26/toward-hermes-being-the-default) for Android and Apple's JavascriptCore on IOS. It appears to be the only existing effort to implement such an interface. 

### Long-term
In an ideal world, Servo should have a clear separation of concerns between the Web platform and the execution of script, integrated by way of a generic interface. 
The idea is a Web IDL layer implementation with a "bring your own JS engine" approach, by way of an interface that any bindings layer could implement. This would open the door for Servo to use other engines to execute script. Besides existing engines used in other browsers, this would also enable Wasm-based runtimes such as [Wasmtime](https://github.com/bytecodealliance/wasmtime), as well as future runtimes not invented yet. But, it is impossible to start formulating what such a layer would look like, until we have cleaned-up the `components/script` crate to hide the use of SpiderMonkey APIs behind safe and idiomatic Rust. This brings us to the short-term--or rather: continuous--approach we are pursuing. 

### Short-term
A clean-up of script consists of the following: 

- Hide `js::jsapi` behind safe and idiomatic Rust abstractions in `components/script/dom/bindings` or in `js::rust`. 
- Conform to Web IDL spec, and to how Web IDL is used in other specifications.

An example of this is the work replacing the use of `jsapi::JSObject`, a kind of wrapper around a pointer to an untyped JS object, with higher-level and typed concepts. A recently completed part of this work was related to Servo's implementation of WebGPU: The [previous code](https://github.com/servo/servo/blob/5c1723c9833c133e1af641533293e63d8723f8d3/components/script/dom/gpubuffer.rs#L288) would return a `JSObject` from a Web platform operation to SpiderMonkey, after having obtained this object [using a low-level and unsafe SpiderMonkey API call](https://github.com/servo/servo/blob/5c1723c9833c133e1af641533293e63d8723f8d3/components/script/dom/gpubuffer.rs#L321). The [current code](https://github.com/servo/servo/blob/9be989146d5b958cafcc930385e63595a885cb20/components/script/dom/gpubuffer.rs#L289) works instead with [higher-level and safe concepts](https://github.com/servo/servo/blob/9be989146d5b958cafcc930385e63595a885cb20/components/script/dom/gpubuffer.rs#L320) hiding the use of low-level and unsafe SpiderMonkey APIs. The result is code that is easier to use and contribute to, because it does not require fiddling with unsafe and SpiderMonkey specific concepts, that is less coupled to SpiderMonkey, because the coupling is [centralized](https://github.com/servo/servo/blob/c94d5842db11330ff32b0dd2b8ce036c53b410bb/components/script/dom/bindings/typedarrays.rs#L164) in one place and hidden from the rest, and that is closer to the WebGPU and Web IDL specs, because [they deal](https://gpuweb.github.io/gpuweb/#ref-for-dom-gpubuffer-getmappedrange) not with an object but with an ArrayBuffer(Web IDL only rarely deals with objects as opposed to more specific types).

As this work continues, an increasing amount of SpiderMonkey specifics will be moved to `components/script/dom/bindings`, which will make it possible to enumerate what we are doing with our script engine, and to start investigating how this could be expressed through a generic interface: one that any bindings layer to a script execution engine could implement to integrate with Servo. 