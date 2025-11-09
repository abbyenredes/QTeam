# Quantum Protein Energy Analysis – *S. aureus F98Y*

Este proyecto implementa un flujo de trabajo cuántico-clásico para analizar la diferencia de energía entre variantes proteicas de *Staphylococcus aureus*, utilizando **Qiskit** y métodos como **Grover** y **VQE (Variational Quantum Eigensolver)**.

El notebook principal (`S-aureus-F98Y.ipynb`) guía el proceso desde la generación de mapas de contacto y cálculo de energías hasta la construcción del Hamiltoniano cuántico.

---

## 🧠 Tecnologías principales

- **Python 3.12**
- **Qiskit 2.2** y submódulos:
  - `qiskit-aer` para simulación cuántica  
  - `qiskit-algorithms` para Grover y VQE  
  - `qiskit-ibm-runtime` para integración con IBM Quantum  
- **NumPy** y **Matplotlib** para análisis y visualización
- **Jupyter Notebook** como entorno interactivo

---


## Start project:

Create the virtual environment with venv:

Windows
```bash
py -3.12 -m venv .venv
```
Linux/Mac
```textplain
python3.12 -m venv .venv
```

Start the virtual environment

Linux/Mac
```textplain
source .venv/bin/activate
```
Windows
```textplain
.venv/Scripts/activate
```
Install dependencies
```textplain
pip install -r requirements.txt
```
## 🧩 Ejecución

Abre el entorno de Jupyter:

```bash
jupyter notebook
```

Carga el archivo **S-aureus-F98Y.ipynb**.

Ejecuta las celdas en orden para generar los gráficos de energía, los Hamiltonianos cuánticos y los resultados de **Grover/VQE**.

---

## 📊 Resultados esperados

- Comparación de energía entre la proteína silvestre y la mutante (gráfico **ΔΔG en kT**).  
- Simulación cuántica del **Hamiltoniano molecular**.  
- Ejemplo de uso de **PhaseOracle**, **Grover** y **VQE** con **AerSimulator**.

---

## 🧾 Notas

- Este proyecto usa **Qiskit ≥ 2.2**; por tanto, clases como `QuantumInstance` o `opflow` ya no están disponibles.  
- Si encuentras advertencias de deprecación (`DeprecationWarning`), revisa la documentación de Qiskit para las nuevas clases equivalentes (`PhaseOracleGate`, `SparsePauliOp`, etc.).

```textplain
pip install -r requirements.txt
```
