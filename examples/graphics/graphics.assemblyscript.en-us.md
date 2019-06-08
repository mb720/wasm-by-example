# Graphics

## Overview

As stated before, **WebAssembly is a great fit for computationally intensive tasks**. And even the official [AssemblyScript Documentation covers this](https://docs.assemblyscript.org/faq#is-webassembly-always-faster). For example, Tasks that involve things like big data, heavy logic with conditionals, or nested looping. Thus, generating / rendering graphics **can** get a significant speedup by moving these mentioned parts into WebAssembly. In this example, we will be generating 20x20 colored checkerboard images once per second, and displaying them on a [HTML5 Canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) using [Pixel Manipulation on the ImageData Object](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Pixel_manipulation_with_canvas).

---

## Implementation

As usual, let's get started with our `index.ts` file. You will notice here we grow our memory, as in order to pass back our pixel values into Javascript, we will write these values into Wasm Memory. That way, Javascript can read them later. Please be sure to read the comments in the following code examples, and be sure to follow links or look at previous examples if something does not make sense. Let's get into it:

```typescript
// Set up our memory
// By growing our Wasm Memory by 1 page (64KB)
memory.grow(1);

// Function to generate our checkerboard, pixel by pixel
export function generateCheckerBoard(
  darkValueRed: i32,
  darkValueGreen: i32,
  darkValueBlue: i32,
  lightValueRed: i32,
  lightValueGreen: i32,
  lightValueBlue: i32
): void {
  const checkerBoardSize = 20;

  // Since Linear memory is a 1 dimensional array, but we want a grid
  // we will be doing 2d to 1d mapping
  // https://softwareengineering.stackexchange.com/questions/212808/treating-a-1d-data-structure-as-2d-grid
  for (let x: i32 = 0; x < checkerBoardSize; x++) {
    for (let y: i32 = 0; y < checkerBoardSize; y++) {
      // Set our default case to be dark squares
      let isDarkSquare: boolean = true;

      // We should change our default case if
      // We are on an odd y
      if (y % 2 === 0) {
        isDarkSquare = false;
      }

      // Lastly, alternate on our x value
      if (x % 2 === 0) {
        isDarkSquare = !isDarkSquare;
      }

      // Now that we determined if we are dark or light,
      // Let's set our square value
      let squareValueRed = darkValueRed;
      let squareValueGreen = darkValueGreen;
      let squareValueBlue = darkValueBlue;
      if (!isDarkSquare) {
        squareValueRed = lightValueRed;
        squareValueGreen = lightValueGreen;
        squareValueBlue = lightValueBlue;
      }

      // Let's calculate our index, using our 2d -> 1d mapping.
      // And then multiple by 4, for each pixel property (r,g,b,a).
      let squareNumber = y * checkerBoardSize + x;
      let squareRgbaIndex = squareNumber * 4;

      // Finally store the values.
      store<u8>(squareRgbaIndex + 0, squareValueRed); // Red
      store<u8>(squareRgbaIndex + 1, squareValueGreen); // Green
      store<u8>(squareRgbaIndex + 2, squareValueBlue); // Blue
      store<u8>(squareRgbaIndex + 3, 255); // Alpha (Always opaque)
    }
  }
}
```

Next, we can compile the module following the [Hello World](/example-redirect?exampleName=hello-world) examples compilation process, replacing the appropriate file names.

Next, Let's load / instantiate the wasm module, `index.wasm` in a new `index.js` file. Again, we will follow the module instantiation from the [Hello World](/example-redirect?exampleName=hello-world) example. A lot of the logic here is expanding on the [WebAssembly Linear Memory Example](/example-redirect?exampleName=webassembly-linear-memory), but applying the learnings to a DOM API. The most important thing here is probably how we are copying out memory from Wasm, using `.slice` calls. Please see some reference links if things get confusing. Here is the `index.js` below!

```javascript
const runWasm = async () => {
  // Instantiate our wasm module
  const wasmModule = await wasmBrowserInstantiate(
    "/examples/graphics/demo/assemblyscript/index.wasm"
  );

  // Get our exports object, with all of our exported Wasm Properties
  const exports = wasmModule.instance.exports;

  // Get our memory object from the exports
  const memory = exports.memory;

  // Create a Uint8Array to give us access to Wasm Memory
  const wasmByteMemoryArray = new Uint8Array(memory.buffer);

  // Get our canvas element from our index.html
  const canvasElement = document.querySelector("canvas");

  // Set up Context and ImageData on the canvas
  const canvasContext = canvasElement.getContext("2d");
  const canvasImageData = canvasContext.createImageData(
    canvasElement.width,
    canvasElement.height
  );

  // Clear the canvas
  canvasContext.clearRect(0, 0, canvasElement.width, canvasElement.height);

  const getDarkValue = () => {
    return Math.floor(Math.random() * 100);
  };

  const getLightValue = () => {
    return Math.floor(Math.random() * 127) + 127;
  };

  const drawCheckerBoard = () => {
    const checkerBoardSize = 20;

    // Generate a new checkboard in wasm
    exports.generateCheckerBoard(
      getDarkValue(),
      getDarkValue(),
      getDarkValue(),
      getLightValue(),
      getLightValue(),
      getLightValue()
    );

    // Pull out the RGBA values from Wasm memory
    // 100 * 100 * 4 = checkboard max X * checkerboard max Y * number of pixel properties (R,G.B,A)
    const imageDataArray = wasmByteMemoryArray.slice(
      0,
      checkerBoardSize * checkerBoardSize * 4
    );

    // Set the values to the canvas image data
    canvasImageData.data.set(imageDataArray);

    // Clear the canvas
    canvasContext.clearRect(0, 0, canvasElement.width, canvasElement.height);

    // Place the new generated checkerboard onto the canvas
    canvasContext.putImageData(canvasImageData, 0, 0);
  };

  drawCheckerBoard();
  setInterval(() => {
    drawCheckerBoard();
  }, 1000);
};
runWasm();
```

Lastly, lets load our ES6 Module, `index.js` Javascript file in our `index.html`. And let's be sure to add a canvas element as well! **Random tip:** use the [image-rendering](https://css-tricks.com/almanac/properties/i/image-rendering/) property to display pixel art, and other "sharp" images correctly.

```html
<!-- Other HTML here. -->

<body>
  <canvas
    width="20"
    height="20"
    style="image-rendering: pixelated; width: 100%;"
  >
  </canvas>
</body>

<!-- Other HTML here. -->
```

And you should get something similar to the demo ([Source Code](/source-redirect?path=examples/graphics/demo/assemblyscript)) below!

---

## Demo

<iframe src="/examples/graphics/demo/assemblyscript/"></iframe>

Next, lets took a look at an example of implementing [Audio with WebAssembly](/example-redirect?exampleName=audio).