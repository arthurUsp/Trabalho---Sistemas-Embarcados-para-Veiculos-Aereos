# Trabalho---Sistemas-Embarcados-para-Veiculos-Aereos
Este repositório contém os códigos base para o trabalho da disciplina de Sistemas Embarcados para veículos aéreos.

Para cloná-lo e fazer seu trabalho, crie um fork dele e faça as alterações que julgar necessárias para demonstrar seu entendimento do assunto.

Serão avaliados as suas mudanças e seu comentários sobre elas.

O seu trabalho deve ser compilado e relatado no README.md de seu repositório.

Qualquer dúvida estamos à disposição! 
e-mail: arthurdsd@usp.br

Bom trabalho!


---

## Melhoria 2 — Monitoramento da bateria com histerese e aprimoramento da filtragem digital

**Responsável:** Esther Vieira Parise Bianchini - 15636362

**Branch:** `feat/battery-monitoring`

**Arquivo modificado:** `YMFC-AL_Flight_controller.ino`

### 1. Problema identificado

O monitoramento da tensão da bateria é importante
para a segurança operacional de veículos aéreos
não tripulados, pois permite identificar condições
de alimentação inadequadas durante a operação.

No código original, foram identificadas as seguintes
limitações:

- Ausência de histerese no indicador de bateria;
- Ausência de uma condição explícita para desativar
  o alerta após a recuperação da tensão;
- Compartilhamento do LED entre os indicadores de
  bateria baixa e falhas de temporização;
- Utilização de limites de tensão definidos
  diretamente nas condições do código;
- Perda de precisão numérica durante a filtragem
  da tensão, devido ao armazenamento do resultado
  em uma variável do tipo `int`.

Essas características podem comprometer a precisão
do monitoramento e dificultar a interpretação dos
alertas apresentados pelo sistema.

### 2. Solução proposta

A implementação foi dividida em duas etapas:

1. Monitoramento da tensão utilizando histerese;
2. Aprimoramento da precisão numérica do filtro
   digital utilizado no monitoramento da bateria.

#### 2.1. Monitoramento da bateria com histerese

Na primeira etapa, foram definidos dois limites
distintos para a ativação e desativação do alerta
de bateria baixa.

Considerando inicialmente uma bateria LiPo 3S,
foram adotados os seguintes parâmetros:

| Parâmetro | Valor |
|---|---:|
| Limite de ativação do alerta | 10,5 V |
| Limite de desativação do alerta | 10,8 V |
| Faixa de histerese | 0,3 V |

Os limites foram armazenados nas constantes
`BATTERY_LOW_THRESHOLD` e
`BATTERY_RECOVERY_THRESHOLD`.

Como o código representa a tensão em centésimos
de volt, os valores utilizados são 1050 e 1080,
respectivamente.

A histerese permite que pequenas oscilações de
tensão próximas ao limite não provoquem mudanças
sucessivas no estado do indicador.

O estado do alerta é atualizado pela função
`update_battery_warning()`.

O comportamento implementado é apresentado
na tabela a seguir:

| Condição | Estado do alerta |
|---|---|
| Tensão menor ou igual a 10,5 V | Ativado |
| Tensão entre 10,5 V e 10,8 V | Estado anterior mantido |
| Tensão maior ou igual a 10,8 V | Desativado |

Os limites adotados são parâmetros iniciais de
projeto e deverão ser confirmados experimentalmente
com a bateria utilizada.

#### 2.2. Separação dos indicadores de falha

O código original utiliza o LED conectado ao pino
digital 12 para indicar tanto condições de bateria
baixa quanto falhas de temporização.

Para evitar que a recuperação da tensão apague um
alerta de temporização, foram introduzidas duas
variáveis booleanas independentes:

- `low_battery`: representa o estado do alerta
  de bateria baixa;
- `timing_fault`: representa a ocorrência de
  uma falha de temporização.

O estado do LED é determinado pela operação
lógica OU entre as duas variáveis:

```cpp
digitalWrite(12, low_battery || timing_fault);
```

| Bateria baixa | Falha temporal | LED |
|---|---|---|
| Não | Não | Desligado |
| Sim | Não | Ligado |
| Não | Sim | Ligado |
| Sim | Sim | Ligado |

A falha de temporização permanece registrada até
a reinicialização do sistema, preservando o
comportamento original desse indicador.

#### 2.3. Aprimoramento do filtro digital

O código original utiliza um filtro passa-baixas
digital de primeira ordem para reduzir a influência
de ruídos sobre a medição da tensão da bateria.

A equação do filtro pode ser expressa por:

$$
V_f[k] = (1-\alpha)V_f[k-1] + \alpha V_m[k]
$$

Em que:

- $V_f[k]$ representa a tensão filtrada atual;
- $V_f[k-1]$ representa a tensão filtrada anterior;
- $V_m[k]$ representa a nova tensão medida;
- $\alpha$ representa o peso da nova medição.

O coeficiente utilizado no código original é
$\alpha = 0,08$.

Entretanto, o armazenamento da estimativa em uma
variável do tipo `int` provoca o descarte da parte
fracionária do resultado a cada iteração.

Por exemplo, considerando uma estimativa inicial
de 1200 e uma nova medição igual a 1210, temos:

$$
V_f[k] = 0,92(1200) + 0,08(1210) = 1200,8
$$

Ao armazenar esse resultado em uma variável inteira,
o valor obtido será 1200.

Caso a entrada permaneça constante, a mesma situação
poderá ocorrer nas iterações seguintes, impedindo
que o filtro acompanhe pequenas variações de tensão.

Para reduzir essa perda de precisão, foi criada
uma variável independente do tipo `float`:

```cpp
float battery_voltage_filtered = 0.0f;
```

A equação do filtro foi reorganizada como:

$$
V_f[k] = V_f[k-1] +
\alpha(V_m[k]-V_f[k-1])
$$

A implementação correspondente é:

```cpp
battery_voltage_filtered +=
    BATTERY_FILTER_ALPHA *
    (measured_voltage - battery_voltage_filtered);
```

A utilização de uma variável de ponto flutuante
permite preservar as pequenas correções realizadas
entre as iterações.

Essa alteração melhora a precisão numérica da
estimativa, sem aumentar a resolução física
do conversor analógico-digital.

#### 2.4. Inicialização e modularização da filtragem

A variável `battery_voltage_filtered` é inicializada
com o valor da primeira medição analógica durante
a execução da função `setup()`.

Essa estratégia evita um transiente artificial
associado à inicialização do filtro com tensão
igual a zero.

Também foi criada a função `update_battery_voltage()`,
responsável pela aquisição e filtragem da tensão.

Os seguintes parâmetros foram definidos como
constantes:

| Constante | Valor | Finalidade |
|---|---:|---|
| `BATTERY_ADC_GAIN` | 1.2317 | Fator de conversão |
| `BATTERY_ADC_OFFSET` | 65 | Compensação original |
| `BATTERY_FILTER_ALPHA` | 0.08 | Coeficiente do filtro |

Os fatores de conversão e o coeficiente do filtro
foram preservados em relação ao projeto original.

A implementação utiliza uma única leitura do ADC
por iteração, compartilhada pelos dois filtros.

O filtro original foi mantido para a compensação
da tensão dos motores, enquanto o novo filtro é
utilizado exclusivamente no monitoramento da
bateria.

Dessa forma, a alteração não modifica diretamente
os ganhos PID nem a equação original de compensação
dos motores.

#### 2.5. Análise da resposta temporal do filtro

O controlador de voo possui um período nominal
de amostragem de 4 ms, correspondente a uma
frequência de 250 Hz.

A constante de tempo equivalente do filtro pode
ser calculada por:

$$
\tau = \frac{-T_s}{\ln(1-\alpha)}
$$

Substituindo os parâmetros adotados:

$$
\tau = \frac{-0,004}{\ln(0,92)}
$$

$$
\tau \approx 0,048 \text{ s}
$$

Portanto, a constante de tempo equivalente é de
aproximadamente 48 ms.

O tempo teórico necessário para atingir 95% de
uma variação em degrau é de aproximadamente 144 ms,
considerando a resposta exponencial equivalente.

O coeficiente do filtro original foi preservado
porque ainda não existem medições experimentais
que permitam justificar uma nova frequência de
filtragem.

### 3. Comparação entre as implementações

| Característica | Original | Commit 1 | Commit 2 |
|---|---|---|---|
| Monitoramento com histerese | Não | Sim | Sim |
| Limites configuráveis | Não | Sim | Sim |
| Estados independentes de alerta | Não | Sim | Sim |
| Tipo do filtro de monitoramento | int | int | float |
| Preservação da parte fracionária | Não | Não | Sim |
| Função específica de filtragem | Não | Não | Sim |
| Leituras ADC por iteração | 1 | 1 | 1 |
| Coeficiente do filtro | 0,08 | 0,08 | 0,08 |
| Compensação original dos motores | Preservada | Preservada | Preservada |

### 4. Benefícios esperados

As alterações propostas apresentam os seguintes
benefícios:

- Maior confiabilidade do indicador de bateria;
- Redução de mudanças sucessivas no estado do alerta;
- Preservação dos avisos de falhas de temporização;
- Redução dos erros associados ao truncamento
  durante a filtragem digital;
- Melhor organização e modularização do código;
- Facilidade de configuração dos parâmetros;
- Preservação da lógica original de compensação
  dos motores.

### 5. Compilação e validação

#### 5.1. Ambiente de compilação

O código original e as duas versões modificadas
foram compilados utilizando o Arduino IDE, com
a placa Arduino Uno selecionada, baseada no
microcontrolador ATmega328P.

As três compilações foram realizadas com sucesso.

#### 5.2. Resultados das compilações

| Parâmetro | Original | Commit 1 | Commit 2 |
|---|---:|---:|---:|
| Memória Flash utilizada | 12374 bytes | 12418 bytes | 12548 bytes |
| Ocupação da memória Flash | 38% | 38% | 38% |
| Memória SRAM global | 495 bytes | 497 bytes | 501 bytes |
| Ocupação da memória SRAM | 24% | 24% | 24% |
| Memória SRAM restante | 1553 bytes | 1551 bytes | 1547 bytes |
| Resultado da compilação | Sucesso | Sucesso | Sucesso |

A capacidade disponível para armazenamento do
programa é de 32256 bytes de memória Flash,
enquanto a capacidade total de SRAM é de 2048 bytes.

#### 5.3. Análise do consumo adicional de memória

| Implementação | Flash adicional | SRAM global adicional |
|---|---:|---:|
| Monitoramento com histerese | 44 bytes | 2 bytes |
| Aprimoramento do filtro | 130 bytes | 4 bytes |
| **Total** | **174 bytes** | **6 bytes** |

O aumento total corresponde a aproximadamente
1,41% da memória Flash utilizada originalmente
e 1,21% da memória SRAM global utilizada
originalmente.

O aumento de SRAM é consistente com as novas
variáveis globais declaradas:

| Variável | Tipo | Memória ocupada |
|---|---|---:|
| `low_battery` | bool | 1 byte |
| `timing_fault` | bool | 1 byte |
| `battery_voltage_filtered` | float | 4 bytes |
| **Total** | | **6 bytes** |

Apesar das funcionalidades acrescentadas, o código
permanece dentro dos limites de memória disponíveis
no microcontrolador.

#### 5.4. Testes funcionais propostos

Para avaliar o comportamento das implementações,
propõem-se os seguintes testes de bancada:

| Teste | Resultado esperado |
|---|---|
| Inicialização com tensão de 12,6 V | Alerta desativado |
| Redução da tensão filtrada para 10,5 V | Alerta ativado |
| Recuperação da tensão filtrada para 10,6 V após o alerta | Alerta permanece ativado |
| Recuperação da tensão filtrada para 10,8 V ou mais | Alerta desativado, na ausência de falha temporal |
| Simulação de falha de temporização | LED ativado |
| Recuperação da bateria após falha temporal | LED permanece ativado |
| Aplicação de pequena variação positiva na tensão medida | Novo filtro acompanha a variação |
| Verificação do período de execução | Manutenção do período nominal de 4 ms |

Os resultados apresentados nessa tabela representam
o comportamento esperado, e não medições
experimentais já realizadas.

### 6. Limitações

A compilação bem-sucedida confirma que o código
é aceito pelo compilador e permanece dentro dos
limites de memória informados.

Entretanto, a compilação não garante o cumprimento
dos requisitos temporais nem o funcionamento
adequado em condições reais.

Os limites de tensão e os fatores de conversão
do ADC deverão ser verificados de acordo com
a bateria e o circuito de condicionamento utilizados.

O monitoramento da tensão total da bateria não
permite identificar individualmente a tensão
de cada célula e não substitui um sistema de
proteção contra descarga excessiva.

A melhoria da precisão numérica foi aplicada
exclusivamente ao filtro de monitoramento.
O filtro original utilizado na compensação
dos motores foi preservado.

O impacto das operações adicionais de ponto
flutuante sobre o período de execução do
controlador deverá ser avaliado experimentalmente.

Os testes físicos permanecem pendentes e deverão
ser realizados em bancada, sem hélices e com
os motores desconectados nos testes de tensão.

### 7. Histórico das alterações

- `c7b0677`: implementação do monitoramento de
  bateria com histerese;
- `ef596c9`: aprimoramento da precisão do filtro
  digital da bateria.
