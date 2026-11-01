*make venv*

python3 -m venv .venv

*activate venv*

source .venv/bin/activate

*setup venv pip (third-party library)*

python -m pip install --upgrade pip

*exit venv*

deactivate


*suggest tools*

python -m pip install numpy scipy pandas matplotlib sympy
python ... jupyterlab ipykernel

*open jupyterlab*

jupyter lab




