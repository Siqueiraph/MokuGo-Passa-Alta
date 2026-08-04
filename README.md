# Moku:Go - Aprendendo a operar a partir de um Filtro Passa Alta
**Documentação de Solução de Problemas: Infraestrutura, API e Processamento de Sinais**

Este documento - gerado por IA - mapeia os gargalos físicos e lógicos enfrentados na integração entre Python, SciPy e o hardware Moku:Go (Liquid Instruments), desde a camada de rede até a otimização matemática do Diagrama de Bode.

---

## 1. Infraestrutura e Conectividade (Camada Física e Rede)

### 1.1. Falha de DHCP e Endereço APIPA (O Erro de "Falha Geral")
*   **O Problema:** Ao conectar o PC ao Moku:Go (via Wi-Fi ou cabo), o Windows falha no handshake DHCP e entra em pânico, atribuindo à placa de rede um IP aleatório de Autoconfiguração (APIPA), geralmente `169.254.x.x`. Como o Moku:Go está travado em `192.168.73.1`, o Python não encontra rota de rede e lança erros de Timeout ou "Falha Geral".
*   **A Solução:** Assumir o controle manual da placa de rede.
    1. Executar `ncpa.cpl` no Windows.
    2. Nas propriedades do Adaptador (IPv4), definir IP Estático:
       * **IP:** `192.168.73.2`
       * **Máscara:** `255.255.255.0`
       * **Gateway:** `192.168.73.1`
    3. Validar a rota de comunicação no CMD com `ping 192.168.73.1`.

### 1.2. O Ponto Cego do FPGA (Erro `NoInstrumentBitstream`)
*   **O Problema:** O Moku:Go é baseado em um chip FPGA reconfigurável, não em placas fixas. Para se transformar em um Analisador de Resposta em Frequência (FRA), o Python precisa enviar uma "planta baixa" elétrica chamada *Bitstream*. Por padrão, o pacote `pip` do Moku não inclui esses arquivos pesados, gerando o erro de falta de instrumento.
*   **A Solução:** Rodar a Interface de Linha de Comando (CLI) com acesso à internet uma única vez para baixar a arquitetura:
    * `mokucli instrument download <versão-do-MokuOS>` (ex: `4.2.2`).

---

## 2. Evolução da API e Sintaxe de Código

### 2.1. Argumentos Obsoletos (Erro `unexpected keyword argument 'sweep_scale'`)
*   **O Problema:** Versões antigas da API exigiam a declaração de que a varredura seria logarítmica via `sweep_scale='Log'`. A API moderna removeu o argumento, assumindo a escala logarítmica como o comportamento natural do instrumento FRA.
*   **A Solução:** Remover o argumento da chamada `FRA.set_sweep()`.

### 2.2. A Sintaxe do Frontend (Erro `unexpected keyword argument 'attenuation'`)
*   **O Problema:** Atualizações recentes da biblioteca em Python removeram o argumento genérico de atenuação da configuração das portas de entrada.
*   **A Solução:** A API agora exige a declaração explícita do limite de tensão física (*range*). A chamada do *frontend* deve substituir `attenuation=1.0` por `range='10Vpp'`.

### 2.3. O Fim do Polling e o Loop Amador (Erro `has no attribute 'get_status'`)
*   **O Problema:** A prática de usar um loop `while` com `time.sleep()` para ficar checando o status do equipamento inunda a rede e trava o processador. A API moderna removeu o método `get_status()`.
*   **A Solução:** Utilizar chamadas de bloqueio (*blocking calls*). O Python delega ao próprio método a ordem de congelar a execução silenciosamente até que a matriz inteira seja transferida para a RAM.
    * `FRA.start_sweep()`
    * `Data = FRA.get_data(wait_complete=True)`

### 2.4. Mudança na Árvore do Dicionário (Erro `KeyError: 'frequency'`)
*   **O Problema:** Atualizações de firmware frequentemente mudam a arquitetura dos dados exportados. A chave global de frequência foi removida e embutida independentemente dentro de cada canal.
*   **A Solução:** Em vez de tentar "adivinhar" variáveis ou usar `data['frequency']`, inspecione as chaves usando `.keys()` e extraia o eixo X diretamente de um dos canais:
    * `Freqs = np.array(Data['ch1']['frequency'])`

---

## 3. Os Limites da Física e Processamento Vetorial

### 3.1. Quantização Digital vs. Modo Estrito (Erro `InvalidParameterException / Coerced Value`)
*   **O Problema:** Ao definir ciclos de aquisição (`averaging_cycles`), a teoria pode exigir, por exemplo, 0.2000000000 segundos. Contudo, o relógio digital (clock) do FPGA fatiará o tempo de forma finita, arredondando o valor para 0.1999999998 segundos. No modo restrito, a API do Python percebe a diferença microscópica de tempo e aborta a execução por segurança.
*   **A Solução:** Inserir a flag de acordo de hardware `strict=False` no método `set_sweep()`. Isso autoriza o Moku a fazer os arredondamentos necessários para executar a varredura sem travar o script.

### 3.2. Referência Relativa e as Falsas Descontinuidades de Fase
*   **O Problema:** Extrair a fase usando apenas o Canal 2 plota o atraso de forma absoluta contra o oscilador do próprio Moku, estragando o comportamento teórico do filtro em altas frequências. Além disso, instrumentos de bancada limitam a medição de fase ao intervalo cíclico de -180° a +180°. Se um sinal cruzar esse limite durante a varredura, o gráfico apresentará um salto violento e falso de 360°, arruinando a curva.
*   **A Solução:** A fase deve ser calculada de forma rigorosamente relativa à entrada (`phase_ch2 - phase_ch1`). Usa-se o processamento profissional do NumPy (`np.unwrap`) para corrigir os ângulos em radianos antes de convertê-los de volta para graus:
    * `Transf_phase = np.rad2deg(np.unwrap(np.deg2rad(phase_ch2 - phase_ch1)))`

---

## 4. O Ponto Cego da Instrumentação FRA (O Falso Platô de 20 dB)

### 4.1. O Estado Fantasma do FPGA (A Escala de 10x)
*   **O Problema:** A curva de magnitude do filtro converge incorretamente para $\approx +20\text{ dB}$, mesmo com a matemática do código (`Mag_ch2 - Mag_ch1`) sendo a via perfeita para o cálculo de grandezas logarítmicas. Na escala de decibéis, $+20\text{ dB}$ equivale exatamente a um multiplicador de tensão de **10 vezes**, o que é impossível em um circuito passivo.
*   **A Causa:** O FPGA retém na memória a última calibração feita nele. Uma prévia utilização do equipamento no aplicativo Moku Desktop com a atenuação da ponta de prova (*Probe Attenuation*) travada em **10x** faz o processador multiplicar o sinal captado por 10 digitalmente, empurrando o gráfico inteiro $\approx 20\text{ dB}$ para cima no código Python.

### 4.2. As Soluções Profissionais (Expurgando o Offset)
*   **Via Hardware - Reset de Sessão:** Aniquilar estados herdados limpando o cache do equipamento com um comando logo após inicializar a variável de controle, antes de configurar os *frontends*.
    * `FRA.set_defaults()`
*   **Via Software - Calibração Padrão VNA (Math Trace):** Semelhante ao uso de um Analisador de Redes, compensa-se a mentira da escala física normalizando o traço subtraindo o teto máximo dele mesmo, cravando a banda de passagem em $0\text{ dB}$.
    * `Offset_residual = Transf_mag.max()`
    * `Transf_mag = Transf_mag - Offset_residual`

### 4.3. Acomodação Termodinâmica no Ajuste Fino (Curve Fit)
*   **O Problema:** Usar um modelo matemático teórico rígido onde o ganho da banda de passagem é forçado a $0\text{ dB}$. Circuitos reais (protoboards e cabos BNC) possuem resistência intrínseca, gerando perdas de inserção. A matemática ideal tenta "abraçar" dados com perdas, distorcendo o cálculo da Frequência de Corte ($f_c$).
*   **A Solução:** Reconhecer a perda de inserção criando um parâmetro extra de *offset* no modelo teórico submetido ao SciPy. Extrair palpites dinâmicos baseados no teto máximo real atingido pela placa e otimizar sem cegueira física:
    * `def Calc_Transmissao(f, fc, T_pass):`
    * `return 20 * np.log10(f / fc) - 10 * np.log10( 1 + (f / fc)**2 ) + T_pass`