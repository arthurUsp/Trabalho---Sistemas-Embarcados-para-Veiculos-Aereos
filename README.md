# Trabalho---Sistemas-Embarcados-para-Veiculos-Aereos
Este repositório contém os códigos base para o trabalho da disciplina de Sistemas Embarcados para veículos aéreos.

Para cloná-lo e fazer seu trabalho, crie um fork dele e faça as alterações que julgar necessárias para demonstrar seu entendimento do assunto.

Serão avaliados as suas mudanças e seu comentários sobre elas.

O seu trabalho deve ser compilado e relatado no README.md de seu repositório.

Qualquer dúvida estamos à disposição! 
e-mail: arthurdsd@usp.br

Bom trabalho!

--------------------------------------

## Melhoria 1 — Validação dos canais do receptor

**Responsável:** Esther Vieira Parise Bianchini - 15636362

**Branch:** `fix/receiver-validation`

**Arquivo modificado:** `YMFC-AL_setup.ino`

### 1. Problema identificado

O código original apresenta limitações na etapa de
validação dos quatro canais PWM do receptor.

A função `wait_for_receiver()` utiliza uma máscara
de bits para identificar os canais que possuem
pulsos com duração entre 900 e 2100 microssegundos.

Entretanto, a condição de erro original verifica
apenas se a máscara possui valor igual a zero.
Dessa forma, o sistema pode considerar a validação
bem-sucedida mesmo quando apenas parte dos canais
está funcionando.

Além disso, as variáveis responsáveis por armazenar
os sinais PWM podem manter seus últimos valores
mesmo após a interrupção do recebimento dos pulsos.

### 2. Solução proposta

A implementação foi dividida em duas etapas.

#### 2.1. Validação dos quatro canais

Na primeira etapa, foram realizadas as seguintes
alterações:

- Validação obrigatória dos quatro canais do receptor;
- Reinicialização da máscara de bits a cada verificação;
- Exigência de cinco verificações consecutivas válidas;
- Utilização de operações atômicas para leitura das
  variáveis compartilhadas com a interrupção;
- Identificação individual dos canais inválidos;
- Correção da comparação temporal para reduzir
  problemas relacionados ao overflow do temporizador.

A utilização da biblioteca `util/atomic.h` garante
que os valores compartilhados com a rotina de
interrupção sejam copiados sem que uma interrupção
ocorra durante a operação de leitura.

Essa alteração é relevante porque o ATmega328P
possui arquitetura de 8 bits, enquanto os valores
dos canais são armazenados em variáveis de 16 bits.

#### 2.2. Verificação temporal dos pulsos PWM

Na segunda etapa, a rotina de interrupção
`ISR(PCINT0_vect)` foi modificada para armazenar,
além da duração do pulso, o instante de recebimento
de cada sinal.

Foram acrescentadas as seguintes variáveis:

- `last_pulse_time`: armazena o instante do último
  pulso recebido em cada canal;
- `received_channels`: identifica os canais que
  já receberam pelo menos um pulso;
- `RECEIVER_TIMEOUT_US`: estabelece o tempo máximo
  de ausência de novos pulsos.

O limite temporal adotado inicialmente foi de
100 ms (100000 microssegundos).

Considerando um receptor operando a 50 Hz, esse
intervalo corresponde a cinco períodos nominais
do sinal PWM.

Assim, cada canal deve satisfazer simultaneamente
três condições:

1. Ter recebido pelo menos um pulso;
2. Apresentar duração superior a 900 e inferior
   a 2100 microssegundos;
3. Ter recebido seu último pulso há menos de 100 ms.

Também foi acrescentada uma proteção contra a
conversão de durações excessivamente grandes
para variáveis do tipo `int`.

### 3. Comparação entre as implementações

| Característica | Original | Modificado |
|---|---|---|
| Validação obrigatória dos quatro canais | Não | Sim |
| Leitura atômica | Não | Sim |
| Verificações consecutivas | Não | Sim |
| Verificação temporal dos pulsos | Não | Sim |
| Diagnóstico individual de falhas | Não | Sim |
| Tempo máximo de configuração do receptor | 10 s | 10 s |

### 4. Benefícios esperados

As alterações propostas têm como objetivo aumentar
a confiabilidade da etapa de configuração do drone,
evitando a aceitação de canais ausentes, inválidos
ou com informações desatualizadas no momento
da validação.

Também permitem identificar possíveis problemas
de conexão entre o receptor e o microcontrolador,
facilitando o diagnóstico durante a configuração.


### 5. Compilação e validação

#### 5.1. Ambiente de compilação

As duas etapas da implementação foram compiladas
utilizando o Arduino IDE, com a placa Arduino Uno
selecionada, baseada no microcontrolador ATmega328P.

As compilações foram realizadas com sucesso, sem
erros, permitindo analisar o consumo de memória
associado às modificações implementadas.

#### 5.2. Resultados das compilações

| Parâmetro | Commit 1 | Commit 2 |
|---|---:|---:|
| Memória Flash utilizada | 19990 bytes | 20572 bytes |
| Ocupação da memória Flash | 61% | 63% |
| Memória SRAM global utilizada | 515 bytes | 532 bytes |
| Ocupação da memória SRAM | 25% | 25% |
| Memória SRAM restante | 1533 bytes | 1516 bytes |
| Resultado da compilação | Sucesso | Sucesso |

O Arduino Uno possui 32256 bytes de memória Flash
disponíveis para o programa e 2048 bytes de memória
SRAM.

#### 5.3. Análise do custo computacional

A implementação da verificação temporal dos pulsos
PWM resultou em um aumento de 582 bytes na memória
Flash e de 17 bytes na memória SRAM global, em
relação à primeira etapa da implementação.

O aumento observado na memória SRAM é consistente
com a declaração das novas variáveis globais:

| Variável | Quantidade | Memória ocupada |
|---|---:|---:|
| `last_pulse_time` | 4 unsigned long | 16 bytes |
| `received_channels` | 1 byte | 1 byte |
| **Total** | | **17 bytes** |

Apesar do aumento no consumo de memória, o programa
permaneceu dentro dos limites de armazenamento do
microcontrolador.

A segunda implementação utiliza 63% da memória
Flash disponível e 25% da memória SRAM, conforme
os percentuais apresentados pelo Arduino IDE.

Dessa forma, a melhoria acrescenta mecanismos
de validação e diagnóstico do receptor com um
baixo custo adicional de memória global.

#### 5.4. Validação da implementação

A compilação bem-sucedida confirma a compatibilidade
do código com o ambiente de desenvolvimento e com
a arquitetura do microcontrolador selecionado.

Entretanto, a compilação não comprova isoladamente
o funcionamento do sistema em condições reais.

Para a validação experimental, propõem-se os
seguintes testes de bancada:

| Teste | Resultado esperado |
|---|---|
| Quatro canais válidos e ativos | Configuração autorizada |
| Apenas três canais ativos | Configuração rejeitada |
| Ausência de pulsos em todos os canais | Configuração rejeitada |
| Pulso com duração fora dos limites | Canal identificado como inválido |
| Interrupção do sinal durante a validação | Detecção de informação desatualizada |
| Restabelecimento dos sinais dentro do timeout | Validação após cinco verificações consecutivas |

Os testes experimentais em hardware permanecem
pendentes e deverão ser realizados em condições
controladas de bancada, sem as hélices instaladas.


### 6. Limitações

A melhoria implementada atua na validação inicial
do receptor durante a etapa de configuração.

Ela não realiza o monitoramento contínuo dos canais
durante o voo, que será abordado posteriormente
na implementação do failsafe.

Além disso, receptores que continuam gerando pulsos
PWM após a perda de comunicação com o transmissor
podem não ter essa falha identificada apenas pela
verificação temporal.

O limite de 100 ms deverá ser confirmado de acordo
com as características do receptor utilizado.

A validação experimental em hardware permanece
pendente.

### 7. Histórico das alterações

- `052b656`: correção da validação dos quatro canais;
- `94adff4`: verificação da atualização dos sinais PWM.
