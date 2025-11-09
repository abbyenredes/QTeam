# Modelo y gráficos mejorados para VQE F98Y

Este documento complementa la Celda 7 del notebook `S-aureus-F98Y.ipynb`.
Incluye:
- Explicación breve de por qué cambian los gráficos en cada ejecución.
- Código para hacer la ejecución reproducible (misma salida cada vez).
- Un “modelo nuevo” de energía continua (MJ-Ising continuo) para suavizar el paisaje de optimización.
- Gráficos mejorados: superposición de convergencias y comparación avanzada.

## 1) Por qué cambian los gráficos
- Inicialización aleatoria: la celda crea `x0 = np.random.randn(...) * 0.5`, que varía entre ejecuciones. Aunque el notebook fija `np.random.seed(42)` en otra celda, el estado del RNG avanza con cada uso y no se resetea dentro de la Celda 7.
- Mínimos locales: el uso de `sign(tanh(params))` discretiza los parámetros y convierte el problema en no convexo con múltiples mínimos locales; cambios sutiles en el punto inicial llevan a óptimos diferentes.
- Hardware real (si se usa): el ruido y los `shots` pueden introducir variabilidad adicional.

## 2) Hacer la ejecución reproducible
Pega este ajuste dentro de la definición de `ImprovedVQE.run` en tu Celda 7; añade un parámetro `seed` y usa un RNG local:

```python
class ImprovedVQE:
    ...
    def run(self, hamiltonian_terms, sequence, mj_potential, max_iter=50, seed=42):
        """Ejecuta VQE con Hamiltoniano real, de forma reproducible"""
        print(f"Iniciando VQE MEJORADO...")
        print(f"   Secuencia: {sequence}")
        print(f"   Terminos Hamiltoniano: {len(hamiltonian_terms)}")
        print(f"   Backend: {self.backend.name if hasattr(self.backend, 'name') else 'Simulator'}")
        print()

        start_time = time.time()
        n_residues = len(sequence)

        # RNG local con semilla fija
        rng = np.random.default_rng(seed)
        x0 = rng.normal(0, 0.5, size=n_residues)

        optimizer = COBYLA(maxiter=max_iter)
        print("Optimizando con Hamiltoniano MJ...")
        print("-" * 60)

        result = optimizer.minimize(
            fun=lambda p: self.evaluate_energy_with_hamiltonian(p, hamiltonian_terms, mj_potential),
            x0=x0
        )
        elapsed = time.time() - start_time
        ...
```

Al llamar `run(...)` para WT y Mutante, usa la misma semilla o distintas pero controladas:

```python
results_wt = vqe_wt.run(hamiltonian_wt['terms'], fragment_wt['sequence'], mj_potential, max_iter=50, seed=42)
results_mut = vqe_mut.run(hamiltonian_mut['terms'], fragment_mut['sequence'], mj_potential, max_iter=50, seed=42)
```

## 3) Modelo nuevo: MJ-Ising continuo
En lugar de discretizar con `sign(tanh)`, usa la magnetización continua `m = tanh(params)`. Esto suaviza el paisaje y tiende a dar trayectorias más estables.

Pega esta función en la clase y un método “runner” continuo:

```python
class ImprovedVQE:
    ...
    def evaluate_energy_continuous(self, params, hamiltonian_terms):
        """Energía continua: E = Σ J_ij * m_i * m_j con m = tanh(params)"""
        # Número de residuos según términos
        n_residues = len(set([t['residues'][0] for t in hamiltonian_terms] + 
                             [t['residues'][1] for t in hamiltonian_terms]))
        m = np.tanh(params[:n_residues])
        energy = 0.0
        for term in hamiltonian_terms:
            i, j = term['residues']
            J_ij = term['coefficient']
            if i < len(m) and j < len(m):
                energy += J_ij * m[i] * m[j]
        self.history.append(energy)
        return energy

    def run_continuous(self, hamiltonian_terms, sequence, max_iter=50, seed=123):
        """Optimiza energía continua (MJ-Ising continuo)"""
        print("Iniciando VQE CONTINUO (MJ-Ising)...")
        print(f"   Secuencia: {sequence}")
        print(f"   Términos: {len(hamiltonian_terms)}")
        print()
        start_time = time.time()
        n_residues = len(sequence)
        rng = np.random.default_rng(seed)
        x0 = rng.normal(0, 0.5, size=n_residues)
        optimizer = COBYLA(maxiter=max_iter)
        result = optimizer.minimize(
            fun=lambda p: self.evaluate_energy_continuous(p, hamiltonian_terms),
            x0=x0
        )
        elapsed = time.time() - start_time
        return { 'energy': result.fun, 'history': self.history, 'time': elapsed, 'iterations': self.iteration, 'params': result.x, 'success': True }
```

Luego ejecuta y guarda una comparación continua:

```python
vqe_wt_cont = ImprovedVQE(backend_real)
vqe_mut_cont = ImprovedVQE(backend_real)
res_wt_cont = vqe_wt_cont.run_continuous(hamiltonian_wt['terms'], fragment_wt['sequence'], max_iter=50, seed=7)
res_mut_cont = vqe_mut_cont.run_continuous(hamiltonian_mut['terms'], fragment_mut['sequence'], max_iter=50, seed=7)

delta_cont = res_mut_cont['energy'] - res_wt_cont['energy']

plt.figure(figsize=(8,6))
labels = ['WT (F98)', 'Mutante (Y98)']
x = np.arange(2)
width = 0.35
plt.bar(x - width/2, [res_wt_cont['energy'], res_mut_cont['energy']], width, color='#2ecc71', edgecolor='black', label='VQE continuo')
plt.bar(x + width/2, [energy_wt, energy_mut], width, color='#95a5a6', edgecolor='black', label='Clásico MJ')
plt.xticks(x, labels)
plt.ylabel('Energía (kT)')
plt.title('Comparación MJ clásico vs MJ-Ising continuo (F98Y)')
plt.legend()
plt.grid(axis='y', alpha=0.3)
plt.text(x[1], (energy_mut+res_mut_cont['energy'])/2, f'Δ_cont={delta_cont:+.2f}', ha='center', fontsize=10,
         bbox=dict(boxstyle='round', facecolor='yellow', alpha=0.6))
plt.tight_layout()
plt.savefig('vqe_continuous_comparison_f98y.png', dpi=150)
```

## 4) Gráficos mejorados
### 4.1 Superposición de convergencias WT vs Mutante
Superpone ambas curvas y añade un suavizado (media móvil) para visualizar tendencia:

```python
def smooth(y, w=5):
    if len(y) < w:
        return y
    return np.convolve(y, np.ones(w)/w, mode='valid')

fig, ax = plt.subplots(figsize=(10,6))
ax.plot(results_wt['history'], color='#2980b9', alpha=0.5, label='WT bruto')
ax.plot(results_mut['history'], color='#c0392b', alpha=0.5, label='Mutante bruto')
ax.plot(smooth(results_wt['history']), color='#1f618d', linewidth=2, label='WT suavizado')
ax.plot(smooth(results_mut['history']), color='#922b21', linewidth=2, label='Mutante suavizado')
ax.axhline(results_wt['energy'], color='#2980b9', linestyle='--', label=f'WT óptimo {results_wt['energy']:.2f} kT')
ax.axhline(results_mut['energy'], color='#c0392b', linestyle='--', label=f'Mut óptimo {results_mut['energy']:.2f} kT')
ax.set_xlabel('Iteración')
ax.set_ylabel('Energía (kT)')
ax.set_title('Convergencia VQE (WT vs Mutante)')
ax.grid(True, alpha=0.3)
ax.legend(loc='best')
plt.tight_layout()
plt.savefig('vqe_convergence_overlay_f98y.png', dpi=150)
```

### 4.2 Comparación avanzada (con anotaciones)
Unifica el estilo y añade anotaciones claras de diferencias:

```python
fig, ax = plt.subplots(figsize=(12,7))
labels = ['WT (F98)', 'Mutante (Y98)']
x = np.arange(2)
width = 0.3

b1 = ax.bar(x - width, [energy_wt, energy_mut], width, label='Clásico MJ', color='#7f8c8d', edgecolor='black')
b2 = ax.bar(x, [results_wt['energy'], results_mut['energy']], width, label='VQE (discreto)', color='#3498db', edgecolor='black')
b3 = ax.bar(x + width, [res_wt_cont['energy'], res_mut_cont['energy']], width, label='VQE (continuo)', color='#2ecc71', edgecolor='black')

ax.set_xticks(x)
ax.set_xticklabels(labels)
ax.set_ylabel('Energía (kT)')
ax.set_title('Comparación de métodos: MJ clásico vs VQE (discreto/continuo)')
ax.grid(axis='y', alpha=0.3)
ax.legend(loc='best')

# Anotar valores
for bars in (b1, b2, b3):
    for bar in bars:
        h = bar.get_height()
        ax.text(bar.get_x()+bar.get_width()/2., h, f'{h:.2f}', ha='center', va='bottom', fontsize=9)

# Anotar deltas
ax.text(x[1]+width, (res_mut_cont['energy']+res_wt_cont['energy'])/2,
        f'Δ_cont={delta_cont:+.2f}', ha='center', fontsize=10,
        bbox=dict(boxstyle='round', facecolor='yellow', alpha=0.6))

plt.tight_layout()
plt.savefig('vqe_comparison_enhanced_f98y.png', dpi=150)
```

## 5) Interpretación rápida
- Si las curvas suavizadas descienden y se estabilizan cerca de las líneas de óptimo, la optimización es exitosa.
- Si VQE continuo y VQE discreto concuerdan con MJ, hay alta consistencia; si VQE divergiera, revela correlaciones o configuraciones distintas.
- La anotación `Δ` te ayuda a ver la dirección y magnitud del cambio entre WT y Mutante para cada método.

## 6) Cómo usarlo
- Copia los bloques de código en celdas nuevas al final de `S-aureus-F98Y.ipynb`.
- Ejecuta primero las celdas que definen y corren los VQE (WT/Mutante), luego las celdas de visualización.
- Se generarán archivos PNG: `vqe_convergence_overlay_f98y.png`, `vqe_continuous_comparison_f98y.png`, `vqe_comparison_enhanced_f98y.png`.

Si prefieres, puedo integrar estos cambios directamente en el notebook para que todo quede automático y reproducible en una sola ejecución.