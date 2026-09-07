# User guide of Hyper-K Trigger Timing Distributor (HkTtd)

User guide of HK Trigger Timing Distributor

Site URL
https://hk-timing-module-development.github.io/ug-hkttd/

## Local preview

### Requirements

- Linux
- Python 3.14
- Python `venv` module
- Git

### Initial setup

The following steps are required only once.

Create a virtual environment in the repository directory:

```console
python3.14 -m venv .venv
```

Activate the virtual environment:

```console
source .venv/bin/activate
```

Install the required packages:

```console
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### Preview the documentation

Activate the virtual environment:

```console
source .venv/bin/activate
```

Start the MkDocs development server:

```console
mkdocs serve
```

Open the following URL in a web browser:

http://127.0.0.1:8000/

MkDocs automatically updates the website when the documentation files are
changed.

Press `Ctrl+C` to stop the server.

Leave the virtual environment after finishing your work:

```console
deactivate
```
