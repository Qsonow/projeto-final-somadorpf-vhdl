# Etapa 1 — Validação do VHDL Original (Somador de Ponto Flutuante Simplificado)

## Objetivo

Comprovar que o algoritmo matemático do somador de ponto flutuante de 13 bits
(livro *FPGA Prototyping by VHDL Examples*, Pong P. Chu, seção 3.7.4) funciona
corretamente **antes** de qualquer modificação para a placa DE10-Lite.

## O que foi feito

1. O código VHDL original (`fp_adder.vhd`) foi transcrito **exatamente** como
   apresentado no material da disciplina, sem nenhuma alteração de lógica ou
   de portas.
2. Foi criado um testbench (`fp_adder_tb.vhd`) com 4 casos de teste, escolhidos
   para cobrir os 4 comportamentos críticos do 4º estágio (normalização):

   | # | Caso | O que valida |
   |---|------|---------------|
   | 0 | Soma simples, sem shift, sem carry-out | Caminho básico de soma |
   | 1 | Soma que gera **carry-out** | Ramo `sum(8) = '1'` → `expn <= expb + 1` |
   | 2 | Subtração com cancelamento total (**underflow para zero**) | Ramo `leado > expb` → resultado forçado a zero |
   | 3 | Subtração que exige **shift de normalização** (1 bit) | Contador de zeros à esquerda (`leado`) e deslocamento (`sum_norm`) |

3. Compilação e simulação com GHDL (padrão VHDL-2008):
   ```bash
   ghdl -a --std=08 fp_adder.vhd
   ghdl -a --std=08 fp_adder_tb.vhd
   ghdl -e --std=08 fp_adder_tb
   ghdl -r --std=08 fp_adder_tb --wave=fp_adder.ghw
   ```
4. Visualização da forma de onda no GTKWave (`gtkwave fp_adder.ghw`), observando
   os sinais internos `sum`, `leado`, `sum_norm`, `expn` e `fracn` durante os
   quatro casos de teste.

## Resultado

Todos os 4 testes bateram com o valor esperado (calculado manualmente antes da
simulação), sem nenhum `assertion error`:

```
Teste 0
  esperado: sign='0' exp=1000 frac=11000000
  obtido  : sign='0' exp=1000 frac=11000000
Teste 1
  esperado: sign='0' exp=1001 frac=11111111
  obtido  : sign='0' exp=1001 frac=11111111
Teste 2
  esperado: sign='1' exp=0000 frac=00000000
  obtido  : sign='1' exp=0000 frac=00000000
Teste 3
  esperado: sign='0' exp=0111 frac=10100000
  obtido  : sign='0' exp=0111 frac=10100000
```

## Observação relevante (comportamento de borda)

No **Teste 2**, os dois operandos têm exatamente a mesma magnitude
(`exp & frac` idênticos). O 1º estágio (sorting) usa uma comparação estrita
(`>`), então em caso de empate o **operando 2** é escolhido como "big number":

```vhdl
if (exp1 & frac1) > (exp2 & frac2) then
    -- operando 1 vira "big"
else
    -- operando 2 vira "big"  <- entra aqui em empate
end if;
```

Isso faz com que o sinal do resultado zero siga o sinal do operando 2
(`sign_out = sign2`), gerando um "zero negativo" quando `sign2 = '1'`.
Não é um bug — é o comportamento definido pelo próprio algoritmo original —,
mas é um ponto interessante para comentar no relatório, pois mostra que a
representação escolhida não trata `+0` e `-0` como equivalentes na lógica
de seleção do estágio 1.

## Checklist do que ainda falta ADICIONAR a este resumo

- [ ] **Screenshot do GTKWave** mostrando pelo menos o Teste 3 (o caso de shift),
      com os sinais `sum`, `leado`, `sum_norm` visíveis — a imagem que você já
      me mandou é um bom começo, mas vale recortar mostrando mais sinais internos.
- [ ] **Print do terminal** com o output completo do GHDL (você já tem, é só
      colar em um bloco de código no README).
- [ ] Uma frase confirmando **quem do grupo** rodou a simulação (para depois
      usar na taxonomia CRediT — ex: "Validation", "Software").
- [ ] Se quiser, uma **tabela de conversão decimal → binário** dos 4 casos de
      teste (eu já fiz as contas pra você, mas fica mais didático no relatório
      mostrar o valor decimal ao lado do binário).
- [ ] Uma linha citando que este testbench e a transcrição do código foram
      feitos com apoio de IA (Claude), conforme exigido na Etapa 4 —
      guarde o link/PDF desta conversa para anexar depois.

## Arquivos desta etapa
- `fp_adder.vhd` — código original (sem alterações)
- `fp_adder_tb.vhd` — testbench com 4 casos de validação
- `fp_adder.ghw` — waveform gerada pela simulação (gerar localmente)
