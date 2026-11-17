# Plan: Manager diario del Thinker (Claude Code, cada mañana)

Basado en lo que hay en Drive (`PYEABridge`): `monitor_gui.py`, `tabs_*.py`, `wfo_runner.py`, `calculadora_fondeo.py`,
`pesos_cartera.py`, `aplicar_pesos_sandbox.py`, `cuenta_total.py`, `politicas_reentreno.py`, `lista_reentreno.py`,
`reentreno_cola.py`, `chequeo_laptop.py`, `VIGILANCIA_LAPTOP_agente.md`, y los JSON (`accounts.json`,
`cuenta_total.json`, `strategy_enabled.json`, `strategy_versions.json`, `portfolio_configs.json`,
`wfo_pending_results.json`, `edge_check_results.json`).

## 0. Restricción de arquitectura (condiciona todo)

- Los servidores y MT5 corren en la **laptop Windows (LAPTOPMSI)**. El estado vivo (heartbeat, ledger,
  `sizing_config_dynamic.json`, DD) vive en `%LOCALAPPDATA%\PYEABridge_runtime`, **fuera de Drive**.
- Una sesión remota de Claude Code (nube) ve **solo Drive** (y este repo). No ve el runtime ni puede
  ejecutar nada en la laptop. Tampoco hay `numpy`/datos de mercado salvo que se instalen/lleguen por Drive.
- Por eso el diseño es **en tres tramos**: la laptop exporta estado → la nube piensa y propone → la laptop
  valida y aplica. Claude nunca toca el servidor directamente.

Alternativa más simple (si no insistís en la nube): Programador de tareas de Windows en la laptop que ejecute
`claude -p "seguí MANAGER_DIARIO.md"` a las 07:00. Tiene acceso total (runtime, Python, WFO) y se salta los
tramos 1 y 3. Costo: depende de que la laptop esté encendida y de dar permisos a Claude en esa máquina.
Recomiendo empezar por esta para validar la lógica, y migrar a nube si hace falta.

## 1. Flujo diario

| Hora | Dónde | Qué |
|---|---|---|
| 06:00 | Laptop (tarea programada) | `exportar_estado_diario.py` (nuevo, solo lectura): vuelca a Drive `manager/estado_AAAA-MM-DD.json` con equity/DD/pico/riesgo abierto por cuenta, ledger de las últimas 24 h, `sizing_config_dynamic.json` de cada cuenta, salida de `chequeo_laptop.py`, drift de `drift_liviano.py`, `drift_autopausa.json`, cola de `reentreno_cola.py`. |
| 07:00 | Nube (rutina Claude Code) | Lee el estado + JSON de Drive; corre las fases A–F; escribe `manager/reporte_AAAA-MM-DD.md` y `manager/propuesta_AAAA-MM-DD.json`; commitea ambos a este repo (histórico). |
| 07:30+ | Laptop | `aplicar_propuesta.py` (nuevo): valida la propuesta contra límites duros, hace backup (`backups/pre_manager_<fecha>/`), aplica con la misma vía que `aplicar_pesos_sandbox.py --aplicar` (el servidor relee en ≤2 s, no se reinicia). Fase 1: solo con tu OK (`--aplicar`). |

## 2. Fases del manager (qué decide Claude)

**A. Salud y reporte (siempre).** Replica `VIGILANCIA_LAPTOP_agente.md`: ALERTAS/AVISOS con el formato ya definido
(servidor vivo, heartbeat, DD trailing ≥5/7 %, DD día ≥2/3 %, guardián, cartera igual en las 4 cuentas, riesgo total
si abre todo > 8 %, fallback no validado). Reporte arriba de todo: qué requiere tu acción hoy.

**B. Drift y reentreno.** Con `drift_liviano`/`tabs_drift` y `politicas_reentreno.mejor_politica()`: por estrategia,
estado (OK / amarillo / rojo / ya autopausada) y la política que el sandbox dice que rinde más (congelar, reentrenar
cada W, pausa, reemplazo mismo símbolo/familia). Propone: pausar, reentrenar (encola en `reentreno_cola`), o reemplazar.
Respeta que **P1 está inflada por sesgo de selección** (la justa es P1r).

**C. WFO y candidatas.** Toma `wfo_pending_results.json` y `edge_check_results.json`: (1) descartadas que ahora
validan, (2) candidatas en `strategies/` aún no aplicadas, (3) las que mejoran el portfolio (`portfolio_candidato.py`,
`diversificacion.py`: correlación, aporte marginal a Sharpe/DD). Si hay que correr WFO nuevo, lo deja **encolado**
para la laptop (`wfo_runner.py` es de 238 KB y necesita datos de mercado locales; no se corre en la nube).
Criterio de validación = el de `strategy_checklist.py` / `checklist_nivel2.json`; Claude no inventa umbrales.

**D. Pesos y rebalanceo.** ERC / pesos del sandbox (`portfolio_snapshots/pesos_sandbox_todas.json`) como base.
Recalcula con el año anterior, DD p95 objetivo (hoy 5 %), y propone el nuevo `baseline_risk_pct_by_entry`.
Mantiene `manual_entries` (filas manuales) como hoy.

**E. Fondeo más rápido.** Usa `calculadora_fondeo.arbol()` con caminos bootstrap de la historia OOS de la cartera
aplicada y las reglas de `accounts_conditions.json`: por cuenta elige entre seguir / resetear / comprar nueva y el
`account_risk_mult` con menor `t50` sujeto a `p_quiebra ≤ 5 %`, `p_trabada ≤ 5 %` y peor día ≤ 85 % del límite diario.
Aplica el recorte de edge (`confianza` 0,57). Hoy los 4 mult están en ×1,25 (en `cuenta_total.json`).

**F. Cambios al código del Thinker.** Si un hallazgo requiere tocar `.py` (bug, columna nueva en el reporte), va como
**PR en este repo**, nunca editado directo en Drive; lo revisás y lo bajás vos.

## 3. Límites duros (los verifica `aplicar_propuesta.py`, no Claude)

- Riesgo total si abre todo ≤ 8 % por cuenta (el 2026-10-03 una escala mal puesta dio 16 %); hoy 6,4 %.
- Cambio de peso por estrategia ≤ ±25 % relativo por día y ≤ el máximo por entrada de `portfolio/`.
- `account_risk_mult` ≤ 1,5 y ≤ ±0,25 por día; nunca subir con DD trailing ≥ 5 % o DD día ≥ 2 %.
- No tocar posiciones abiertas, EA, `accounts.json`, ni reiniciar/parar el servidor (parar frena las 4 cuentas).
- Estrategia nueva entra con peso inicial reducido (p. ej. 50 % del ERC) y solo si pasa el checklist.
- Si falta el estado del día, o el export tiene > 12 h, o hay una ALERTA abierta: **solo reporte, sin propuesta**.
- Misma cartera en las 4 cuentas (modo Total): se aplica a la maestra y se replica con `cuenta_total.replicar()`.

## 4. Niveles de autonomía (subir de a poco)

1. **Semanas 1–2: solo reporte.** Sin propuesta de cambios. Comparás el reporte con lo que ves en el Thinker.
2. **Semanas 3–4: propuesta + tu OK.** Aplicás con `aplicar_propuesta.py --aplicar` tras leer el reporte.
3. **Después: auto-aplicar solo lo chico** (rebalanceos dentro de límites; pausas por drift rojo). Todo lo demás
   (estrategia nueva, subir mult, reemplazos, resets/compra de cuentas) sigue pidiendo OK.

## 5. Qué hay que construir

1. `exportar_estado_diario.py` + tarea programada en la laptop (solo lectura, ~1 día).
2. `aplicar_propuesta.py` con validación de límites, backup y `--dry-run` por defecto (~1–2 días).
3. `MANAGER_DIARIO.md` (instrucciones para la rutina, al estilo de `VIGILANCIA_LAPTOP_agente.md`): reglas, formato del
   reporte, esquema de `propuesta.json`.
4. Rutina programada de Claude Code (diaria, ~07:00 hora local) apuntando a este repo; lectura/escritura de Drive.
5. Reporte: el `.md` en Drive + aviso por mail/push con el resumen y las ALERTAS.

## 6. Riesgos y mitigaciones

- **Overfit del WFO** → recorte de edge, P1r en vez de P1, validación fuera de muestra, peso inicial reducido.
- **Estado viejo o corrupto** → chequeo de edad del export; sin estado no hay propuesta.
- **Doble aplicación** (Thinker abierto + manager) → el Thinker pisa pesos al Aplicar: cerrarlo antes, o que el manager
  escriba solo vía `manager_lock` que el Thinker respete. Ya pasó con la auto-pausa duplicada (ver `GUIA_THINKER_REMOTO.md`).
- **Credenciales**: `PYEA_runtime` contiene `email_alerts_config.json` (clave de app de Gmail); no exportar nunca eso a Drive.
- **Simulaciones ≠ realidad**: `calculadora_fondeo` no modela intradía, slippage ni swaps; la quiebra real puede ser mayor.

## 7. Pendiente de confirmar con vos

- Dónde están las reglas de la carpeta **`portfolio`** (topes por estrategia/símbolo, escala): no aparece en la raíz de
  Drive; veo `portfolio_snapshots/`, `portfolio_configs.json` y `portfolio_magic_numbers.json`. Pasame la ruta exacta.
- Carpeta `strategies/` (backtest validadas): ¿el criterio de "validada" es `checklist_nivel2.json`?
- ¿Nube con puente por Drive, o tarea en la laptop con `claude -p`?
- Qué cuentas son reales y cuáles de challenge hoy (ver `accounts_conditions.json`) y los costos de reset/compra.
