# FaceID Netsec

Sistema de portaria de condomínio que reconhece o morador pelo rosto e libera o acesso, mostrando ao porteiro quem chegou e se a entrada foi autorizada. Apresentado na **Expotech 2026 (FIEB)**.

A rede que sustenta o sistema, o **Condomínio Seguro**, foi projetada e simulada inteiramente no **Cisco Packet Tracer**.

> Projeto do curso Técnico em Redes de Computadores — FIEB / ITB Brasílio Flores de Azevedo, Barueri-SP. Período: jan/2024 a set/2026.

---

## O sistema

O reconhecimento roda **localmente em Python**, sem depender de serviço externo, com dois modelos:

| Modelo | Função |
|---|---|
| **YuNet** | Detecta o rosto na imagem e marca cinco pontos de referência. |
| **SFace** | Transforma o rosto em uma assinatura numérica (embedding) e compara com a dos moradores cadastrados. |

A comparação gera uma pontuação de semelhança, e essa pontuação decide se a entrada é liberada ou negada. O processamento local chega a **30 FPS**.

### Tela da portaria

O painel do porteiro foi feito em **Node-RED** e mostra, em tempo real:

- a foto capturada e o nome do morador identificado;
- se o acesso foi **liberado** ou **negado**, e a semelhança do reconhecimento;
- os contadores da sequência de testes: total de acessos, liberados, negados e a taxa de acerto;
- o registro com o horário de cada entrada.

Em uma sequência de testes foram 42 acessos, 26 liberados e 16 negados.

### Minha contribuição

- Arquitetura do sistema, da captação do rosto até a decisão de acesso e a exibição do resultado ao porteiro.
- Integração de todas as partes em um único fluxo.
- Seleção dos comandos Cisco usados no Packet Tracer para configurar a rede.

---

## A rede: Condomínio Seguro

Topologia hierárquica com internet, provedor, dois roteadores de borda (**RT-BORDA-1** e **RT-BORDA-2**), dois núcleos (**SW-CORE-1** e **SW-CORE-2**) ligados por EtherChannel, e quatro switches de acesso: **SW-PORTARIA**, **SW-BLOCOA**, **SW-BLOCOB** e **SW-SERVIDORES**.

### VLANs

| VLAN | Função | O que fica nela |
|---|---|---|
| 10 | Servidores | DHCP, DNS, NTP, broker MQTT |
| 20 | Automação | Trava e sensores |
| 30 | CFTV | Câmeras |
| 40 | Moradores | PCs e celulares dos moradores |
| 50 | Visitantes | Rede de visitantes |
| 99 | Gerência | Administração dos equipamentos |
| 999 | Nativa | Sem nenhum dispositivo |

### Os 9 passos da configuração

**1. VLANs e 802.1Q.** A rede é dividida em 6 VLANs por função. O broadcast de um morador só chega a quem está na VLAN 40. As portas de usuário ficam em modo access, sem negociar trunk, e a VLAN nativa dos trunks é a 999, que não tem nenhum dispositivo. Isso evita switch spoofing e double tagging.

```
SW-BLOCOA(config)# interface fa0/13
SW-BLOCOA(config-if)# switchport mode access
SW-BLOCOA(config-if)# switchport access vlan 40
SW-BLOCOA(config)# interface range gi0/1 - 2
SW-BLOCOA(config-if-range)# switchport trunk native vlan 999
SW-BLOCOA(config-if-range)# switchport nonegotiate
```

**2. EtherChannel com LACP.** Os dois núcleos se ligam por dois cabos agrupados em um Port-channel. A banda total dobra, mas cada fluxo vai por um cabo só, escolhido por hash. Se um dos cabos cai, o link continua de pé, sem o spanning-tree precisar reconvergir. O Port-channel é um trunk: leva as VLANs entre os núcleos.

```
SW-CORE-1(config)# interface range gi1/0/1 - 2
SW-CORE-1(config-if-range)# channel-group 1 mode active
SW-CORE-1# show etherchannel summary
```

**3. Rapid-PVST+.** Cada switch de acesso tem um uplink para cada núcleo. O Rapid-PVST+ roda uma árvore por VLAN: encaminha por um uplink e bloqueia o outro, que fica de reserva e assume em segundos se o ativo cair. A raiz é dividida entre os núcleos, então o tráfego fica repartido entre os dois.

```
SW-CORE-1(config)# spanning-tree mode rapid-pvst
SW-CORE-1(config)# spanning-tree vlan 10,20,99,999 priority 24576
SW-CORE-1(config)# spanning-tree vlan 30,40,50 priority 28672
```

**4. HSRP v2.** Cada VLAN tem um IP virtual compartilhado pelos dois núcleos. Um é Active e responde por ele; se cair, o outro assume o mesmo IP e o mesmo MAC virtual quando o hold time vence. Com preempt, o papel de Active volta ao titular. Em cada VLAN, o Active também é a raiz do spanning-tree.

```
SW-CORE-1(config-if)# standby version 2
SW-CORE-1(config-if)# standby 10 ip 10.10.10.1
SW-CORE-1(config-if)# standby 10 priority 150
SW-CORE-1(config-if)# standby 10 preempt
```

**5. OSPF área 0.** Núcleos e roteadores de borda viram vizinhos OSPF na área 0. Os roteadores aprendem as redes das VLANs e a resposta da internet encontra o caminho de volta. Os roteadores anunciam a rota padrão, e com `passive-interface default` nenhuma porta de usuário fala OSPF.

```
SW-CORE-1(config-router)# passive-interface default
SW-CORE-1(config-router)# no passive-interface vlan999
RT-BORDA-2(config-router)# default-information originate
```

**6. NAT seletivo.** O roteador de borda só traduz as VLANs 10, 40 e 50, então o morador navega. A câmera (VLAN 30) não tem tradução: o pacote até sai, mas com IP privado e não tem caminho de volta pela internet.

```
RT-BORDA-1(config)# access-list 111 permit ip 10.10.0.0 0.0.255.255 any
RT-BORDA-1(config)# ip nat inside source list 111 interface gi0/0 overload
RT-BORDA-1# show ip nat translations
```

**7. DHCP falso × DHCP Snooping.** Um único servidor DHCP atende todas as VLANs: o `ip helper-address` no núcleo transforma o broadcast do cliente em um pedido direto ao servidor. No ataque, alguém liga um servidor DHCP falso numa porta comum; se ele responder primeiro, o cliente aceita o gateway dele. O DHCP Snooping só aceita OFFER e ACK vindos de porta confiável (os uplinks e a porta do servidor verdadeiro), então a resposta falsa é descartada. Ele também monta a tabela de vínculos MAC, IP, porta e VLAN que a inspeção de ARP usa no passo seguinte.

```
SW-CORE-2(config-if)# ip helper-address 10.10.10.20
SW-BLOCOB(config)# ip dhcp snooping vlan 10,20,30,40,50
SW-BLOCOB(config)# no ip dhcp snooping information option
SW-BLOCOB(config-if)# ip dhcp snooping trust
```

**8. ARP poisoning × Dynamic ARP Inspection.** No ARP poisoning, o atacante manda respostas ARP falsas dizendo que o IP do gateway pertence ao MAC dele. Os aparelhos da VLAN atualizam a tabela ARP e passam a mandar o tráfego para o atacante, que pode ler ou alterar os dados antes de repassar (man-in-the-middle). O Dynamic ARP Inspection confere cada ARP que chega por porta não confiável contra a tabela do DHCP Snooping: se o par IP e MAC não bate, o ARP é descartado.

```
SW-BLOCOB(config)# ip arp inspection vlan 40,50
```

**9. Port-Security.** O port-security com sticky grava o MAC do primeiro aparelho que usa a porta. Se alguém desconectar o PC da portaria e ligar outro aparelho no lugar, tentando entrar na VLAN 10 dos servidores, o MAC diferente dispara a violação. No modo restrict, o switch descarta o tráfego do aparelho estranho e registra a violação, sem desligar a porta.

```
SW-PORTARIA(config-if-range)# switchport mode access
SW-PORTARIA(config-if-range)# switchport port-security
SW-PORTARIA(config-if-range)# switchport port-security mac-address sticky
SW-PORTARIA(config-if-range)# switchport port-security violation restrict
```

### Serviços

DHCP com relay, DNS e NTP, todos na VLAN 10 de servidores. Os dispositivos de automação se comunicam por MQTT.

---

## Tecnologias

| Área | Itens |
|---|---|
| Redes | VLAN, 802.1Q, EtherChannel/LACP, Rapid-PVST+, HSRP v2, OSPF, NAT, DHCP (com relay), DNS, NTP, ACLs |
| Segurança | DHCP Snooping, Dynamic ARP Inspection, Port-Security |
| Reconhecimento facial | Python, YuNet, SFace |
| Integração | MQTT, Node-RED |
| Simulação | Cisco Packet Tracer |

---

## Estrutura do repositório

```
faceid-netsec/
├── README.md
├── reconhecimento/   # código Python (YuNet + SFace)
├── rede/             # arquivo .pkt, topologia e plano de endereçamento
├── configs/          # configurações dos equipamentos (sem senhas)
└── docs/             # documentação dos 9 passos e apresentação
```

---

## Autor

**Isaac Ramos Pereira** — Técnico em Redes de Computadores (cursando), Barueri-SP.

Cisco Networking Academy: IT Essentials (80h), CCNA Introduction to Networks (160h) e CCNA Switching, Routing and Wireless Essentials (70h).

[LinkedIn](https://www.linkedin.com/in/isaac-ramos-pereira-526273328)
