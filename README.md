# MCTA024 – Somador de Ponto Flutuante em FPGA (DE10-Lite)

Projeto Prático da disciplina **Sistemas Digitais (MCTA024)** — UFABC.

---

## Etapa 1

### 1. Objetivo do Projeto

Este projeto tem como objetivo adaptar o **somador de ponto flutuante simplificado de 13 bits** apresentado no livro *FPGA Prototyping by VHDL Examples* (Pong P. Chu), originalmente concebido para uma arquitetura de placa mais antiga, para a placa **Terasic DE10-Lite** (FPGA Intel MAX 10, dispositivo 10M50DAF484C7G).

O circuito recebe dois operandos em formato de ponto flutuante simplificado — 1 bit de sinal, 4 bits de expoente e 8 bits de significando — e realiza a soma seguindo o processo clássico de aritmética em notação científica: **ordenação** (identificação do maior e menor operando), **alinhamento** dos expoentes, **adição/subtração** dos significandos e **normalização** do resultado final.

O trabalho foi dividido em três frentes técnicas: (1) validação por simulação do algoritmo original em GHDL/GTKWave, sem nenhuma alteração de hardware; (2) adaptação da entidade de topo para rotear as entradas e saídas do circuito aos elementos físicos da DE10-Lite (chaves e displays de 7 segmentos), novamente validada por simulação; e (3) síntese física no Quartus Prime e teste do circuito na placa real. O uso de inteligência artificial (Claude) foi documentado ao longo de todo o processo, conforme detalhado na seção de Diário de Bordo de IA.

### 2. Descrição gráfica do funcionamento do sistema

O diagrama abaixo mostra o fluxo de dados pelos 4 estágios do `fp_adder`, usando os nomes reais dos sinais declarados no VHDL:

![Diagrama dos 4 estágios do fp_adder](docs/img/diagrama_estagios.png)

*(imagem exportada do widget de diagrama gerado durante o desenvolvimento — ver conversa anexada)*

**Tabela 1 — Codificador de zeros à esquerda (`leado`)**

Circuito do tipo *priority encoder*: percorre os bits de `sum` do mais significativo pro menos significativo e para no primeiro `1` encontrado.

| Condição (primeiro bit em 1) | `leado` | Zeros à esquerda |
|---|---|---|
| `sum(7) = '1'` | `000` | 0 |
| `sum(6) = '1'` | `001` | 1 |
| `sum(5) = '1'` | `010` | 2 |
| `sum(4) = '1'` | `011` | 3 |
| `sum(3) = '1'` | `100` | 4 |
| `sum(2) = '1'` | `101` | 5 |
| `sum(1) = '1'` | `110` | 6 |
| nenhum dos acima | `111` | 7 (ou todos zero) |

**Tabela 2 — Decisão de normalização (4º estágio)**

| Condição | `expn` | `fracn` | Significado |
|---|---|---|---|
| `sum(8) = '1'` | `expb + 1` | `sum(8 downto 1)` | Carry-out na adição — resultado excedeu 1 bit, desloca à direita |
| `leado > expb` | `"0000"` | `"00000000"` | Underflow — resultado pequeno demais pra representar normalizado, vira zero |
| caso contrário | `expb - leado` | `sum_norm` | Normalização padrão — desloca à esquerda pelo nº de zeros contados |

---

## Etapa 2

### 3. Adaptações de Hardware (DE10-Lite)

A arquitetura original do livro (`fp_adder_test`, Listing 3.20) foi projetada para uma placa Xilinx mais antiga, com um display de 7 segmentos multiplexado no tempo (compartilhando 4 dígitos através de um módulo `disp_mux`) e um esquema de entrada baseado em 8 switches + 4 botões físicos. Essa arquitetura não é compatível diretamente com a DE10-Lite, que tem 10 switches, apenas 2 botões e 6 displays de 7 segmentos **independentes** (sem necessidade de multiplexação).

**O que mudamos no VHDL original:**

* **Removemos** a dependência das entidades `hex_to_sseg` e `disp_mux` do livro (não fornecidas no material e projetadas para multiplexação, desnecessária na DE10-Lite). Criamos uma função `hex_to_sseg` própria, combinacional, com a tabela de segmentos ativa em nível baixo específica da nossa placa.
* **Roteamos** o operando 1 como uma constante fixa no código (`sign1`, `exp1`, `frac1`), já que a DE10-Lite não tem switches suficientes para os 26 bits dos dois operandos simultaneamente.
* **Roteamos** o operando 2 inteiramente pelos 10 switches físicos (`SW`), distribuindo 1 bit de sinal, 4 bits de expoente e 5 bits do significando (os 2 bits menos significativos do significando ficam fixos em `"00"` por limitação de switches disponíveis).
* **Reorganizamos** a saída: em vez de multiplexar 4 dígitos num barramento de display único (como no livro), conectamos cada estágio do resultado a um display independente (`HEX0`–`HEX3`), simplificando a lógica de exibição.
* **Mantivemos** o núcleo do algoritmo (`fp_adder.vhd`) sem nenhuma alteração — a entidade original validada na Etapa 1 é instanciada dentro do wrapper (`fp_adder_top.vhd`) via `entity work.fp_adder`, garantindo que a lógica matemática já comprovada continue intacta.

**Descrição gráfica do sistema**

O diagrama de blocos da seção 2 permanece válido em nível lógico/algorítmico — as adaptações desta etapa são de **roteamento de pinos e interface física**, não alteram o fluxo de dados entre os 4 estágios internos do `fp_adder`.

### 4. Evidências de Validação

#### Simulação

Abaixo, as imagens do funcionamento do 4º estágio (normalização), considerando os 4 casos detalhados no `fp_adder_tb.vhd`:

| Caso | Descrição | Print GTKWave |
|---|---|---|
| 1 | Soma com alinhamento de expoente | ![Caso 1](docs/img/gtkwave_caso1.png) |
| 2 | Carry-out na adição | ![Caso 2](docs/img/gtkwave_caso2.png) |
| 3 | Subtração com shift de normalização | ![Caso 3](docs/img/gtkwave_caso3.png) |
| 4 | Underflow para zero | `[TODO: print do caso 4, 30-40ns]` |

Validação adicional do wrapper `fp_adder_top` (Etapa 2), confirmando que o roteamento pelos switches e a decodificação para os displays não alteraram a lógica matemática:

```
Caso A (op2 desprezivel)      -> PASS
Caso B (carry-out)            -> PASS
Caso C (resultado negativo)   -> PASS
```

*(8/8 verificações PASS — ver `fp_adder_top_tb.vhd`)*

#### Código VHDL Final

O núcleo do algoritmo (`fp_adder.vhd`) permanece **idêntico ao original do livro**, validado na Etapa 1, e é apenas instanciado dentro do wrapper abaixo — nenhuma linha da lógica de sort/align/add-sub/normalize foi alterada.

O arquivo `fp_adder_top.vhd` é o que contém as adaptações de hardware. Os trechos mais relevantes estão marcados com `==> ADAPTACAO N` nos comentários, correspondendo aos itens listados na seção 3:

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity fp_adder_top is
    port(
        SW   : in  std_logic_vector(9 downto 0);
        HEX0 : out std_logic_vector(7 downto 0);
        HEX1 : out std_logic_vector(7 downto 0);
        HEX2 : out std_logic_vector(7 downto 0);
        HEX3 : out std_logic_vector(7 downto 0)
    );
end entity fp_adder_top;

architecture arch of fp_adder_top is

    signal sign1, sign2, sign_out : std_logic;
    signal exp1, exp2, exp_out     : std_logic_vector(3 downto 0);
    signal frac1, frac2, frac_out  : std_logic_vector(7 downto 0);

    -- ==> ADAPTACAO 1: decodificador de 7 segmentos proprio para a DE10-Lite
    -- (substitui o "hex_to_sseg" externo do livro, que dependia de
    --  multiplexacao via "disp_mux" — desnecessaria aqui, pois a DE10-Lite
    --  tem 6 displays HEX independentes)
    function hex_to_sseg(hex : std_logic_vector(3 downto 0)) return std_logic_vector is
        variable seg : std_logic_vector(7 downto 0);
    begin
        case hex is
            when "0000" => seg := "11000000"; -- 0
            when "0001" => seg := "11111001"; -- 1
            when "0010" => seg := "10100100"; -- 2
            when "0011" => seg := "10110000"; -- 3
            when "0100" => seg := "10011001"; -- 4
            when "0101" => seg := "10010010"; -- 5
            when "0110" => seg := "10000010"; -- 6
            when "0111" => seg := "11111000"; -- 7
            when "1000" => seg := "10000000"; -- 8
            when "1001" => seg := "10010000"; -- 9
            when "1010" => seg := "10001000"; -- A
            when "1011" => seg := "10000011"; -- b
            when "1100" => seg := "11000110"; -- C
            when "1101" => seg := "10100001"; -- d
            when "1110" => seg := "10000110"; -- E
            when others => seg := "10001110"; -- F
        end case;
        return seg;
    end function;

begin

    -- ==> ADAPTACAO 2: operando 1 fixado como constante em codigo
    -- (a DE10-Lite nao tem switches suficientes para os 26 bits dos
    --  dois operandos ao mesmo tempo — decisao de projeto do grupo)
    sign1 <= '0';
    exp1  <= "1000";
    frac1 <= "10101000";  -- valor fixo = +168.0 (0.65625 x 2^8)

    -- ==> ADAPTACAO 3: operando 2 totalmente roteado pelos 10 switches
    -- SW(9)          -> sinal
    -- SW(8 downto 5) -> expoente (4 bits)
    -- SW(4 downto 0) -> 5 bits mais significativos do significando
    --                   (2 bits menos significativos fixos em "00")
    sign2 <= SW(9);
    exp2  <= SW(8 downto 5);
    frac2 <= '1' & SW(4 downto 0) & "00";

    -- Nucleo do algoritmo: entidade original da Etapa 1, sem alteracoes
    fp_add_unit : entity work.fp_adder
        port map(
            sign1 => sign1, sign2 => sign2,
            exp1  => exp1,  exp2  => exp2,
            frac1 => frac1, frac2 => frac2,
            sign_out => sign_out, exp_out => exp_out, frac_out => frac_out
        );

    -- ==> ADAPTACAO 4: saida em displays independentes, sem multiplexacao
    -- (substitui o esquema multiplexado do livro por 4 displays diretos
    --  da DE10-Lite: HEX0-HEX3)
    HEX0 <= hex_to_sseg(frac_out(3 downto 0));
    HEX1 <= hex_to_sseg(frac_out(7 downto 4));
    HEX2 <= hex_to_sseg(exp_out);
    HEX3 <= "10111111" when sign_out = '1' else "11111111"; -- '-' ou apagado

end arch;
```

---

## Etapa 3

### Funcionamento na Placa

`[TODO: imagens/vídeos do circuito rodando na DE10-Lite física, cobrindo os mesmos 4 casos de teste da simulação — a ser preenchido após a síntese no Quartus e a gravação via JTAG]`

---

## Etapa 4

### 5. Diário de Bordo de IA

Utilizamos o **Claude (Anthropic)** para auxiliar na geração dos testbenches, na criação do wrapper de adaptação para a placa DE10-Lite e na estruturação da documentação. Abaixo está a análise crítica do uso da ferramenta.

**Prompts Utilizados (exemplos representativos):**

> "Otimo, extraia o codigo do pdf e monte o arquivo fp_adder_tb.vhd"

> "Nesse caso entao eu preciso da placa para continuar os testes?"

> "Como vamos proceder [com a adaptação para a DE10-Lite, já que só temos 10 switches e 2 botões]?"

*(conversa completa disponível em PDF anexado a este repositório)*

**O Erro da IA (Alucinação):**

Ao gerar o primeiro testbench para o `fp_adder_top.vhd` (`fp_adder_top_tb.vhd`), a IA calculou os padrões de segmento esperados para os displays de 7 segmentos (`HEX0`, `HEX1`, `HEX2`) **trocados** em relação à própria tabela de conversão (`hex_to_sseg`) que ela mesma havia escrito no código. Por exemplo, o valor esperado para o dígito "9" foi escrito incorretamente como o padrão do dígito "3" (`10000011` ao invés de `10010000`), e o valor esperado para o expoente "8" foi escrito como o padrão do dígito "0" (`11000000` ao invés de `10000000`). O erro só foi detectado porque a IA **executou a simulação antes de entregar o arquivo** (rodando o GHDL num ambiente sandbox), e o resultado de `FAIL` no terminal expôs a inconsistência entre o valor esperado (escrito errado) e o valor real produzido pelo circuito (correto).

**A Correção Humana:**

A correção foi feita revisando, linha por linha, a tabela `hex_to_sseg` contra cada valor esperado no testbench, e recompilando com `ghdl -a` / `ghdl -r` até os 8 casos de verificação retornarem `PASS`. Esse episódio reforça um ponto importante do uso responsável de IA no projeto: **a IA pode errar até em tarefas mecânicas de "conferência de tabela"**, e a validação por simulação (rodar o GHDL de fato, e não confiar cegamente no código gerado) foi o que garantiu a correção antes da entrega — reforçando por que a Etapa 1 e 2 do projeto exigem simulação validada antes de ir para a placa física.

### 6. Contribuição dos participantes

Taxonomia CRediT (https://credit.niso.org/):

* `[Nome do Aluno 1]` — Administração do Projeto, Desenvolvimento, implementação e teste de software, Análise Formal
* `[Nome do Aluno 2]` — Validação de dados e experimentos
* `[Nome do Aluno 3]` — Redação do manuscrito original, Validação de dados e experimentos

---

## Bibliografia

Chu, P. P. *FPGA Prototyping by VHDL Examples*. Hoboken, NJ, John Wiley & Sons, Inc., 2008.

Intel/Altera. *Laboratory Exercise 1*. In https://ftp.intel.com/Public/Pub/fpgaup/pub/Teaching_Materials/current/Laboratory_Exercises/Digital_Logic/VHDL/lab1.pdf
