# Arquiteto Quant-Microestrutura — Blueprint em Pine Script v6

## 1. Relação Macro vs. Micro (Valor vs. Preço)
O Motor Macro define o **Preço Justo** via paridade de juros coberta ajustada por basis, enquanto o Sensor Micro (VTC) detecta a eficiência de consumo de liquidez para determinar **quando** o preço tenderá a convergir ao valor. A lógica causal é: o diferencial de juros (DI1 vs. cupom) ancora o valor teórico; desvios do preço de tela refletem fricções e fluxos. O VTC monitora se a agressão disponível é suficiente para deslocar o preço em direção ao justo ou se há absorção/exaustão que interrompe o movimento. O preço observado é, portanto, um epifenômeno da cadeia "Valor → Fluxo → Execução" descrita no PDF (seções 1–3).

## 2. Estruturas de Dados (Pine v6)
- **Curva DI1**: `map<string, float>` para cache de vértices (ticker → taxa), combinado com `array<string>` para tickers ativos e `matrix<float>` para calendários de dias úteis por vértice (colunas: vencimento, DU acumulado). Interpolação usa arrays de DU e taxas.
- **Histórico de Volatilidade (GK)**: `array<float>` para buffer rolling de `log(High/Low)^2`, com janela N configurável; `array<float>` de bandas calculadas (±1σ, ±2σ) para cones.
- **Matriz de Correlações (HRP)**: `matrix<float>` para correlação entre vetores de retornos dos sinais (Macro, VTC, CDS/DXY). `matrix<float>` adicional para distâncias hierárquicas e quasi-diagonalização; `array<float>` para pesos finais.

## 3. Notas de Pine Script v6
- v6 permite **`request.security()` com ticker dinâmico** (strings construídas em runtime) e suporta `map`, `matrix`, e `array` com métodos de manipulação (ex.: `matrix.mult`, `matrix.transpose`).
- Tipagem explícita reduz erros de casting; manter inicializações fora de `if` para evitar alocação repetida.
- Limites de tempo de execução: calcular HRP em janelas curtas e atualizar apenas em `barstate.islast` para evitar timeouts. Evitar `matrix.inv` completa; usar pseudo-inversão simplificada (inverso de variância por cluster).

## 4. Módulos e Pseudocódigo

### 4.1 Módulo Macro — `Futuro_Justo`
- **Interpolação DI1 (Flat Forward)**
```pinescript
// entrada: target_du (dias úteis até vencimento WDO)
// 1) montar tickers dinâmicos para DI anterior/posterior
string di_prev = f_di_ticker(target_du, -1)
string di_next = f_di_ticker(target_du, +1)
float r_prev = request.security(di_prev, tf, close)
float r_next = request.security(di_next, tf, close)
int du_prev = f_du_to_maturity(di_prev)
int du_next = f_du_to_maturity(di_next)
float w = (target_du - du_prev) / math.max(1.0, (du_next - du_prev))
float r_interp = r_prev + w * (r_next - r_prev)
```
- **Cupom Cambial (FRC) ou Implícito**
```pinescript
float cupom = request.security(frc_ticker, tf, close, gaps=barmerge.gaps_off)
if na(cupom)
    float fut_px = close        // WDO corrente
    float spot = request.security(spot_ticker, tf, close)
    cupom := math.pow((fut_px / spot) / (1 + r_interp), 252.0/target_du) - 1
```
- **Paridade Ajustada por Basis (CIP)** — seção 2.1–2.4
```pinescript
float justo_teorico = spot * math.pow((1 + r_interp) / (1 + cupom), target_du/252.0)
float basis = ta.ema(justo_teorico - close, 10)
float justo_ajustado = justo_teorico + basis
plot(justo_ajustado, color=color.gold)
```

### 4.2 Módulo Micro — VTC (Absorção/Exaustão)
- **Eficiência de Deslocamento** (seção 3.1): `E_d = ΔPreço / Volume`.
```pinescript
float delta_px = close - close[1]
float v = volume
float ed = v != 0 ? delta_px / v : 0
float ed_smooth = ta.ema(ed, 20)
// Absorção: ed baixo e volume alto com range estreito
bool absorcao = (math.abs(ed_smooth) < ed_thresh_low) and (v > vol_thresh_high) and (high - low < range_thresh)
// Exaustão: ed alto com volume decrescente
bool exaustao = (math.abs(ed_smooth) > ed_thresh_high) and (v < vol_thresh_low)
```
- **HVN/LVN Magnets** (seção 3.2)
```pinescript
var map<float, float> profile = map.new<float, float>()
float bucket = math.floor(close / tick_size) * tick_size
map.set(profile, bucket, map.get(profile, bucket, 0.0) + volume)
// identificar HVN/LVN periodicamente
if barstate.islast
    array<float> keys = map.keys(profile)
    array.sort(keys, order.ascending)
    // localizar máximos/mínimos de volume
```

### 4.3 Módulo Risco — Volatilidade Garman-Kohlhagen (seção 4.1)
```pinescript
int N = input.int(20, "Janela GK")
var array<float> gk_terms = array.new_float()
float term = math.pow(math.log(high/low), 2)
array.push(gk_terms, term)
if array.size(gk_terms) > N
    array.shift(gk_terms)
float sum_terms = 0.0
for i = 0 to array.size(gk_terms)-1
    sum_terms += array.get(gk_terms, i)
float sigma_gk = math.sqrt(sum_terms / (4 * array.size(gk_terms) * math.log(2)))
float band1 = close * sigma_gk
plot(close + 1*sigma_gk, color=color.new(color.blue, 50))
plot(close - 1*sigma_gk, color=color.new(color.blue, 50))
```

### 4.4 Módulo HRP (síntese intraday)
```pinescript
// Vetores de retornos padronizados dos sinais
matrix<float> corr = f_corr_matrix(returns_matrix)
matrix<float> dist = matrix.apply(corr, (c) => math.sqrt(0.5*(1-c)))
array<int> order = f_quasi_diag(dist)
array<float> weights = f_recursive_bisect(order, corr)
// peso do trade = weight_macro*macro_signal + weight_vtc*micro_signal + weight_cds*cds_signal
```

## 5. Plano de Execução (Ordem de Desenvolvimento)
1. **Fundação Macro**: Implementar utilitários de calendário B3, geração dinâmica de tickers DI/FRC, interpolação DI1 e cálculo do `Futuro_Justo` com basis CIP.
2. **Sensor Micro**: Construir VTC (E_d, absorção/exaustão), profile HVN/LVN via maps/arrays e lógica de ímãs.
3. **Volatilidade**: Codificar estimador GK, cones de volatilidade e integração com sinais VTC.
4. **Risco/Alocação**: Implementar correlações, HRP simplificado e fator Kelly fracionado.
5. **Dashboard**: Plots do justo ajustado, bandas GK, marcadores VTC e tabela resumo.
