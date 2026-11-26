# Guía para cerrar el issue 008 (corrida piloto)

Estado al 2026-11-25: P3 ya verificó el lado de Colab (criterios 2 y 4 en la T4; ver la sección *Verificación en Colab* de `issues/008-corrida-piloto.md`). Falta:

| Criterio de 008 | Quién lo cierra | Paso |
|---|---|---|
| 1. MMAS completa 2 iteraciones con la aptitud real y escribe caché y feromona | **P1** | P1-3 |
| 2. Ninguna falta de memoria (o queda registrada con aptitud 0) | **P1** (4060) | P1-4 |
| 3. 40 evaluaciones con el presupuesto real caben en ≈2 h | **P1** mide, **P2** decide | P1-5, P2-2 |
| 4. Fusión de los JSON y huellas coherentes entre cuentas | **P3** (con un fragmento de **P2**) | P2-3, P3-2 |
| 5. Decisión formal: plan completo o contingencia | **P1 + P2** | Cierre |

**Orden:** P1 (pasos 1–5) y P2-1 en paralelo → P2-2 con los números de P1 → P2-3 y P3 → cierre por P1.

Si algo sale distinto de lo que dice aquí, **para**, copia el error completo y mándalo al grupo. Nadie cambia código salvo P2 en el paso P2-2.

Todo lo del piloto se escribe en `resultados_piloto/` (local) o `resultados_piloto/` de Drive. **Nunca** en `resultados/`, que es de la noche 1.

---

## P1: dueño de la RTX 4060 (≈40 min)

Laptop conectada a la corriente, modo *Máximo rendimiento*, sin juegos ni video abiertos (igual que en `issues/instrucciones-p1-003.md`).

### P1-1. Preparar

```powershell
git pull
```

```powershell
.venv\Scripts\Activate.ps1
```

```powershell
python -m pytest
```

Debe decir `passed` sin ningún `failed`. **Anota el commit:**

```powershell
git rev-parse HEAD
```

### P1-2. Limpiar pilotos anteriores

La caché reanuda en lugar de evaluar. Si quedó un piloto viejo de MMAS (por ejemplo el paso 8 opcional de 003), el piloto no evaluaría nada nuevo. Bórralo:

```powershell
Remove-Item -Recurse -Force resultados_piloto\en-la\real -ErrorAction SilentlyContinue
```

### P1-3. MMAS con la aptitud real, 2 iteraciones (criterio 1, ≈3–5 min)

```powershell
python buscar.py --metodo mmas --semilla 0 --par en-la --aptitud real --presupuesto-tokens minimo --hormigas 2 --iteraciones 2
```

Debe imprimir `Iteración 1: ...`, `Iteración 2: ...` y terminar con `Mejor configuración (...)`. Las aptitudes saldrán ≈0: **es lo esperado** con el presupuesto mínimo.

Comprueba que existen los dos archivos:

```powershell
Get-ChildItem resultados_piloto\en-la\real
```

Deben estar `cache_aptitud.json` y `feromona_mmas_semilla0.json`.

**Prueba de reanudación (1 min):** vuelve a lanzar **el mismo comando** de arriba. Debe decir `Evaluaciones nuevas en esta ejecución: 0` y terminar en segundos.

Lanza también la línea base y 3 candidatos aleatorios, para tener los tres métodos en la 4060:

```powershell
python buscar.py --metodo defecto --semilla 0 --par en-la --aptitud real --presupuesto-tokens minimo
```

```powershell
python buscar.py --metodo aleatoria --semilla 0 --par en-la --aptitud real --presupuesto-tokens minimo --presupuesto 3
```

Ambos deben terminar sin errores.

**Anota:** terminó sí/no, evaluaciones nuevas de MMAS, reanudación con 0 nuevas sí/no, y si algún resultado salió con estado distinto de `ok`.

### P1-4. Memoria con la configuración más grande admitida (criterio 2, ≈1 min)

La memoria por lote no depende del presupuesto, así que basta el mínimo. Prueba la configuración más grande que no supere el tope de parámetros:

```powershell
python evaluar.py --presupuesto minimo --d_model 512 --heads 8 --capas_codificador 5 --capas_decodificador 5 --multiplicador_ffn 4 --salida resultados_piloto/grande.json
```

- Si sale `Estado: rechazado_parametros`, repite bajando ambas capas a `4`, luego a `3`, hasta obtener `Estado: ok` o `sin_memoria`.
- Si sale `sin_memoria`: está bien para el criterio (queda registrado con aptitud 0), pero anótalo y avisa a P2.

**Anota:** capas usadas, `Estado`, `MEMORIA GPU PICO` y número de parámetros.

### P1-5. Tiempo con el presupuesto real de esa configuración grande (criterio 3, ≈3–6 min)

Mismo comando que la configuración que dio `ok` en P1-4, pero con `--presupuesto real`. Por ejemplo, si fueron 4 capas:

```powershell
python evaluar.py --presupuesto real --d_model 512 --heads 8 --capas_codificador 4 --capas_decodificador 4 --multiplicador_ffn 4 --salida resultados_piloto/grande_real.json
```

**Anota** `TIEMPO` total (T_grande) y `MEMORIA GPU PICO`.

**Cuenta:** la configuración por defecto tardó 126 s (issue 003). Calcula

> T_medio ≈ (126 + T_grande) / 2   y   40 × T_medio

- Si 40 × T_medio ≤ 7 200 s (2 h): **el presupuesto real sirve tal cual.**
- Si no: pásale los números a P2 (paso P2-2) antes de seguir.

### P1-6. Mandar al grupo

Pega esta tabla rellena en el grupo y avisa a P2:

| Dato | Valor |
|---|---|
| Commit |3d189afa298b20062a8ead9003fe2121dd8154b9|
| P1-3: MMAS 2×2 terminó / evaluaciones nuevas |si / 3|
| P1-3: reanudación con 0 nuevas (sí/no) |si|
| P1-3: defecto y aleatoria (3) terminaron (sí/no) |si|
| P1-3: estados distintos de `ok` |0|
| P1-4: capas / estado / parámetros / memoria pico |Capas: 4, Estado: ok, Parámetros: 33,521,664, Memoria Pico: 3,079 MiB asignados / 3,520 MiB reservados |
| P1-5: T_grande / memoria pico |183/3079=0.05943488|
| P1-5: 40 × T_medio (s) |1hora y 44min (T_medio=156s)|

---

## P2: MMAS y presupuesto (≈30 min)

### P2-1. Comprobar ρ y la razón τ_min/τ_max con la aptitud falsa (no necesita GPU)

Valores actuales (PRD): `rho = 0.2`, `razon_tau = 0.1` (es la `a` de τ_min = a · τ_max). Están en los parámetros por defecto de `buscar_mmas` en `traduccion/busqueda.py` (≈línea 345) y de `ejecutar_mmas` en `traduccion/mmas.py`.

El piloto real no sirve para ajustarlos: con el presupuesto mínimo todas las aptitudes son ≈0 y τ_max se degenera. Se ajustan con la aptitud falsa y el tamaño real de la noche 1 (8 × 5):

```powershell
python buscar.py --metodo mmas --semilla 0 --par en-la --aptitud falsa --salida resultados_piloto_falsa
```

```powershell
python buscar.py --metodo mmas --semilla 1 --par en-la --aptitud falsa --salida resultados_piloto_falsa
```

Revisa en la salida y en `resultados_piloto_falsa\en-la\falsa\feromona_mmas_semilla*.json`:

- `Evaluaciones nuevas de la corrida` = **40** en ambas semillas (si hay muchos caminos repetidos reutilizados, la feromona se está estancando).
- La mejor global sube entre la iteración 1 y la 5.
- En la iteración 5 ninguna capa tiene toda la feromona en un solo valor con el resto en τ_min desde la iteración 2 (estancamiento prematuro).

Si todo se cumple: **no se cambia nada** (recomendado; es lo que dice el PRD). Si no, prueba `rho = 0.1` o `razon_tau = 0.2`, cambia el valor por defecto en los dos archivos y repite.

### P2-2. Decidir el presupuesto (con los números de P1-5)

- Si 40 × T_medio ≤ 7 200 s: no se toca nada.
- Si no: baja `EPOCAS_PRESUPUESTO_REAL` en `traduccion/entrenamiento.py` (hoy 2) para que 40 × T_medio quepa en ≈2 h. Comprueba que el valor nuevo funciona con `python evaluar.py --verificar` y anota el cambio en el issue 008.

Si cambiaste algo en P2-1 o P2-2:

```powershell
python -m pytest
```

```powershell
git add traduccion
```

```powershell
git commit -m "Issue 008: ajuste de MMAS/presupuesto tras el piloto"
```

```powershell
git push
```

Pasa el **hash del commit** a P3 (será el `COMMIT` de la noche 1). Si no cambiaste nada, pásale el commit que anotó P1.

### P2-3. Un fragmento mínimo en tu cuenta de Colab (criterio 4 entre cuentas, ≈10 min)

Abre `notebooks/ejecutor_colab.ipynb` en Colab con GPU T4 (*Entorno de ejecución → Cambiar tipo → T4 GPU*). Asegúrate de tener el acceso directo a la carpeta compartida en tu Drive. En la primera celda pon:

```python
COMMIT = "<el que te dio P3; el mismo que usará P3>"
METODO = "aleatoria"
SEMILLA = 1                   # semilla 1: la 0 ya está en Drive y se reutilizaría de la caché
FRAGMENTO, FRAGMENTOS = 2, 3
APTITUD = "real"
PRESUPUESTO_TOKENS = "minimo"
PRESUPUESTO_EVALUACIONES = 9
```

*Entorno de ejecución → Ejecutar todo.* Debe terminar sin excepción y la última celda debe mostrar 3 evaluaciones guardadas con `Tesla T4` en la huella. Avisa a P3 cuando termine.

---

## P3: Colab y fusión (≈20 min)

### P3-1. Fijar el commit y probar la reanudación con un corte real

1. Pon en la primera celda el `COMMIT` acordado con P2 (el mismo que usará P2 en P2-3) y súbelo.
2. Configura el fragmento 1 de la semilla 1:

   ```python
   METODO = "aleatoria"
   SEMILLA = 1
   FRAGMENTO, FRAGMENTOS = 1, 3
   APTITUD = "real"
   PRESUPUESTO_TOKENS = "minimo"
   PRESUPUESTO_EVALUACIONES = 9
   ```

3. *Ejecutar todo.* Cuando la celda 6 (lanzamiento) muestre que terminó **la primera** evaluación y empezó la segunda: *Entorno de ejecución → Desconectar y borrar entorno de ejecución*.
4. Vuelve a conectar y *Ejecutar todo* sin cambiar nada.
5. Debe decir `nuevas en esta ejecución: 2` (no 3) y terminar con 3 evaluaciones guardadas.

**Anota** el resultado en la sección *Verificación en Colab* del issue 008, en la línea *Reanudación tras un corte real de Colab*, y marca el criterio correspondiente en el issue 006.

### P3-2. Fusionar fragmentos de dos cuentas (criterio 4)

Cuando P2 termine P2-3, en una celda nueva del notebook (tras las de configuración):

```python
!cd {CARPETA_REPO} && python fusionar.py "{RUTAS.resultados_piloto}/en-la/real/aleatoria_semilla1_fragmento*de3.json" --salida "{RUTAS.resultados_piloto}/en-la/real/aleatoria_semilla1.json"
```

Debe decir `Archivos: 2 | resultados fusionados: 6`, `Duplicados descartados: 0` y **no** mostrar `ATENCIÓN`. (Si P4 también corre el fragmento 3, serán 3 archivos y 9 resultados.)

**Anota** en el issue 008: archivos, resultados, duplicados y que las huellas de las dos cuentas coinciden.

---

## Cierre (P1, con el visto bueno de P2)

1. En `issues/008-corrida-piloto.md`, añade una sección `## Piloto en la RTX 4060 (P1, fecha)` con la tabla de P1-6 y la decisión de P2 (ρ, razón τ, presupuesto: sin cambios o qué cambió y por qué).
2. Marca `- [x]`:
   - Criterio 1, si P1-3 terminó y existen caché y feromona.
   - Criterio 2, si P1-4 y la verificación de P3 no tuvieron faltas de memoria (o quedaron con aptitud 0).
   - Criterio 3, escribiendo al lado `40 × T_medio = … s` y el presupuesto final.
   - Criterio 4, si la fusión de P3-2 no tuvo incoherencias.
   - Criterio 5, escribiendo la decisión:
     - **Plan completo** si los criterios 1–4 están marcados: noche 1 con MMAS + defecto en la 4060 y aleatoria en 3 fragmentos de Colab.
     - **Contingencia** (sin búsqueda aleatoria, solo MMAS frente a la línea base) si falló la parte de Colab (P2-3, P3-1 o P3-2) y no se arregla hoy.
3. Mueve el issue y súbelo:

```powershell
git mv issues/008-corrida-piloto.md issues/done/
```

```powershell
git add issues
```

```powershell
git commit -m "Issue 008 cierre"
```

```powershell
git push
```

4. Avisa al grupo con la decisión y el `COMMIT` de la noche 1. **No** subas `resultados_piloto/`.
