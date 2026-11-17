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

## 7. Morning brief diario (agregado)

Un solo documento cada mañana, `manager/brief_AAAA-MM-DD.md` (y aviso por mail/push), ordenado por urgencia:

1. **Hoy hay que hacer** (máx. 5 ítems, con el comando exacto): alertas abiertas, aprobaciones pendientes de propuestas,
   reentrenos/WFO que vencen, cuentas cerca de meta o de límite, eventos del día (apertura de semana, medianoche del bróker).
2. **Estado de las 4 cuentas**: equity, DD trailing y diario, riesgo abierto, distancia a la meta de fondeo.
3. **Decisiones de Argos de ayer y su resultado** (ver §8) y **desacuerdos** con lo que hiciste vos.
4. **Propuestas nuevas** (pausar / rebalancear / reemplazar / mult de riesgo) con la evidencia y el nivel de confianza.
5. **Resultados de las fases B–E** resumidos (drift, WFO, candidatas, fondeo).

## 8. Test paralelo: Argos (sandbox) vs. vos (en vivo)

Objetivo: durante un período X comparar, con las mismas reglas y los mismos datos, **las decisiones que tomás vos en vivo**
contra **las que tomaría Argos (Claude Code)**, y auditar curva de equity, aciertos y fallos. Argos **no opera nada** en esta
etapa: decide en un sandbox y se simula.

**Diseño (champion / challenger, "shadow mode").**
- *Cartera en vivo* = la real (`sizing_config_dynamic.json` de cada cuenta), con tus decisiones.
- *Cartera Argos* = copia virtual (`manager/sandbox/argos_cartera.json`): arranca idéntica a la en vivo el día 0 y a partir
  de ahí aplica solo lo que Argos decide cada mañana (pesos, pausas, reemplazos, mult).
- Cada día, con los trades reales cerrados (ledger) y las series por estrategia (`portfolio_sim_series/`,
  `simulador_cuentas.py`, `pesos_cartera.riesgo_efectivo`), se **recalcula la curva de equity de ambas carteras** con sus
  pesos respectivos: es la misma lógica de la solapa en vivo, aplicada a la cartera virtual.
- Contrafactual de las pausas/reemplazos: una estrategia que Argos pausa **sigue "en sombra"** (política P4 de
  `politicas_reentreno.py`) para medir qué se perdió o se evitó; un reemplazo se simula con el OOS de la candidata. Se
  marca como *estimado* (no ejecutable en vivo): esto es el punto más débil del test y se reporta con su margen.
- Mismas reglas de firma en ambas (DD 10 % trailing, 4 % diario, escalador, guardián) para que la comparación sea justa.

**Memoria persistida de decisiones** (`manager/memoria/decisiones.jsonl`, una línea por decisión, commiteada a git cada
día: el historial de git hace de sello de tiempo y evita reescribir el pasado):

```
{id, fecha, decisor: "argos"|"usuario", tipo: "pausar|reactivar|peso|reemplazo|mult|reentrenar|nada",
 objetivo: estrategia/cuenta, antes, despues, justificacion, evidencia (refs a archivos/cifras),
 nivel_confianza, horizonte_eval_dias, regla_nivel: "N1"|"N2"|"contexto"|"regimen",
 resultado: {estado: "pendiente|acierto|fallo|neutro", delta_R, delta_equity_pct, delta_dd, fecha_eval}}
```
- Tus decisiones las registrás vos (un comando `registrar_decision.py` o una planilla simple); si no, se infieren
  del cambio de `sizing_config_dynamic.json` / `drift_autopausa.json` entre dos exportes.
- Al cumplirse el horizonte (p. ej. 5 y 20 días hábiles), Argos **etiqueta el resultado** y escribe la lección
  (`manager/memoria/lecciones.md`: qué patrón falló/acertó). Esa memoria se **relee al principio de cada brief**, para
  que Argos no repita errores. Lo que escribe es auditable: nunca borra, solo agrega.

**Niveles de validación en el test** (los estás evaluando): N1 = criterio base; N2 = confirmación; más las capas de
**contexto** y **régimen**. Cada decisión de Argos lleva en `regla_nivel` qué capas la respaldaron. Así, al final del
período se puede medir **qué capa agrega valor**: aciertos de decisiones con N1 solo vs. N1+N2 vs. N1+N2+contexto/régimen.
Esto responde directamente a la duda de si vale la pena incorporar N2/contexto/régimen. Empezar con **N1 como regla
de decisión** y registrar N2/contexto/régimen como *información adicional* (no bloqueante); promoverlas a regla solo si el
test muestra mejora.

**Métricas de la comparación (período X, sugerido ≥ 30 días hábiles; ideal 60–90):**
- Curva de equity de ambas y diferencia acumulada; retorno, DD máximo, DD p95, Sharpe/Sortino, tiempo bajo el agua.
- Tasa de acierto de decisiones, por tipo y por capa (N1/N2/contexto/régimen); R medio por decisión.
- Desacuerdos usuario vs. Argos: en cuántos tuvo razón cada uno y de cuánto fue el delta.
- Proximidad a la meta de fondeo y `t50` de `calculadora_fondeo` en ambas carteras.
- Costo: cuántas decisiones/día y cuánto cambió la cartera (rotación).
- Honestidad estadística: con ~30 días y pocas decisiones la diferencia no es significativa; el informe final da
  intervalos (bootstrap de días) y dice "no concluyente" si no se separa del ruido.

**Criterio de promoción (se fija ANTES de empezar, para no ajustarlo a posteriori):** Argos pasa de sandbox a *propuesta
con tu OK en vivo* solo si, al cierre de X: (a) retorno ajustado por riesgo ≥ el tuyo, (b) DD máximo no peor, (c) cero
violaciones de límites duros, y (d) la ventaja se mantiene en la segunda mitad del período (no solo en una racha).

## 9. Qué cambia en la construcción (§5)

Se agregan: `manager/sandbox/` (cartera virtual + simulador diario `simular_sandbox.py`), `registrar_decision.py`,
`evaluar_decisiones.py` (etiqueta aciertos/fallos y arma la comparación de equity), y el template del brief. Fase 1 del
plan (§4) pasa a ser: **brief + sandbox Argos en sombra, sin tocar nada en vivo**; el auto-aplicar queda para después del
período X y de cumplir el criterio de promoción.

## 10. Pendiente de confirmar con vos

- Dónde están las reglas de la carpeta **`portfolio`** (topes por estrategia/símbolo, escala): no aparece en la raíz de
  Drive; veo `portfolio_snapshots/`, `portfolio_configs.json` y `portfolio_magic_numbers.json`. Pasame la ruta exacta.
- Dónde viven hoy N1, N2, contexto y régimen (¿`strategy_checklist.py`, `contexto_estudio.py`, `LEEME_REGIMEN_DC.md`?).
- Duración del período X y cómo querés registrar tus decisiones (comando o planilla).
- ¿Nube con puente por Drive, o tarea en la laptop con `claude -p`? (El sandbox necesita los ledger y series: en la
  laptop es directo.)
- Qué cuentas son reales y cuáles de challenge hoy (ver `accounts_conditions.json`) y los costos de reset/compra.
