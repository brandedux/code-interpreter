# Stateful code interpreter

The repository contains a template and modules for the code interpreter sandbox. It is based on the Jupyter server and implements the Jupyter Kernel messaging protocol. This allows for sharing context between code executions and improves support for plotting charts and other display-able data.

## Motivation

The code generated by LLMs is often split into code blocks, where each subsequent block references the previous one. This is a common pattern in Jupyter notebooks, where each cell can reference the variables and definitions from the previous cells. In the classical sandbox each code execution is independent and does not share the context with the previous executions.

This is suboptimal for a lot of Python use cases with LLMs. Especially GPT-3.5 and 4 expects it runs in a Jupyter Notebook environment. Even when ones tries to convince it otherwise. In practice, LLMs will generate code blocks which have references to previous code blocks. This becomes an issue if a user wants to execute each code block separately which often is the use case.

This new code interpreter template runs a Jupyter server inside the sandbox, which allows for sharing context between code executions.
Additionally, this new template also partly implements the [Jupyter Kernel messaging protocol](https://jupyter-client.readthedocs.io/en/latest/messaging.html). This means that, for example, support for plotting charts is now improved and we don't need to do hack-ish solutions like [in the current production version](https://github.com/e2b-dev/E2B/blob/main/sandboxes/code-interpreter/e2b_matplotlib_backend.py) of our code interpreter.

The current code interpreter allows to run Python code but each run share the context. That means that subsequent runs can reference to variables, definitions, etc from past code execution runs.

## Current state

Known limited in features such as:

- All executions share single kernel

We'll be updating this module as we gather more user feedback.

## Installation

### Python

```sh
pip install e2b_code_interpreter
```

### JavaScript

```sh
npm install e2b_code_interpreter
```

## Examples

### Minimal example with the sharing context

#### Python

```python
from e2b_code_interpreter import CodeInterpreterV2

with CodeInterpreterV2() as sandbox:
    sandbox.exec_cell("x = 1")

    result = sandbox.exec_cell("x+=1; x")
    print(result.result)  # outputs 2

```

#### JavaScript

```js
import { CodeInterpreterV2 } from 'e2b_code_interpreter'

const sandbox = await CodeInterpreterV2.create()
await sandbox.execPython('x = 1')

const result = await sandbox.execPython('x+=1; x')
console.log(result.output)  # outputs 2

await sandbox.close()
```

### Get charts and any display-able data

#### Python

```python
import base64
import io

from matplotlib import image as mpimg, pyplot as plt

from e2b_code_interpreter import CodeInterpreterV2

with CodeInterpreterV2() as sandbox:
    # you can install dependencies in "jupyter notebook style"
    sandbox.exec_cell("!pip install matplotlib")

    # plot random graph
    result = sandbox.exec_cell(
        """
    import matplotlib.pyplot as plt
    import numpy as np

    x = np.linspace(0, 20, 100)
    y = np.sin(x)

    plt.plot(x, y)
    plt.show()
    """
    )

    # there's your image
    image = result.display_data[0]["image/png"]

    # example how to show the image / prove it works
    i = base64.b64decode(image)
    i = io.BytesIO(i)
    i = mpimg.imread(i, format='PNG')

    plt.imshow(i, interpolation='nearest')
    plt.show()
```

#### JavaScript

```js
import { CodeInterpreterV2 } from "e2b_code_interpreter";

const sandbox = await CodeInterpreterV2.create();

const code = `
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 20, 100)
y = np.sin(x)

plt.plot(x, y)
plt.show()
`;

// you can install dependencies in "jupyter notebook style"
await sandbox.execPython("!pip install matplotlib");

const result = await sandbox.execPython(code);

// this contains the image data, you can e.g. save it to file or send to frontend
result.display_data[0]["image/png"];

await sandbox.close();
```

### Streaming code output

#### Python

```python
from e2b_code_interpreter import CodeInterpreterV2

code = """
import time

print("hello")
time.sleep(5)
print("world")
"""
with CodeInterpreterV2() as sandbox:
    sandbox.exec_cell(code, on_stdout=print, on_stderr=print)
```

#### JavaScript

```js
import { CodeInterpreterV2 } from "e2b_code_interpreter";

code = `
import time

print("hello")
time.sleep(5)
print("world")
`;

const sandbox = await CodeInterpreterV2.create();

await sandbox.execPython(
  code,
  (out) => console.log(out),
  (outErr) => console.error(outErr),
);
```

### Pre-installed Python packages inside the sandbox

The full and always up-to-date list can be found in the [`requirements.txt`](https://github.com/e2b-dev/E2B/blob/stateful-code-interpreter/sandboxes/code-interpreter-stateful/requirements.txt) file.

```text
# Jupyter server requirements
jupyter-server==2.13.0
ipykernel==6.29.3
ipython==8.22.2

# Other packages
aiohttp==3.9.3
beautifulsoup4==4.12.3
bokeh==3.3.4
gensim==4.3.2
imageio==2.34.0
joblib==1.3.2
librosa==0.10.1
matplotlib==3.8.3
nltk==3.8.1
numpy==1.26.4
opencv-python==4.9.0.80
openpyxl==3.1.2
pandas==1.5.3
plotly==5.19.0
pytest==8.1.0
python-docx==1.1.0
pytz==2024.1
requests==2.26.0
scikit-image==0.22.0
scikit-learn==1.4.1.post1
scipy==1.12.0
seaborn==0.13.2
soundfile==0.12.1
spacy==3.7.4
textblob==0.18.0
tornado==6.4
urllib3==1.26.7
xarray==2024.2.0
xlrd==2.0.1
```

### Custom template using Code Interpreter

The template requires custom setup. If you want to build your own custom template and use Code Interpreter, you need to do:
1. Copy `jupyter_server_config.py` and `start-up.sh` from this PR
2. Add following commands in your Dockerfile
```Dockerfile
# Installs jupyter server and kernel
RUN pip install jupyter-server ipykernel ipython
RUN ipython kernel install --name "python3" --user
# Copes jupyter server config
COPY ./jupyter_server_config.py /home/user/.jupyter/
# Setups jupyter server
COPY ./start-up.sh /home/user/.jupyter/
RUN chmod +x /home/user/.jupyter/start-up.sh
```
3. Add the following option `-c "/home/user/.jupyter/start-up.sh"` to `e2b template build` command or add this line to your `e2b.toml`. 
```yaml
start_cmd = "/home/user/.jupyter/start-up.sh"
```  
