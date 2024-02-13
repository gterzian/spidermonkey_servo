# Servo and SpiderMonkey


### Introduction

[Servo](https://github.com/servo/servo) is a Web engine--a piece of software implementing a host of standards which together form the Web platform--which relies on [SpiderMonkey](https://spidermonkey.dev/) for script execution(Javascript and Wasm). 
This report will provide an overview of Servo's integration of SpiderMonkey, and provide an outlook on improving the modularity of that integration. 

#### Script execution and the Web

SpiderMonkey in and of itself is unrelated to the Web platform; rather, it is an implementation of the [ECMAScript specification](https://tc39.es/ecma262/), as well as of Wasm related specifications such as the [Wasm JS interface](https://webassembly.github.io/spec/js-api/index.html), [core Wasm](https://www.w3.org/TR/wasm-core/), and the [Wasm Web API](https://www.w3.org/TR/wasm-web-api/). Those capabilities are then integrated into Servo, the engine implementing the Web platform as specified by core standards such as [HTML](https://html.spec.whatwg.org/), specifications providing various types of infrastructure shared across the Web platform, such as [Web IDL](https://webidl.spec.whatwg.org/), and a host of additional specifications providing peripheral capabilities to the Web platform, such as [WebGPU](https://gpuweb.github.io/gpuweb/). 

The integration of SpiderMonkey into Servo takes places at various levels of granularity, resulting in an API surface that does not map neatly to the various specifications which Servo and SpiderMonkey are meant to implement, and resulting in places in a tight-coupling between Servo's implementation of various Web platform APIs and SpiderMonkey. A good example of such tight-coupling is Servo's implementation of the [Streams standard](https://streams.spec.whatwg.org), which, while not being part of EcmaScript but rather part of the Web platform, relies on a implementation provided(and now [deprecated](https://spidermonkey.dev/blog/2022/01/14/newsletter-firefox-96-97.html)) by SpiderMonkey itself. 