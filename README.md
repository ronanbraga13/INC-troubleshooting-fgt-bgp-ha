# INC - Troubleshooting FortiGate BGP + HA

Lab de troubleshooting de um **incidente simulado** em FortiGate com HA (High Availability), BGP (Border Gateway Protocol), IPsec e ECMP (Equal-Cost Multi-Path).

A proposta deste repositório é operacional: reproduzir o INC, coletar evidências, localizar a quebra e validar a correção. Pouca teoria e foco no que seria feito durante um incidente.

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
- o BGP precisa reconvergir;
- as rotas chegavam a desaparecer durante a transição;
- a comunicação apresentava uma segunda janela de perda após o chaveamento.

Em um dos primeiros testes foram observadas **72 perdas consecutivas de ICMP** (`icmp_seq=17` a `icmp_seq=88`). A imagem abaixo é uma **transcrição visual dos logs fornecidos durante o teste**, e não um print original da CLI.

![Transcrição das 72 perdas consecutivas de ICMP](evidencias/00-transcricao-72-perdas-icmp.svg)

## Validação durante o INC

```bash
get system ha status
get router info bgp summary
get router info bgp neighbors <IP_DO_NEIGHBOR> routes
get router info routing-table bgp
```

### Neighbors do lab

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

Para acompanhar a sessão TCP do BGP durante o failover:

```bash
diagnose sniffer packet any 'port 179' 4 0 l
```

## O que foi encontrado

### 1. HA chaveava, mas o BGP ainda estava reconvergindo

Após o failover, o novo Primary assumia corretamente, porém os neighbors chegaram a permanecer em `Active` e a tabela BGP ficou temporariamente sem rotas.

![BGP em Active e ausência de rotas após o failover](evidencias/05-bgp-active-sem-rotas-pos-failover.png)

O `route-ttl` padrão observado era:

```text
route-ttl = 10
```

Aumentar somente o `route-ttl` mantinha as rotas sincronizadas por mais tempo, mas não corrigia sozinho a reconvergência do BGP.

### 2. Timers BGP

Os timers inicialmente observados eram:

```text
keepalive = 60s
holdtime  = 180s
```

No lab foram ajustados para:

```bash
config router bgp
    set keepalive-timer 10
    set holdtime-timer 30
end
```

A mudança reduziu o tempo de recuperação das sessões após o failover.

### 3. A sessão voltava antes das rotas

Mesmo com o BGP em `Established`, foi observado `State/PfxRcd = 0` por alguns segundos.

Na Matriz, durante a segunda janela de perda, o comando abaixo não retornava as redes `10.10.x.x` do Rio:

```bash
get router info routing-table bgp | grep 10.10
```

Quando as rotas abaixo reapareciam, o ping voltava imediatamente:

```text
10.10.10.0/24
10.10.20.0/24
10.10.30.0/24
```

Isso confirmou que a segunda indisponibilidade estava relacionada à **ausência temporária da rota de retorno na Matriz**.

![Ausência temporária das rotas de retorno na Matriz](evidencias/10-rota-retorno-ausente-matriz.png)

## Advertisement interval

Foi confirmado nos peers:

```text
Minimum time between advertisement runs is 30 seconds
```

O `advertisement-interval` foi reduzido para 1 segundo nos dois lados.

### Matriz

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

### Rio

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

Validação:

```bash
get router info bgp neighbors <IP_DO_NEIGHBOR> | grep -i "Minimum time"
```

Esperado:

```text
Minimum time between advertisement runs is 1 seconds
```

## Configuração final do teste

```text
route-ttl              = 60s
keepalive-timer        = 10s
holdtime-timer         = 30s
advertisement-interval = 1s
```

HA:

```bash
config system ha
    set route-ttl 60
end
```

Também foi testado `route-ttl 80`, porém o resultado permaneceu em aproximadamente 3 perdas. Como não houve ganho prático, foi mantido o valor de 60 segundos.

## Resultado

Após os ajustes, os testes repetidos de failover passaram a apresentar aproximadamente **3 perdas de ICMP**, sem a longa indisponibilidade observada no início.

![Resultado após os ajustes, com poucos pacotes perdidos](evidencias/12-resultado-final-3-perdas.png)

```text
ANTES
HA failover
→ reconvergência lenta
→ rotas temporariamente ausentes
→ longa janela de perda

DEPOIS
HA failover
→ timers BGP ajustados
→ anúncios de rota acelerados nos dois sentidos
→ route-ttl cobrindo a janela de reconvergência
→ aproximadamente 3 perdas transitórias
→ comunicação estabilizada
```

## Conclusão

O problema não estava no chaveamento do HA em si. O impacto prolongado era causado pelo tempo necessário para reconstruir as sessões BGP e, principalmente, para que as rotas fossem novamente anunciadas e instaladas nos dois sentidos.

Os ajustes reduziram a indisponibilidade de dezenas de segundos para uma perda transitória de poucos pacotes.

> Os valores deste lab foram definidos a partir do comportamento observado neste cenário simulado. Em produção, timers BGP e parâmetros de HA devem ser validados de acordo com a topologia, peers, SLAs e comportamento do ambiente.

## Evidências e documentação

- [Ver as evidências do incidente, em ordem cronológica](EVIDENCIAS.md).
- [Documentação técnica oficial da Fortinet consultada](REFERENCIAS.md).
