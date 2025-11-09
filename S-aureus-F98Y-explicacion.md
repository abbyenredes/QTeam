# Explicación del código (Celda 7) y análisis de resultados

Ruta de referencia del código: `S-aureus-F98Y.ipynb`, celda “CELDA 7 FINAL CORREGIDA: VQE REAL que usa el Hamiltoniano construido”.
Imágenes generadas: 
- `vqe_improved_convergence_f98y.png`
- `vqe_vs_classical_comparison_f98y.png`

## 1) Objetivo general
- Ejecutar un VQE (Variational Quantum Eigensolver) simplificado para comparar la energía del fragmento wildtype (F98) y del mutante (Y98) de la proteína DHFR de S. aureus.
- Usar un “Hamiltoniano” basado en el potencial de Miyazawa–Jernigan (MJ) como un modelo de interacción tipo Ising entre residuos.
- Comparar la energía “clásica” (MJ directo) vs la energía optimizada por VQE.

## 2) Flujo del código de la Celda 7
- Selección de backend: 
  - Si hay conexión a IBM Runtime y `USE_REAL_HARDWARE = True`, intenta elegir un backend real (menos ocupado, suficientes qubits).
  - Si no, usa `simulator` (Aer) como backend.
- Clase `ImprovedVQE`:
  - Guarda el backend, un historial de energías y cuenta de iteraciones.
  - `evaluate_energy_with_hamiltonian(params, hamiltonian_terms, mj_potential)`:
    - Convierte los parámetros continuos en “spins” discretos (`-1` o `+1`) con `sign(tanh(param))`.
    - Calcula la energía tipo Ising: `E = Σ J_ij * s_i * s_j` sobre los pares de residuos definidos por el Hamiltoniano MJ.
    - Registra la energía en `history` y reporta cada 10 iteraciones.
  - `run(hamiltonian_terms, sequence, mj_potential, max_iter=50)`:
    - Prepara un vector inicial `x0` aleatorio (`np.random.randn(n_residues)*0.5`).
    - Optimiza con `COBYLA` para minimizar la energía devuelta por `evaluate_energy_with_hamiltonian`.
    - Devuelve energía final, historial, tiempo, iteraciones y parámetros.
- Ejecución del VQE:
  - `results_wt` para wildtype (F98).
  - `results_mut` para mutante (Y98).
- Análisis y reporte:
  - Calcula `delta_vqe = results_mut.energy - results_wt.energy`.
  - Compara con `delta_energy` (la diferencia MJ clásica calculada en celdas previas).
  - Imprime interpretación (estabilizante/neutra/desestabilizante).
- Visualización:
  - Genera las dos figuras y las guarda como PNG.

## 3) Qué representa cada gráfico
- `vqe_improved_convergence_f98y.png` (convergencia):
  - Muestra el historial de energía por iteración del VQE para WT y para el mutante (cada uno en su subplot).
  - La línea horizontal discontinua marca la energía final alcanzada tras la optimización.
  - Interpretación: un descenso sostenido indica que el optimizador va encontrando configuraciones de “spins” que reducen la energía del Hamiltoniano MJ.
- `vqe_vs_classical_comparison_f98y.png` (comparación):
  - Barras para WT y Mutante con: 
    - Energía clásica MJ (barras grises).
    - Energía VQE optimizada (barras azules).
    - Una barra roja de “Delta clásico” en el segundo grupo para visualizar el cambio entre WT y Mutante.
  - Interpretación: 
    - Si la barra VQE del mutante es mayor que la del WT, el mutante aparece menos estable bajo este modelo; si es menor, más estable.
    - La etiqueta con `Δ` resume la diferencia entre mutante y wildtype.

## 4) Por qué los gráficos cambian en cada ejecución
Incluso usando el simulador, los resultados pueden variar por:
- Inicialización aleatoria de parámetros:
  - En `run()`, `x0 = np.random.randn(n_residues)*0.5` parte de valores aleatorios distintos en cada ejecución. 
  - Aunque en otra celda se fija `np.random.seed(42)`, el generador de números aleatorios avanza su estado cada vez que se usa. Si re-ejecutas la celda 7 sola, el estado ya no es el mismo y se obtiene un `x0` diferente, llevando a trayectorias de optimización distintas y mínimos locales diferentes.
- Paisaje no convexo y mínimos locales:
  - La función objetivo (Ising con `sign(tanh)`) es altamente no convexa y por troceo puede generar múltiples mínimos locales. Pequeñas variaciones iniciales llevan a soluciones diferentes.
- Opcionalmente, ejecución en hardware real:
  - Si se activa IBM Runtime, el ruido y el tamaño finito de `shots` introducen variabilidad adicional. En el documento actual, la energía se calcula de forma determinista en Python, pero cualquier medición real añadiría fluctuaciones.

## 5) Cómo hacer los resultados reproducibles
Si quieres que los gráficos no cambien entre ejecuciones, aplica estas prácticas:
- Fijar la semilla justo antes de crear `x0` dentro de `run()`:
  ```python
  rng = np.random.default_rng(42)
  x0 = rng.normal(0, 0.5, size=n_residues)
  ```
  Así, cada ejecución comienza en el mismo punto.
- Mantener constantes hiperparámetros del optimizador:
  - `COBYLA(maxiter=50, rhobeg=0.5, tol=1e-6)` (por ejemplo) y no cambiarlos entre corridas.
- Evitar fuentes de aleatoriedad externas:
  - Usar solo el simulador (Aer) si deseas resultados exactamente repetibles.
  - Si más adelante añades mediciones cuánticas, fija `shots` y `seed_simulator` en Aer.

## 6) Notas y recomendaciones técnicas
- El “Hamiltoniano MJ” aquí se trata como un modelo tipo Ising de interacción entre residuos. Es un proxy útil para comparar configuraciones, pero no es una simulación química ab initio.
- El mapeo `sign(tanh(params))` es una forma rápida de llevar parámetros continuos a spins discretos; puede producir saltos bruscos y múltiples mínimos locales. Si quieres suavizar el paisaje, podrías usar una energía continua sin discretización temprana.
- Para extender a un VQE “cuántico real”, se necesitaría construir un operador de Fermión/Pauli del sistema y medir expectativas con `Estimator` sobre un ansatz (por ejemplo `EfficientSU2`), lo cual es más caro pero físicamente más cercano al VQE convencional.

## 7) Lectura rápida de resultados típicos
- Convergencia: 
  - Espera curvas descendentes y luego estabilización. Si oscilan mucho, aumenta `max_iter` o ajusta `rhobeg`/`tol`.
- Comparación: 
  - Observa las diferencias entre barras “Clásico” y “VQE”. Si ambas concuerdan (diferencias pequeñas), tu modelo discreto y el optimizador están alineados; si difieren, el VQE encuentra configuraciones de spins con correlaciones distintas.

---
Si te gustaría, puedo aplicar un pequeño ajuste en la notebook para fijar la semilla dentro de `ImprovedVQE.run()` y dejar los resultados completamente reproducibles.