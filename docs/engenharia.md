<picture><source media="(prefers-color-scheme: dark)" srcset="../assets/brand/logo-wordmark-tight-duotone.svg"><img src="../assets/brand/logo-wordmark-tight-dark.svg" alt="Con-Hectar" height="26"/></picture>

# Diário de engenharia

[← Voltar ao README](../README.md)

Extraído do relatório de sistema da coleira **C0001**: o que foi medido no sistema rodando, as falhas encontradas — cada uma se apresentando como algo diferente do que era — e o que continua em aberto.

---

## O que foi medido

Do sistema em funcionamento, não de raciocínio sobre ele.

| Verificação | Resultado | Por que essa é a verificação certa |
|---|:-:|---|
| Link GPS | **120 / 0** sentenças com checksum válido / inválido | A contagem de caracteres sobe mesmo lendo ruído no baud errado; só checksums válidos provam o link |
| Frames de rádio | **6 / 0 perdidos** | O gateway decodificou byte a byte o que a coleira imprimiu, confirmado por contadores nas duas pontas |
| Latência de ingestão | **7 s** | Fixo gravado às 03:30:48, enviado às 03:30:55 |
| Gatilho de retenção | **6 → 3** | Verificado com um teto reduzido num dispositivo descartável; dados reais intocados, teto restaurado |
| Autenticação do stream | **401 / 200** | Recusado sem cookie de sessão, servido com ele |

```
03:30:48  C0001 seq=26 -16.71…,-49.26… @ 03:30:47  (stored)
03:30:48    ack -> $A,C0001,26*07
03:30:55  uploaded 1
```

> ⚠️ **Consumo de energia: calculado, não medido.** Valores típicos de datasheet; nenhum multímetro esteve no sistema ainda. Ver [`hardware.md`](hardware.md#energia--a-análise-que-mudou-o-roadmap).

---

## Falhas encontradas

| Sintoma | O que realmente era | Correção |
|---|---|---|
| **O mapa abria na cidade errada** | Sem cercas nem centro da fazenda configurados, o mapa caía num padrão nacional enquanto a única coleira estava a ~200 km. Ela estava sendo desenhada o tempo todo — fora da tela. | Enquadramento automático nas coleiras ativas |
| **Frames corrompidos que não eram problema de rádio** | Caracteres faltando, mas os sobreviventes *em ordem* — dois processos lendo a mesma UART e roubando bytes um do outro. | Um único dono da porta serial |
| **Um comando funcionando que parecia morto** | Python bufferiza stdout num pipe; um monitor rodando via SSH não imprimia nada por minutos. Esse silêncio deixou o processo duplicado acima vivo. | Saída sem buffer |
| **Todo ciclo transmitindo duas vezes** | A tentativa 1 sempre expirava e a 2 sempre era confirmada. Pacote perdido é aleatório; "a primeira sempre falha" é uma resposta chegando depois da janela. Um teste A/B descartou o sono do rádio como causa. | Janela de ACK dobrada |
| **Um sentinela fingindo ser dado** | HDOP 99,99 ("sem precisão") chegava ao banco como precisão de ~500 m — um círculo que o mapa desenharia feliz. | Sentinelas filtrados na origem |
| **Um feed ao vivo sem autenticação** | O stream de posições ficava fora do código de acesso que protegia o painel acima dele. | Stream exige sessão |
| **Uma função interna exposta publicamente** | A API REST publica tudo no schema público; a função de retenção era chamável de fora. | Movida para fora do schema exposto |

---

## Decisões de produto tiradas da engenharia

- **O painel deixou de mostrar bateria.** Não há divisor nem leitura de ADC na coleira, então qualquer número seria inventado. Removemos toda superfície de bateria em vez de exibir um dado que teríamos de adivinhar.
- **O ACK não espera a nuvem** (ver [`arquitetura.md`](arquitetura.md#3-a-decisão-mais-importante-o-ack-não-espera-a-nuvem)).
- **A análise de energia redirecionou o roadmap** do firmware para a PCB.

---

## Em aberto

| Item | Tipo | Situação |
|---|---|---|
| Proteção de nível lógico entre MCU e rádio | Hardware | Mitigado em firmware em parte dos pinos; divisores resistivos pendentes |
| Coleira não guarda fixos quando não alcança o gateway | Perda de dados | Buffer em memória não volátil planejado ("modo mochila") |
| Conectividade do gateway na rede local | Implantação | Falhou no último teste de campo — prioridade nº 1 |
| Alertas de cerca derivados no navegador | Arquitetura | Funciona com o painel aberto; migrar a derivação para o gateway, que já vê todo fixo |
| Leitura de bateria | Funcionalidade | Depende da nova PCB |
| Validação da integração Sentinel-2 com conta de produção | Integração | Pendente de credenciais definitivas |
