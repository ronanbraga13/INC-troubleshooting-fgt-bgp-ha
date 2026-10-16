# Referências oficiais Fortinet

Documentação consultada durante a análise e os testes deste INC simulado.

## HA e retenção de rotas

- [Technical Tip: Controlling how HA synchronizes routing table updates](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Controlling-how-HA-synchronizes-routing-table/ta-p/193241)
- [FortiOS CLI Reference — config system ha](https://docs.fortinet.com/document/fortigate/latest/cli-reference/857448677/config-system-ha)

## BGP e convergência

- [FortiOS CLI Reference — config router bgp](https://docs.fortinet.com/document/fortigate/latest/cli-reference/522620/config-router-bgp)
- [Technical Tip: All configurable BGP timers on the FortiGate explained](https://community.fortinet.com/t5/FortiGate/Technical-Tip-All-configurable-BGP-timers-on-the-FortiGate/ta-p/220047)
- [Technical Tip: Facilitating the fastest BGP route failovers](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Facilitating-the-fastest-BGP-route-failovers/ta-p/191989)

## HA + BGP

- [Technical Tip: Configuring FortiGate HA and BGP graceful restart to avoid traffic interruption during an HA failover](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Configuring-FortiGate-HA-and-BGP-graceful-restart/ta-p/193945)
- [Technical Tip: Understanding Graceful Restart and Non-Stop Forwarding for BGP in FortiGate HA](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Understanding-Graceful-Restart-and-Non-Stop/ta-p/302482)

> Os valores utilizados no LAB foram definidos e validados em testes controlados. Não devem ser aplicados automaticamente em ambiente produtivo sem considerar topologia, quantidade de peers, políticas de roteamento e requisitos de convergência.