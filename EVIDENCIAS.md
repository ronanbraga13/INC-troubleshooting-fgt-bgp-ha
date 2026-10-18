# Evidências do INC

## Registro da perda inicial — 72 pacotes ICMP

O log coletado durante os testes apresentou **72 perdas consecutivas** (`icmp_seq=17` a `icmp_seq=88`). A imagem a seguir foi montada a partir dessa transcrição e **não representa uma captura original da tela**.

![Transcrição visual das 72 perdas de ICMP](evidencias/00-transcricao-72-perdas-icmp.svg)

## 1. Baseline antes do failover

![Baseline de comunicação](evidencias/01-baseline-comunicacao-antes-failover.png)

![BGP saudável na Matriz](evidencias/02-bgp-matriz-baseline.png)

![BGP saudável no Rio](evidencias/03-bgp-rio-baseline.png)

## 2. Falha reproduzida após o failover

![IPsec permaneceu UP](evidencias/04-ipsec-up-pos-failover.png)

![Neighbors BGP em Active e ausência de rotas](evidencias/05-bgp-active-sem-rotas-pos-failover.png)

![Configuração inicial de route-ttl](evidencias/06-config-inicial-route-ttl10.png)

## 3. Reconvergência e ajustes

![Sniffer durante a reconvergência BGP](evidencias/07-sniffer-reconvergencia-bgp.png)

![Ajuste de keepalive e holdtime](evidencias/08-ajuste-keepalive-holdtime.png)

![Advertisement interval original de 30 segundos](evidencias/09-advertisement-interval-30.png)

## 4. Causa da segunda interrupção

![Ausência temporária das rotas de retorno na Matriz](evidencias/10-rota-retorno-ausente-matriz.png)

## 5. Ajuste final e resultado

![Advertisement interval ajustado no Rio](evidencias/11-advertisement-interval-1-rio.png)

![Resultado final com perda residual de aproximadamente 3 pacotes](evidencias/12-resultado-final-3-perdas.png)
