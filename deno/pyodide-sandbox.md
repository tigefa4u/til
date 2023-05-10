# Running Python code in a Pyodide sandbox via Deno

I continue to seek a solution to the Python sandbox problem. I want to run an untrusted piece of Python code in a sandbox, with limits on memory and time.

Previous attempt: [Run Python code in a WebAssembly sandbox](https://til.simonwillison.net/webassembly/python-in-a-wasm-sandbox)

Today I came across [this comment](https://github.com/pyodide/pyodide/issues/3420#issuecomment-1542742906) on the Pyodide issue thread about Deno support, which showed how Pyodide can now be loaded in a Deno script using `import pyodideModule from "npm:pyodide/pyodide.js"`.

I decided to see if I could get a solution working where Python ran a Deno script in using `subprocess`, where the Deno script accepted code sent to stdin, executed it using Pyodide and returned the result to stdout.

Here's what I came up with.

## runner.js

This file, `runner.js`, contains a simple Deno script that can execute Python code sent to it on standard input using Pyodide:
```javascript
import pyodideModule from "npm:pyodide/pyodide.js";
import { readLines } from "https://deno.land/std@0.186.0/io/mod.ts";

const pyodide = await pyodideModule.loadPyodide();

for await (const line of readLines(Deno.stdin)) {
    let input;
    try {
        input = JSON.parse(line);
    } catch (error) {
        console.log(JSON.stringify({ error: "Invalid JSON input: " + error.message }));
        continue;
    }

    if (typeof input !== 'object' || input === null) {
        console.log(JSON.stringify({ error: "Input is not a JSON object" }));
        continue;
    }

    if (input.shutdown) {
        break;
    }

    let output;
    try {
        const result = await pyodide.runPythonAsync(input.code || "");
        output = JSON.stringify({ output: result });
    } catch (error) {
        output = JSON.stringify({ error: error.message.trim().split('\n').pop() || ''});
    }
    console.log(output);
}
```
It accepts a JSON object sent to standard input. That object should look like this:
```json
{"code": "4 + 5"}
```
You can also send `{"shutdown": true}` to shut down the Deno process.

The code will be executed using Pyodide (safely, in a WebAssembly sandbox) and the result will be returned as a JSON object on standard output:
```json
{"output": 9}
```
If the code fails to execute, an error message will be returned instead:
```json
{"error": "ZeroDivisionError: division by zero"}
```

### Running that script

```
deno run --allow-read runner.js
```
Paste the following into the terminal and press enter:
```json
{"code": "4 + 5"}
```
Output:
```json
{"output":8}
```

## Calling it from Python

Here's a module I created called `deno_run.py` which can manage the `deno` process:

```python
import subprocess
import json

# The Deno subprocess
deno_process = None

def run_code(string_of_code):
    global deno_process

    try:
        # If the Deno subprocess is not running, start it
        if deno_process is None or deno_process.poll() is not None:
            deno_process = subprocess.Popen(
                ['deno', 'run', '--allow-read', 'runner.js'],
                stdin=subprocess.PIPE,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            )

        # Send the code to the Deno subprocess
        deno_process.stdin.write(json.dumps({"code": string_of_code}).encode())
        deno_process.stdin.write('\n'.encode())
        deno_process.stdin.flush()

        # Read the result from the Deno subprocess
        output = deno_process.stdout.readline().decode()
        return json.loads(output)
    except Exception as e:
        # If the subprocess crashes, return an error message
        return {"error": str(e)}
```

This works! I tested it in a `python` REPL:
```pycon
>>> import deno_run
>>> deno_run.run_code("4 + 5")
{'output': 9}
```
## Next steps: timeouts and memory limits

I haven't implemented this yet, but for memory limits I plan to use the following:

```bash
deno run --v8-flags='--max-heap-size=20' ...
```
This should set the maximum heap size to 20MB. The Deno process will crash if that is exceeded, which is why my Python code restarts the process if it is not running.

I haven't decided on an approach to timeouts yet. It will probably be in the JavaScript itself and look something like this (suggested by GPT-4):

```javascript
    let result = '';
    const promise = pyodide.runPythonAsync(pythonCode).then(res => {
      result = res.toString();
    });

    // If the Python code takes too long to execute, reject the promise
    const timeout = new Promise((resolve, reject) => {
      const id = setTimeout(() => {
        clearTimeout(id);
        reject(new Error("Timed out"));
      }, timeLimit);
    });

    try {
      await Promise.race([promise, timeout]);
      response.body = JSON.stringify({ result });
    } catch (error) {
      response.status = 500;
      response.body = JSON.stringify({ error: error.message });
    }
```