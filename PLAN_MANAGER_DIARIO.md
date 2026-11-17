# Plan: Manager diario del Thinker (Claude Code, cada mañana)

Basado en lo que hay en Drive (`PYEABridge`): `monitor_gui.py`, `tabs_*.py`, `wfo_runner.py`, `calculadora_fondeo.py`,
`pesos_cartera.py`, `aplicar_pesos_sandbox.py`, `cuenta_total.py`, `politicas_reentreno.py`, `lista_reentreno.py`,
`reentreno_cola.py`, `chequeo_laptop.py`, `VIGILANCIA_LAPTOP_agente.md`, y los JSON (`accounts.json`,
`cuenta_total.json`, `strategy_enabled.json`, `strategy_versions.json`, `portfolio_configs.json`,
`wfo_pending_results.json`, `edge_check_results.json`).

## ★ Prioridad: que sea rentable y sostenible (revisión 2026-11-17)

Esta sección manda sobre el resto del plan. Todo lo demás (manager, brief, replay, Argos) es herramienta; el objetivo es
**ingreso neto positivo, repetible y que no ponga en riesgo la plata de la familia**.

**Honestidad primero.** Ningún plan garantiza ganancia. Con cuentas de fondeo, lo que se arriesga son las cuotas y resets; lo
que se gana son los pagos. El negocio es rentable solo si, en el tiempo, **pagos cobrados > cuotas + resets + costos**
(Claude, luz, VPS, impuestos). El trabajo del plan es averiguar eso rápido y barato, y escalar solo con evidencia.

### P0. Situación real (lo que contó el usuario, 2026-11-17)
- **Presupuesto**: 50–100 USD/mes para cuotas + 20 USD/mes de Claude Pro. Expectativa inicial: fondear en 2–3 meses.
- **Lo que pasó en 2 meses**: +2 % las primeras 2 semanas, después ~40 días de pérdidas diarias chicas y constantes; hoy igual
  o peor que al inicio. En el camino: xeno_stokes y rough_vol entraron con PF ≥ 2 y resultaron **sin edge** (corregido en
  los tests); otras estrategias dejaron de validar; y un test del Thinker concluyó que convenía **dejar todas prendidas**.

**Corrección importante (2026-11-17): las 4 cuentas existen desde hace ~1 año**, no 2 meses; los 2 meses son los de la
versión actual del sistema. En un año ninguna se fondeó, y cta2 tuvo su mayor balance en abril en 100.060 (+0,06 % sobre el
inicial) y en octubre estaba en ~93.300. Esto cambia el diagnóstico:
- **Hay mucho más dato**: un año de historial real en las 4 terminales MT5 (el mapeo cuenta↔terminal ya se hizo y se
  verificó, ver `accounts_conditions.json`). El diagnóstico se hace sobre **el año entero, segmentado por versión del
  sistema** (fechas de `strategy_versions.json` / cambios de cartera), para ver si alguna etapa tuvo edge real.
- **Un año sin pasar ninguna de 4 es evidencia fuerte** de que lo operado hasta ahora no tuvo edge suficiente después de
  costos. No prueba que la versión actual falle, pero **sube la vara**: la versión nueva tiene que demostrar edge en el
  diagnóstico y en el replay antes de recibir más plata o más tiempo.
- **Lo ya pagado no vuelve** (costo hundido): las decisiones miran hacia adelante. Mientras las cuentas no cobren cuota mensual,
  mantenerlas vivas con riesgo bajo es gratis y conserva la opción; verificar si la firma tiene regla de **inactividad** (si
  hay que operar cada N días para no perderlas).
- **Puerta de decisión dura**: si el diagnóstico del año y el replay no muestran edge que pase y cobre, la recomendación es
  **frenar gastos** (cuotas, resets y, si hace falta, la suscripción), dejar las cuentas al mínimo de riesgo, y priorizar un
  ingreso estable. Volver a invertir solo con una estrategia que muestre edge fuera de muestra y con costos.

**Lectura de esto (hipótesis principal a comprobar, no certeza):** el patrón "muchas pérdidas chicas, todos los días" es
la firma típica de una cartera grande cuyo **edge promedio después de costos es ~0 o negativo**. Diversificar baja la
variabilidad, pero **no crea edge**: si cada estrategia en promedio no gana, juntar 80 da una curva suave que baja despacio
(spread, swaps, comisiones y slippage se cobran siempre). Que un test diga "mejor todas prendidas que sacar las que no
validan" es consistente con eso: si el criterio de validación no predice qué estrategia va a ganar (lo mismo que mostró
`diversificacion.py`: elegir "las mejores" no le gana al azar), sacar o dejar da casi igual, y el test elige la que más baja la
volatilidad. Además, PF 2 que no era real es el sesgo de selección que ya mide el Deflated Sharpe: de muchas pruebas, alguna
sale linda por azar.

**Consecuencias para el plan (cambian el orden):**
1. **Proteger lo ya pagado**: bajar el riesgo de las 4 cuentas a la mitad mientras dura el diagnóstico (no apagar todo:
   así se sigue midiendo). Cada cuenta viva es plata ya invertida; quemarla para "recuperar" es lo peor con este presupuesto.
2. **No comprar ni resetear** cuentas hasta que el diagnóstico dé verde. Con 50–100/mes, una cuota de 25k con promo (~115) o
   un reset se come 1–2 meses de presupuesto.
3. **Diagnóstico del año en vivo, segmentado por versión (Python, sin Claude)**, en este orden:
   - R realizado por estrategia y por familia vs. el OOS recortado al 57 %; con 2 meses hay pocos trades por estrategia, así
     que se mira **por familia y por categoría de símbolo**.
   - **Costos**: cuánto del resultado se fue en spread, swap y comisión (sospechosos: exóticos USDZAR/USDMXN/USDPLN/EURSEK y
     cripto alt, que tienen spreads y swaps altos).
   - **Ejecución**: diferencia entre el precio del backtest y el real; rechazos de SL del trailing (ya reportados en la vigilancia).
   - **¿Mala suerte o backtest inflado?** Con los caminos bootstrap de `calculadora_fondeo`, ¿qué probabilidad tenía el backtest
     de estar plano o abajo a los 60 días (y a los 365)? Si era < 5 %, el backtest está sobreestimado y hay que recortar más el edge.
4. **Cartera núcleo chica en vez de 80**: solo estrategias que pasen validación estricta (Deflated Sharpe + N1 con
   datos ≤ fecha), cuyo resultado en vivo no las contradiga, en **símbolos líquidos y baratos**; por defecto, **apagada** la que no
   valida. Revisar qué test dio "mejor todas prendidas" y si se hizo fuera de muestra y con costos.
5. **Expectativa realista**: un 10 % con un edge modesto y riesgo prudente suele llevar **4–8 meses**, no 2–3; correr para
   llegar antes es lo que quema cuentas. Usar el `t50` de la calculadora **con el edge ajustado por lo visto en vivo**.
6. **Claude con moderación** (plan Pro tiene límites): el brief diario y el diagnóstico los hace Python; Claude se usa para
   decidir en días con disparador y para revisar el diagnóstico, no para correr cosas que Python puede hacer.
7. **Cuando haya evidencia de edge**, la forma más barata de comprobarlo con plata real es una cuenta chica (1-Step Pro 5k
   ≈ 59, 10k ≈ 109 de lista) y el primer **pago cobrado**: eso es la prueba, no la curva.

### P7. Meta de ingreso: cuánto capital fondeado hace falta y en qué escalones
Gastos familiares: **~5.000 USD/mes** (alquiler, colegios); **mínimo para arreglarse: 2.500 USD/mes**.

**Cuenta honesta.** En fase fondeada, un ritmo **sostenible** (que no quema cuentas) es del orden de **1,5–3 % por mes** de
promedio, con meses negativos incluidos. Más que eso exige un riesgo que, con 10 % de pérdida máxima, rompe cuentas en el año.
Además alguna cuenta fondeada se va a perder, y hay que reponerla; por eso se pide ~1,5× de colchón de capital.

| Meta mensual neta | Capital fondeado a 2 %/mes | a 3 %/mes | Con colchón de reposición (~1,5×) |
|---|---|---|---|
| 2.500 (mínimo) | 125.000 | ~85.000 | **~150.000–200.000** |
| 5.000 (normal) | 250.000 | ~170.000 | **~300.000–400.000** |

- Las 4 cuentas actuales suman 275.000: **si se fondearan y rindieran 2 % sostenido, alcanzarían para el mínimo y se
  acercarían al normal**. El problema hoy no es tamaño: es que todavía no pasan (P0).
- cta3 cobra 90 %, las demás 100 %. Restar impuestos.
- Más de 200–300k en una sola firma concentra el riesgo de contraparte y puede chocar con su tope por persona: repartir en 2–3.

**Escalera de hitos (no saltear):**
1. **Diagnóstico verde** (P0/P2): hay edge real en vivo después de costos.
2. **Primer pago cobrado** (cualquier monto). Es la prueba de que el negocio funciona.
3. **Los pagos cubren las cuotas + Claude** (~120 USD/mes): el negocio se autofinancia.
4. **1.000 USD/mes durante 3 meses**: se reinvierte en cuentas (desde los pagos, no desde la plata de la casa).
5. **2.500 USD/mes durante 6 meses** + **fondo de emergencia de 3–6 meses del mínimo (7.500–15.000)**: recién acá se puede
   contar con esto para lo básico.
6. **5.000 USD/mes**: escalar repartiendo entre firmas.

**Plazos realistas, si el diagnóstico da verde:** fondear 4–8 meses, y llegar al hito 5 en el orden de **12–18 meses**. Hasta
entonces, el trading tiene que ser un **complemento**, no la fuente de la que dependen alquiler y colegios. Si hoy no hay otro
ingreso que cubra al menos el mínimo, conviene asegurar uno en paralelo: es lo que permite operar sin presión, y operar con
presión (subir riesgo para llegar a fin de mes) es la forma más común de quemar cuentas.

**En el brief diario**: barra de progreso por escalón, "meses del mínimo (2.500) cubiertos por pagos cobrados" y proyección
con el edge ajustado por lo visto en vivo (no con el del backtest).

### P1. La única métrica que importa: el libro del negocio
- `manager/negocio.csv`: cada cuota, reset, complemento y **cada pago cobrado** (fecha, cuenta, firma, monto, estado).
- El brief abre con: **resultado neto del mes y acumulado**, pagos pendientes/próximos, cuotas gastadas, y
  "meses de gastos familiares cubiertos". La curva de equity de una cuenta de fondeo no es plata hasta que se cobra.
- Métricas de negocio: *costo por cuenta fondeada*, *pago medio por cuenta fondeada*, *vida media de una cuenta fondeada*
  (cuántos ciclos de pago antes de quemarse), *retorno sobre cuotas* = pagos / cuotas. Rentable = retorno sobre cuotas > 1,5
  sostenido 6 meses (margen para la mala racha).

### P2. Antes de construir nada nuevo: ¿hay edge real en vivo?
- Lo que se ve en `accounts_conditions.json` (al 2026-10-16) es preocupante: **cta2 (100k) estaba en 93.304 con piso en
  90.060 (colchón ≈ 3,3 %)**, con el mayor balance en abril y casi plana desde septiembre. Siete meses sin avanzar en la cuenta
  más grande es una señal a tomar en serio, no ruido a ignorar.
- Primera tarea (1–2 días, Python puro sobre el ledger): **edge realizado en vivo vs. lo que prometía el backtest**, por cuenta
  y por estrategia: R por trade real vs. OOS, con el recorte del 57 %. Si lo realizado está muy por debajo del backtest
  recortado, el problema es de estrategia/ejecución, y ningún manager lo arregla: **primero corregir, después automatizar**.
- Lo mismo con el replay (§8b): su primer entregable no es Argos, es responder "¿la cartera, gestionada de forma simple
  (P0/P2), **pasa y cobra** con reglas y costos reales?". Si la respuesta es no, Argos no va a convertirla en sí.

### P3. Arreglos de seguridad urgentes (antes que cualquier mejora)
1. **Pérdida diaria**: el servidor trabaja con 4 % y guardián en 3,5 %; la firma (Pro 1-Step, cta2/3/4) dice **3 %**. Si es 3 %,
   un día malo quema la cuenta con el guardián "contento". Confirmar en el panel Metrix hoy y, si es 3 %, bajar el
   guardián a ~2,5 % en esas cuentas.
2. **cta2**: en `cuenta_total.json` está en ×1,25, cuando la documentación de octubre decía ×0,75 para ella. Con ~3 % de colchón,
   verificar hoy DD y colchón real; si sigue cerca del piso, bajar el riesgo de esa cuenta hasta que se recupere o decidir
   reset con el árbol de fondeo.
3. **Riesgo por idea ≤ 2 % del inicial** (regla Pro): validar que ninguna entrada, con el mult y entradas escalonadas, lo supere;
   violarlo puede anular un pago aunque la cuenta gane.

### P4. Sostenibilidad: no apostar todo a una sola carta
- **Las 4 cuentas con la misma cartera son, en la práctica, una sola apuesta**: una mala semana las quema juntas. Opciones a
  evaluar en el replay: carteras distintas por cuenta (menos correlación entre cuentas), o arranques/riesgos escalonados para
  que nunca estén todas en la misma fase de DD.
- **Riesgo de contraparte**: la firma puede cambiar reglas, demorar o negar pagos, o cerrar (pasó con varias en 2024–25).
  Mitigación: cobrar en cada ciclo apenas se pueda (en Pro, cobrar además reinicia el piso), no acumular ganancia sin retirar,
  y cuando haya evidencia de rentabilidad, repartir entre 2–3 firmas. Revisar el **tope de capital fondeado por persona** y
  las reglas sobre operar la misma estrategia en varias cuentas.
- **Fase fondeada = modo ingreso**: ya no se busca máximo retorno sino **pagos regulares y vida larga de la cuenta**. El
  riesgo se fija para que la probabilidad de quemar la cuenta en 12 meses sea baja (p. ej. < 10–15 %), aunque cobre menos por mes.
  Una cuenta que paga 2 % al mes durante 2 años vale mucho más que una que paga 6 % dos veces y se quema.

### P5. Reglas de plata (para proteger a la familia)
- **Presupuesto fijo de cuotas por mes**, que se pueda perder sin afectar gastos de la casa. Nunca comprar cuentas para
  "recuperar" una quemada fuera de ese presupuesto.
- **Stop del negocio**: si en 3 meses seguidos el resultado neto es negativo, o se quema el presupuesto trimestral, se
  frenan compras y se vuelve a investigación hasta que el replay y el edge en vivo den verde.
- **Destino de cada pago**, en este orden: (1) recuperar cuotas gastadas, (2) fondo de reserva de cuotas para 3 meses,
  (3) **fondo de emergencia familiar de 6 meses de gastos**, (4) recién después, ingreso para la familia.
- No depender del trading como **único** ingreso hasta tener **6+ meses seguidos de pagos** y el fondo de emergencia armado.
- Impuestos: separar un % de cada pago desde el primero (consultar con un contador cómo se declaran los pagos de la firma).

### P6. Orden de ejecución revisado (de lo que más plata protege/gana a lo que menos)
| # | Qué | Por qué | Esfuerzo |
|---|---|---|---|
| 0 | Bajar riesgo a la mitad y congelar compras/resets (P0) | proteger lo ya pagado | minutos |
| 1 | Seguridad P3 (pérdida diaria, cta2, riesgo por idea) | evita quemar cuentas por configuración | horas |
| 2 | Libro del negocio + brief mínimo (P1, §7 sin Argos) | ver si se gana plata de verdad | 1–2 días |
| 3 | Diagnóstico del año (por versión) + edge realizado vs. backtest (P0, P2) y cartera núcleo | saber si hay algo que escalar | 2–4 días |
| 4 | Replay con políticas simples P0/P2 + árbol de fondeo (§8b, §8c) | ¿pasa y cobra con costos reales? | 1–2 semanas |
| 5 | Matriz de construcción de cartera y cartera por cuenta (§8d, P4) | más pagos con menos quemas | 1 semana |
| 6 | Argos en el replay, comparado contra la mejor política fija | ¿agrega valor la IA? | 1–2 semanas |
| 7 | Argos en sombra en vivo (§8) y luego con tu OK | recién con evidencia | 1–3 meses |

**Puertas de decisión (go / no-go):** después del paso 3 y del 4 se decide con números si seguir, corregir estrategias o
frenar. Si en el paso 6 Argos no le gana a la mejor política fija, se usa la política fija (más simple y barata) y Argos queda
solo para el brief.

**Costo de la herramienta**: Claude decide solo en días con disparador; el resto lo hace Python. Su costo entra en el libro del
negocio como cualquier otro gasto.

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

## 8b. Fase 0: replay histórico de Argos (dic-2025 → hoy), antes del test en vivo

Idea: en vez de esperar meses para juzgar a Argos, **rebobinar al corte de noviembre de 2025** y dejarlo "vivir" día a día
hasta hoy sobre datos que ya pasaron, con el mismo ciclo que tendría en vivo. Da un año de evidencia en días de cómputo.
Es un backtest *de la gestión* (no solo de las estrategias): lo que se mide es la política de Argos.

**Ciclo del replay (un día simulado = una iteración, todo en Python salvo la decisión):**
1. *Corte*: `T` = día simulado. El harness solo expone datos con fecha ≤ `T` (barras, ledger simulado, WFO).
2. *Reporte automático* (Python, determinista): el mismo brief del §7 (estado de cuentas, drift, candidatas, fondeo,
   memoria de decisiones/lecciones hasta `T`).
3. *Decisión*: Claude Code lee el reporte y devuelve **un JSON de decisiones** (esquema fijo, §8). El harness lo valida
   contra los límites duros (§3); lo que viole se rechaza y se loguea.
4. *Ejecución simulada* de `T` a `T+1` con los trades históricos de las estrategias (con sus pesos/pausas vigentes),
   aplicando guardián, escalador, DD 10 % trailing, 4 % diario, lote mínimo y costos.
5. *Log y memoria*: se anota la decisión; al vencer su horizonte se etiqueta acierto/fallo y se escribe la lección.
6. Cada mes simulado (o cuando Argos lo pida) se **re-corre WFO con datos hasta `T`** y el universo de candidatas se
   actualiza. Es la política "reentrenar cada W" de `politicas_reentreno.py` (P1r, la honesta), ahora decidida por Argos.

**Arranque en cero (corte nov-2025):**
- Corridas WFO nuevas con ventanas que terminan en nov-2025; de ahí sale la cartera inicial y las estrategias validadas
  (criterio N1; N2/contexto/régimen registradas como información, igual que en §8).
- Universo = **solo estrategias y símbolos que existían y validaban con datos ≤ nov-2025**. Nada descubierto después
  (evita sesgo de supervivencia y de futuro).
- Cuentas simuladas: las 4 (25k/100k/100k/50k) con reglas de la firma (`accounts.json`, `accounts_conditions.json`).

**Objetivo y cómo se mide.** "Pasar las cuentas de fondeo lo más rápido posible sin quemarlas":
- Un solo año es **una trayectoria**: con suerte o mala suerte engaña. Por eso se lanzan **arranques escalonados**
  (una cuenta nueva cada 1–2 semanas durante el año) y se reporta la **distribución**: % que pasa, días a pasar
  (mediana y p90), % que se quema, % que queda trabada, costo por cuenta fondeada (resets/compras).
- Contra qué se compara (mismos datos, mismas reglas): (1) congelada desde nov-2025 (P0), (2) la auto-pausa actual (P2),
  (3) ERC estático con reentreno mensual (P1r), (4) tu criterio de gestión si lo podés reconstruir. Argos es una política
  más (P9) en el mismo arnés del sandbox que ya existe (`sandbox/politicas`). Si no le gana a P0/P2, no justifica su complejidad.
- Salidas: curva de equity y de DD por cuenta (formato de la solapa en vivo), `decisiones.jsonl`, `lecciones.md`,
  tablas de aciertos/fallos por tipo de decisión y por capa (N1/N2/contexto/régimen).

**Realismo de ejecución (si no, el resultado no vale):**
- Spread, slippage y swaps por símbolo (`symbol_specs.json`); el spread histórico real no está: usar valores
  conservadores y repetir con un escenario peor (×1,5–2).
- Lote mínimo y redondeo hacia abajo (ya pasa en cta1 con metales: no abre), margen, horario de mercado, fines de semana.
- Latencia de decisión: Argos decide con el cierre de `T` y ejecuta en `T+1` (nunca en la misma vela).
- Los trades salen del motor real de las estrategias con los parámetros que el WFO habría dado a esa fecha, no de los
  parámetros finales (esa es la fuga más común).

**Controles de fuga de información (críticos):**
- *Datos*: filtro estricto por fecha en cada lectura; test automático que falla si algún archivo tiene fecha > `T`.
- *Modelo*: Claude conoce eventos de mercado hasta mediados de 2026, es decir, **parte del período replayado**. Mitigación:
  el reporte **anonimiza** fechas y símbolos (estrategia_17, par_A, día +143) y no incluye noticias; se prohíbe
  explícitamente razonar con conocimiento externo y se audita en la justificación. Aun así queda un riesgo residual que se
  declara en el informe; lo definitivo es el test en vivo (§8).
- *Parámetros del arnés*: se congelan (límites, horizontes, prompts, criterio de éxito) **antes** de correr; si se
  cambian, se vuelve a correr desde cero y se anota. Prohibido iterar el prompt mirando el resultado del año completo:
  dejar un período de *calibración* (p. ej. dic–feb) para afinar y **un período ciego** (mar→hoy) para evaluar.

**Costo y ejecución.**
- ~250 días hábiles × 1 llamada a Claude Code ≈ barato; los reportes y la simulación son Python puro. El cuello de botella
  es **el WFO repetido** (`wfo_runner.py`, necesita los datos de mercado locales): correr en la laptop/PC, no en la nube.
- Para ahorrar y acotar el ruido, Claude decide **solo los días con disparador** (alerta, drift rojo, vencimiento de
  reentreno, cuenta cerca de meta/límite, resultado de un WFO nuevo) y cada lunes como revisión semanal; el resto
  "mantener". Se deja registrado cuántas decisiones fueron "nada".
- Reproducibilidad: se guarda el reporte y la respuesta cruda de cada día; con eso se puede reauditar sin rellamar al modelo.

**Etapas.** (a) Armar el arnés y validarlo con una política tonta (P0) hasta que reproduzca resultados conocidos de
`sandbox/politicas`; (b) WFO a nov-2025 y cartera inicial; (c) calibración 3 meses; (d) período ciego; (e) informe con
intervalos y veredicto; (f) recién ahí, el test en vivo en sombra (§8).

## 8c. Objetivo de Argos y reglas de fondeo (definido por el usuario)

**Función objetivo, en este orden:** (1) fondear con el **menor costo esperado** (precio + resets), (2) con el **máximo retorno**
posible hasta pasar, (3) una vez fondeada, **modo estable**: bajar riesgo, proteger el colchón y cobrar. Alternativa a
evaluar: **cuenta Instant** (sin challenge), que evita pagar el challenge pero tiene reglas más duras.

**El árbol de decisión ya existe** (solapa "Calculadora de fondeo": `calculadora_fondeo.py` + `panel_fondeo.py`).
Argos lo usa como herramienta, no lo reimplementa: `arbol()` elige por cuenta entre seguir / resetear / comprar nueva y el
`account_risk_mult`, con `evaluar()` (challenge) y `evaluar_instant()` (Instant). Costo esperado por cuenta fondeada =
`costo / (n_cuentas * p_pasa)`, con precios y descuentos de `accounts_conditions.json` (`_precios_fundingtraders`: p. ej. 1-Step
Pro 25k $229, 50k $329, 100k $589; reset fase 1 con 60 % off, una sola vez; promo TRYFT 50 % en el primer challenge).

**Reglas por tipo de cuenta que Argos debe respetar** (de `_reglas_firma_fundingtraders`, leídas el 2026-10-16):

| | Pro 1-Step (cta2/3/4) | Swing 2-Step (cta1) | Instant |
|---|---|---|---|
| Objetivo | 10 % | 8 % + 5 % | cobro desde +3,5 % |
| Pérdida máx. | 10 % trailing desde mayor balance, sin bloqueo | 10 % estática | **6 %** trailing hasta +3 %, después bloqueada en el inicial |
| Pérdida diaria | **3 %** (equity inicio del día) | 5 % (balance) | 3 % |
| Riesgo por idea | ≤ 2 % del inicial | — | ≤ 1 % por trade |
| Otras | consistencia 50 % (mejor día ≤ 50 % de la ganancia), mín. 2 días | mín. 4 días | **pérdida flotante total ≤ 1 %**, consistencia 15 % |
| Pago | 21 días (14 con complemento), vuelve al balance inicial | 14 días | a pedido, queda en inicial + 3 % |

**Modo estable tras fondear** (nuevo en el árbol): al pasar, Argos cambia de objetivo a "cobrar sin romper": baja el
`account_risk_mult`, respeta la consistencia (no concentrar ganancia en un día) y planifica el ciclo de pago; recordar que
tras cada pago el piso de pérdida máxima se reinicia en el balance inicial (Pro). Se mide en el replay como una segunda fase.

**Puntos a verificar antes de cablear (posibles discrepancias):**
- Instant: el usuario recordaba **5 %** de pérdida máxima; la ayuda leída dice **6 %**, y `evaluar_instant` usa 6. Confirmar
  en la firma y dejarlo como parámetro, no hardcodeado.
- **Pérdida diaria**: la documentación del servidor (`VIGILANCIA_LAPTOP_agente.md`) habla de DD diario 4 % y guardián en 3,5 %,
  mientras `accounts_conditions.json` da **3 %** para Pro 1-Step (cta2/3/4). Si el 3 % es el real, el guardián deja pasar días que rompen la
  cuenta. Es lo primero a revisar porque afecta cuentas en vivo.
- Instant: con 80 estrategias abiertas a la vez, la regla de **flotante ≤ 1 %** y **riesgo ≤ 1 % por trade** puede ser incompatible
  con la cartera actual (que suma 6,4 % si abre todo). Argos debe construir una cartera específica para Instant.
- Los 3 mult actuales están en ×1,25 y el riesgo por idea del 2 % también limita el escalador.

## 8d. Argos arma la cartera: evaluar los métodos de construcción, diversificación y rebalanceo

Hoy hay varias formas de armar y comparar carteras en el Thinker; en el replay se **evalúan como un experimento** y Argos
elige entre ellas (no inventa pesos: elige *método + parámetros* de un menú acotado y Python lo ejecuta):

| Pieza existente | Qué aporta | Variantes a comparar |
|---|---|---|
| `portfolio_candidato.py` | selección por categoría, símbolos no correlacionados, Deflated Sharpe (corrige por nº de pruebas), `seleccionar_candidatos(_multi)`, `buscar_curva_mas_aplanada`, `buscar_optuna`, score `dd_penalty`/K-ratio | tope por categoría y total; criterio de score |
| `diversificacion.py` | cuántas estrategias por cuenta, 4 estilos de sizing (RP 1/σ, ERC, Kelly μ/σ², % fijo), al azar vs "mejores" (Sharpe, PF) vs oráculo, todo llevado al mismo DD p95 | N de estrategias, estilo de sizing, DD objetivo |
| `pesos_cartera.py` / `aplicar_pesos_sandbox.py` | pesos aplicados reales (ERC del sandbox) | ERC estático vs. recalculado |
| `tabs_portfolio.py` | armado y curva de equity como en Backtest | — |
| Rebalanceo | frecuencia y disparador | nunca / mensual / ante drift / por vida útil |

**Diseño del experimento (en el replay, con arranques escalonados):** matriz *constructor × sizing × N × rebalanceo* →
métricas por celda (p_pasa, días, p_quiebra, costo por fondeada, retorno, DD p95, Sharpe, rotación). Se incluye el
**oráculo** solo como techo de referencia. Argos entra en dos niveles: (1) elige la celda inicial con datos ≤ corte y
(2) decide si rebalancear/rotar cada día de disparador. Se mide cuánto agrega cada decisión sobre la mejor celda **fija**: si
no mejora a "elegir la mejor celda una vez y dejarla", la complejidad extra no se justifica. Mantener el recorte de edge (57 %)
y el sesgo de selección ya documentados en `diversificacion.py`/`politicas_reentreno.py`.

**Datos de mercado.** Se cargan desde MT5 en la laptop/servidor (puente) o con el script de extracción del Thinker
(`barras.py`, `fetch_symbol_specs.py`). Para el replay: exportar una vez las barras por símbolo/timeframe hasta hoy a CSV y
que el arnés las lea con filtro por fecha (punto-en-el-tiempo); `symbol_specs.json` para lotes, spreads y contract size.

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
- Reglas (§8c): confirmar con la firma la pérdida máxima del Instant (5 % vs 6 %) y la pérdida diaria real de Pro 1-Step (3 % vs 4 %
  que usa el servidor). ¿Querés cuenta Instant como objetivo, además de challenge? ¿De qué tamaño?
- Replay (§8b): ¿hay datos de mercado locales hasta hoy y cuánto tarda una corrida de `wfo_runner.py` por ventana?
  ¿Qué estrategias/símbolos existían y validaban a nov-2025? ¿Qué reglas exactas tiene cada firma/cuenta (objetivo,
  DD, costo de reset/compra)? ¿Cuál es el criterio de éxito prioritario: días a pasar o % que pasa sin quemar?
- ¿Nube con puente por Drive, o tarea en la laptop con `claude -p`? (El sandbox necesita los ledger y series: en la
  laptop es directo.)
- Qué cuentas son reales y cuáles de challenge hoy (ver `accounts_conditions.json`) y los costos de reset/compra.
- (★) Historial de cuotas, resets y pagos cobrados hasta hoy, para arrancar el libro del negocio.
- ~~Gastos mensuales y presupuesto~~ → respondido: 5.000/mes (mínimo 2.500); 50–100/mes de cuotas + 20 de Claude (ver P0, P7).
- (★) Estado actual de cta2 (balance, mayor balance, piso) y regla de pérdida diaria confirmada en Metrix.
