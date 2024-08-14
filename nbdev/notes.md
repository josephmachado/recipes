
Installation steps:

```bash
# create a venv and activate it
pip install jupyterlab
pip install nbdev
nbdev_install_quarto
pip install jupyterlab-quarto
nbdev_install_hooks
nbdev_new
```

For documentation only site

```bash
rm setup.py .github/workflows/test.yaml nbs/00_core.ipynb
rm -rf <lib_path> #  (this will be the lib_path field in settings.ini):
```

Open jupyter lab with

```bash
jupyter lab
```
