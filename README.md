# INC - Troubleshooting FortiGate BGP + HA

Lab de troubleshooting de um incidente simulado em FortiGate com HA (High Availability), BGP (Border Gateway Protocol), IPsec e ECMP (Equal-Cost Multi-Path).

O objetivo aqui não é explicar toda a teoria do BGP. A proposta é reproduzir o INC, validar o comportamento durante o failover, identificar onde a comunicação quebra e aplicar ajustes até reduzir a indisponibilidade.

## Sintoma

Antes do failover:

- HA saudável e sincronizado;
- dois neighbors BGP estabelecidos;
- 3 prefixos recebidos por neighbor;
- rotas BGP instaladas;
- comunicação ICMP estável entre Rio e Matriz.

Após a perda do Primary do Rio:

- o Secondary assume corretamente;
- os túneis IPsec permanecem ativos;
- as sessões BGP precisam reconvergir;
- durante os primeiros testes, as rotas desapareciam e a comunicação apresentava uma segunda janela de perda significativa após o chaveamento.

## Validação rápida durante o INC

```bash
get system ha status
get router info bgp summary
get router info bgp neighbors <IP_DO_NEIGHBOR> routes
get router info routing-table bgp
```

### Neighbors usados no lab

**Rio - AS 65000**

```text
1.1.1.2
2.2.2.2
```

**Matriz - AS 65001**

```text
1.1.1.1
2.2.2.1
```

## Evidência inicial

No baseline, os dois caminhos estavam operacionais e o ICMP apresentava request/reply normalmente.

![Baseline antes do failover](evidencias/01-baseline-antes-failover.png)

## Reprodução do incidente

Após o failover do HA, o novo Primary assumiu, porém os neighbors BGP ficaram inicialmente em `Active` e a tabela BGP chegou a ficar sem rotas.

![BGP Active após failover](evidencias/02-bgp-active-sem-rotas.png)

Em um dos primeiros testes foram observadas dezenas de perdas consecutivas antes da recuperação do tráfego.

## Investigação

### 1. Route TTL

O valor padrão observado no lab era:

```text
route-ttl = 10
```

O `route-ttl` mantém temporariamente no novo Primary as rotas sincronizadas pelo HA enquanto os protocolos dinâmicos reconvergem.

Aumentar apenas esse valor melhorava temporariamente o forwarding, mas não resolvia a causa da reconvergência lenta.

### 2. Timers BGP

Os timers inicialmente observados eram:

```text
keepalive = 60s
holdtime  = 180s
```

Foram ajustados no lab para:

```bash
config router bgp
    set keepalive-timer 10
    set holdtime-timer 30
end
```

O objetivo foi reduzir o tempo necessário para recuperação das sessões após o failover.

### 3. Advertisement interval

Mesmo após o BGP voltar para `Established`, foi observado um período em que `State/PfxRcd` ainda permanecia em `0`, sem os prefixos necessários na tabela.

Na Matriz foi confirmado:

```text
Minimum time between advertisement runs is 30 seconds
```

Durante a segunda janela de perda, a Matriz ainda não possuía as rotas de retorno `10.10.x.x`. Assim que essas rotas apareceram novamente na tabela BGP, o ping voltou.

![Ausência temporária das rotas de retorno](evidencias/03-rotas-retorno-ausentes.png)

O `advertisement-interval` foi reduzido para 1 segundo nos neighbors dos dois lados.

**Matriz:**

```bash
config router bgp
    config neighbor
        edit "1.1.1.1"
            set advertisement-interval 1
        next
        edit "2.2.2.1"
            set advertisement-interval 1
        next
    end
end
```

**Rio:**

```bash
config router bgp
    config neighbor
        edit "1.1.1.2"
            set advertisement-interval 1
        next
        edit "2.2.2.2"
            set advertisement-interval 1
        next
    end
end
```

## Configuração final utilizada no teste

```text
route-ttl              = 60s
keepalive-timer        = 10s
holdtime-timer         = 30s
advertisement-interval = 1s
```

Para o HA:

```bash
config system ha
    set route-ttl 60
end
```

## Resultado

Após os ajustes, o failover passou a apresentar apenas uma pequena perda transitória de aproximadamente 3 pacotes ICMP, sem a indisponibilidade prolongada observada no início do INC.

![Resultado após ajustes](evidencias/04-resultado-final.png)

Também foi testado `route-ttl 80`, porém o resultado permaneceu em aproximadamente 3 perdas. Por esse motivo, foi mantido `route-ttl 60`, pois não houve ganho prático com o valor maior.

## Resumo do INC

```text
ANTES
HA failover
→ BGP demora a convergir
→ rotas desaparecem / retorno fica ausente
→ longa janela de perda

DEPOIS
HA failover
→ timers BGP ajustados
→ anúncios de rota mais rápidos nos dois lados
→ route-ttl cobre a janela de reconvergência
→ aproximadamente 3 perdas transitórias
→ comunicação estabiliza
```

> Os valores utilizados neste lab foram definidos a partir do comportamento observado neste cenário simulado. Em produção, timers BGP e parâmetros de HA devem ser validados de acordo com a topologia, peers, SLAs e comportamento do ambiente.
