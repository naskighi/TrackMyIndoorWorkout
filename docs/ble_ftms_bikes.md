# Comunicação BLE FTMS para Bikes no projeto TrackMyIndoorWorkout

> Escopo: este documento cobre **somente equipamentos de bike** e como este projeto interpreta/usa FTMS (Fitness Machine Service) para leitura de dados e controle.

## 1) Visão geral da pilha no projeto

A comunicação BLE para bikes é estruturada em camadas:

1. **Mapeamento GATT FTMS**: UUIDs de serviço/características, opcodes e bits de feature.
2. **Descritor de dispositivo**: define como interpretar flags/payload (layout dinâmico por flags no FTMS).
3. **Equipamento (runtime)**: descobre serviços, lê features, envia controle (control point), deduplica pacotes e consolida métricas.
4. **Ajustes por fabricante/modelo**: subclasses com workarounds para desvios de implementação FTMS.

## 2) GATT FTMS usado para bikes

## 2.1 Serviço e características principais

- Serviço FTMS: `0x1826`.
- Characteristic de dados de bike indoor: `0x2AD2` (Indoor Bike Data).
- Characteristic de status da máquina: `0x2ADA`.
- Characteristic de features da máquina: `0x2ACC`.
- Characteristic de controle (Fitness Machine Control Point): `0x2AD9`.

Também há characteristics auxiliares de suporte a ranges de escrita:

- Supported Speed Range `0x2AD4`
- Supported Inclination Range `0x2AD5`
- Supported Resistance Level Range `0x2AD6`
- Supported Heart Rate Range `0x2AD7`
- Supported Power Range `0x2AD8`

## 2.2 Operações de controle implementadas

No projeto, os opcodes FTMS relevantes definidos incluem:

- `requestControl (0x00)`
- `resetControl (0x01)`
- `setResistanceLevelControl (0x04)`
- `startOrResumeControl (0x07)`
- `stopOrPauseControl (0x08)` com info de stop (`0x01`) e pause (`0x02`)
- `spinDownControl (0x13)`

Códigos de resposta FTMS considerados:

- success (`0x01`), opcodeNotSupported (`0x02`), invalidParameter (`0x03`), operationFailed (`0x04`), controlNotPermitted (`0x05`).

## 2.3 Features (leitura/escrita)

Ao conectar, o runtime tenta ler `Fitness Machine Feature (0x2ACC)` e interpreta:

- 32 bits de **read features**
- 32 bits de **write features**

Com isso, o app identifica suporte a comandos e ranges (velocidade, inclinação, resistência, FC alvo, potência alvo) e também se suporta spin-down.

## 3) Fluxo de comunicação BLE (bike FTMS)

## 3.1 Descoberta

1. Conecta ao periférico e descobre serviços/características.
2. Seleciona serviço FTMS e characteristic de dados `0x2AD2` para bikes.
3. (Opcional) Lê `0x2ACC` para capabilities (pode ser bloqueado por preferência).
4. (Opcional) Lê Manufacturer Name (Device Information) para validação do equipamento (alguns descritores pulam isso).

## 3.2 Controle (Control Point)

Quando aplicável, ao conectar ao control point, o app envia `requestControl` (`[0x00]`) antes de operações ativas.

A execução de qualquer comando de controle depende de:

- Bluetooth ligado
- characteristic de controle presente
- preferência de bloquear start/stop não ativa

Se falhar write no control point, a exceção é apenas logada.

## 3.3 Coleta de dados

Para cada notificação de `Indoor Bike Data (0x2AD2)`:

1. Lê flags (normalmente 2 bytes, little endian).
2. O descritor processa bits das flags e monta o layout dos campos no payload.
3. `stubRecord()` extrai métricas (speed, cadence, distance, power, calories, elapsed, HR, resistance etc.).
4. Runtime aplica deduplicação/throttling e merge entre tipos de pacote quando necessário.

## 4) Parsing FTMS Indoor Bike Data no projeto

## 4.1 Regras base de flags/campos

Classe base para bike (`IndoorBikeDeviceDescriptor`) usa o parser genérico FTMS com esta ordem:

1. Instant Speed (**bit invertido** no FTMS bike; presente quando bit0=0)
2. Average Speed
3. Instant Cadence
4. Average Cadence
5. Total Distance
6. Resistance Level
7. Instantaneous Power
8. Average Power
9. Expended Energy (total + por hora + por minuto)
10. Heart Rate
11. MET
12. Elapsed Time
13. Remaining Time

Resoluções/conversões utilizadas:

- Speed: `uint16 / 100` (km/h)
- Cadence: `uint16 / 2` (rpm)
- Distance: `uint24` (metros)
- Resistance: `sint16`
- Power: `sint16` (W)
- Energy total/hora: `uint16`, por minuto: `uint8`
- Heart Rate: `uint8`
- Elapsed Time: `uint16` (segundos)

## 4.2 Sentinela “not available”

Descritores métricos tratam valores FTMS especiais como nulos:

- `uint8`: `0xFF`
- `uint16`: `0xFFFF`
- `uint24`: `0xFFFFFF`
- `uint32`: `0xFFFFFFFF`
- `uint48`: `0xFFFFFFFFFFFF`

Isso é crucial para não considerar lixo como dado válido.

## 4.3 Robustez de tamanho do pacote

A validação de pacote é relaxada: em vez de exigir tamanho exato, o parser aceita se o payload tiver **ao menos** os bytes necessários calculados por flags (`dataLength >= byteCounter`).

## 5) Diferenças por equipamento (bikes)

## 5.1 Generic FTMS Bike (baseline)

- Usa parser FTMS padrão de bike indoor.
- Espera conformidade com flags + payload.
- Serve de fallback para equipamentos desconhecidos com FTMS bike.

## 5.2 Schwinn IC4/IC8 e Bowflex C7 (Nautilus)

No projeto ambos usam descritor indoor bike padrão, porém testes mostram padrão típico:

- Ex.: flags `0x09FE` indicam speed/cadence/distance/power/energy/elapsed/resistance.
- `canMeasureCalories` é desativado em alguns modelos Nautilus no factory (evita confiar em calorias do equipamento como fonte principal em alguns fluxos).

## 5.3 Stages SB20

- Também usa descritor padrão, mas emite **pacotes FTMS fragmentados por tipo de métrica** em alta frequência (ex.: pacote só de speed, só de distance, só de cadence+power).
- O runtime junta/mescla múltiplos pacotes recentes para compor o registro final.

Sem esse merge, outro projeto perderá métricas “ausentes” em notificações individuais.

## 5.4 Yesoul S3

- Usa descritor padrão, mas na prática alterna entre dois layouts frequentes:
  - pacote com speed + elapsed
  - pacote com cadence + distance + power + calories
- O merge temporal no runtime é essencial para reconstruir estado completo.

## 5.5 Matrix Bike (não conforme FTMS)

Tem override explícito de `processFlag` para tratar violação FTMS:

- Flags vistas: `0x15FE` e `0x1DFE` com payload de 20 bytes.
- Flags prometem muitos campos (incluindo energy/time), mas payload real não comporta tudo.
- Implementação ignora campos inconsistentes e parseia o subconjunto confiável: speed, cadence, distance, resistance, power.

Para reproduzir em outro projeto: implemente regra especial por `(flag,dataLength)` antes do parser genérico.

## 5.6 Life Fitness Bike (não conforme FTMS)

Também possui override de flag parsing:

- Combinações tratadas: `0x09FA`, `0x0BFA`, `0x19FA`, `0x1BFA` (com comprimentos 26/27/28/29).
- Flags declaradas não incluem instant cadence em alguns casos, mas payload real inclui.
- Parser usa `processCadenceFlag(..., inverse: true)` para acomodar discrepância.
- Pode incluir/omitir HR e remaining time conforme modo (incluindo cooldown).

Sem esse workaround, cadence/offsets ficam errados.

## 5.7 Cardiostrong IB50

- Herda parser de bike padrão, mas sobrescreve parsing de distância.
- Distância é reportada em **decâmetros (10 m)**, não em metros FTMS padrão.
- Implementação ajusta divider para multiplicar o valor bruto por 10 (`divider: 0.1`).

Sem essa correção, distância sai 10x menor que a real.

## 5.8 Concept2 BikeErg

- No projeto, BikeErg usa caminho `Concept2Erg` específico (serviço/characteristics proprietárias Concept2), não FTMS indoor bike padrão.
- Portanto, para “bikes” no sentido amplo, ele é exceção importante fora do pipeline FTMS `0x2AD2`.

## 6) Estratégia para reconstruir em outro projeto

## 6.1 Mínimo necessário

1. Implementar camada GATT FTMS com:
   - `0x1826`, `0x2AD2`, `0x2ACC`, `0x2AD9`, `0x2ADA`
2. Parser de flags FTMS Indoor Bike com bit0 invertido para speed.
3. Conversões de unidade e tratamento de `not available`.
4. Leitura de `Fitness Machine Feature` + ranges de suporte.
5. Control point com `requestControl` e opcodes de start/stop/resistance.

## 6.2 Compatibilidade real (produção)

Além do “mínimo”, replicar:

- Merge de pacotes heterogêneos por janela temporal (casos Stages/Yesoul).
- Regras especiais por fabricante:
  - Matrix (flags inválidas + payload curto)
  - Life Fitness (flags inconsistentes, cadence invertida)
  - Cardiostrong IB50 (distância em decâmetros)
- Validação flexível de comprimento (`>=` mínimo esperado).

## 6.3 Algoritmo recomendado de parsing

Para cada notificação:

1. Ler `flag` little-endian (2 bytes).
2. Resetar índices/descritores métricos para o pacote.
3. Se `(fabricante/modelo, flag, tamanho)` bater em regra especial, usar parser custom.
4. Caso contrário usar parser FTMS padrão.
5. Extrair métricas com conversão + sentinelas.
6. Inserir no buffer de merge por chave de flag/tipo de pacote.
7. A cada janela de throttle, mesclar melhor valor disponível de cada métrica.

## 7) Tabela-resumo das diferenças por bike

| Equipamento | Base | Diferença principal | Impacto para integração |
|---|---|---|---|
| Generic FTMS Bike | FTMS padrão `0x2AD2` | Nenhuma | Implementar parser padrão completo |
| Schwinn IC4/IC8 | FTMS padrão | Padrões de flag específicos; calorias nem sempre confiáveis | Não depender só de calorias do device |
| Bowflex C7 | FTMS padrão | Similar Schwinn/Nautilus | Mesmo cuidado com calorias |
| Stages SB20 | FTMS padrão | Pacotes fragmentados por métrica | Precisa merge temporal |
| Yesoul S3 | FTMS padrão | Alternância entre layouts parciais | Precisa merge temporal |
| Matrix Bike | FTMS com workaround | Flags “prometem” mais que payload real | Parser especial por flag+tamanho |
| Life Fitness Bike | FTMS com workaround | Flags inconsistentes com payload real | Parser especial + cadence invertida |
| Cardiostrong IB50 | FTMS com workaround | Distância em 10m | Ajuste de escala de distância |
| Concept2 BikeErg | Não FTMS bike padrão | Protocolo Concept2 próprio | Implementar stack separado |

## 8) Observações finais

- O comportamento real dos equipamentos diverge da spec FTMS com frequência.
- Este projeto já codifica vários “patches de mundo real”; para reproduzir fidelidade, copie os workarounds por modelo, não apenas a spec oficial.
- Para bikes FTMS, o ponto crítico é **parser correto + merge de notificações + correções por fabricante**.
