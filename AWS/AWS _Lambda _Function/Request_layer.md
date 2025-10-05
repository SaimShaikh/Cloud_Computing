
layer/ — contains the python/ folder structure and a build.sh script that runs the three commands you listed:

```
mkdir -p python
pip install --target ./python requests
zip -r requests-layer.zip python
```

1) mkdir -p python

- Creates a directory named python.

- -p makes the parent directories as needed and won’t error if it already exists.

- WHY: AWS Lambda layers expect the library files to be inside a top-level folder in the zip. For Python layers the conventional top-level folder is python/ so AWS can place the contents on sys.path for the function.

2) pip install --target ./python requests

- Tells pip to install the requests package (and its dependencies) into the ./python folder, not into the global or virtualenv site-packages.

- Result: ./python/requests/, ./python/urllib3/, ./python/certifi/, etc., plus .dist-info folders.

- WHY: When Lambda unzips a layer that contains a top-level python/ directory, the interpreter sees those packages automatically. This is how you give your Lambda function extra libraries without bundling them inside the function code each time.
