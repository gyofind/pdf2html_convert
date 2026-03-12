# Architecture Document: PDF to HTML Converter

This document outlines the core architectural principles and patterns used in the PDF to HTML Converter browser extension. Its purpose is to guide development and ensure long-term maintainability and extensibility.

## Core Philosophy: Modularity and Separation of Concerns

The extension is built using a **Feature-Sliced Architecture**. Logic is organized by domain (e.g., `engines`, `services`, `ui`) rather than by technical type (e.g., putting all classes in one folder). This makes the codebase easier to navigate, test, and refactor.

## The Conversion Engine: A Strategy Pattern

To support multiple PDF-to-HTML conversion tools, we use the **Strategy design pattern**.

1.  **`ConversionManager` (`/src/services/ConversionManager.js`)**: This is the central orchestrator. It is responsible for selecting the user's preferred conversion engine and executing it. It does not know the implementation details of any specific engine.

2.  **`EngineInterface` (`/src/engines/interface.js`)**: This file defines the "contract" that every engine must follow. It will export a class or object structure that guarantees a consistent API, for example:

    ```javascript
    // /src/engines/interface.js (Conceptual)
    export class EngineInterface {
      constructor(pdfBuffer) {
        if (this.constructor === EngineInterface) {
          throw new Error("Cannot instantiate abstract class EngineInterface.");
        }
        this.pdfBuffer = pdfBuffer;
      }

      async convert() {
        throw new Error("Method 'convert()' must be implemented.");
      }
    }
    ```

3.  **Concrete Strategies (`/src/engines/*`)**: Each subdirectory within `/src/engines` is a self-contained strategy that implements the `EngineInterface`. The default is `pdfjs-engine`.

### Handling WASM vs. Native Binaries

A core project constraint is "Privacy-First/Local-Only." This prohibits the use of external servers for conversion. Tools like `pdftohtml` or `pdf2htmlEX` are native command-line binaries and **cannot be run directly** in a browser's sandboxed environment.

To incorporate such tools, they **must be compiled to WebAssembly (WASM)**.

*   **WASM Approach**: A future `wasm-pdftohtml-engine` would contain the JavaScript bindings (`index.js`) and the compiled `.wasm` file. The engine's `convert()` method would then instantiate the WASM module and process the PDF buffer entirely on the client-side, respecting our local-only constraint.
*   **Limitation**: This is a significant undertaking, as it requires a WASM-compatible build of the target tool. We will architect for this possibility but implement it only when a stable WASM version is available.

## Theming Layer and CSS Scoping

To allow users to inject custom CSS for theming the converted HTML, we must prevent those styles from "leaking" and affecting the host page or the extension's own UI (e.g., the popup).

The chosen strategy is to render the converted HTML inside a **Shadow DOM**.

*   **Mechanism**: The content script (`viewer-injector.js`) will create a host element on the page, and attach a shadow root to it (`host.attachShadow({ mode: 'open' })`). The entire HTML output from the conversion engine is then placed inside this shadow root.
*   **Benefit**: The Shadow DOM provides true CSS encapsulation. Styles defined inside the shadow root do not affect the main document, and styles from the main document (with few exceptions) do not affect the content inside. This creates a perfect, isolated container for our themed content. A `<style>` tag containing the user's custom CSS can be safely prepended to the shadow root.
