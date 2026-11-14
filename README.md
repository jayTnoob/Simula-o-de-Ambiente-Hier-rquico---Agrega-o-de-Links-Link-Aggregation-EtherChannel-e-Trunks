# Simula-o-de-Ambiente-Hier-rquico---Agrega-o-de-Links-Link-Aggregation-EtherChannel-e-Trunks
Essa atividade Orientada pelo professor Jhordano, teve com intuito de fazer o treinamento para fazer o "Configurações de link aggregation." 

# Ambiente hierárquico: EtherChannel (Atividade 08)

Tutorial de configuração de agregação de links (EtherChannel) entre os switches de distribuição e núcleo, e entre os dois núcleos, no Cisco Packet Tracer.

- **Arquivo da simulação:** `Aula_de_switches_16_10.pkt`
- **Curso:** Redes de Computadores, IFRO
- **Disciplina:** Comutação de Redes Locais

## Sumário

1. [Visão geral da topologia](#visão-geral-da-topologia)
2. [Fundamentação: EtherChannel](#fundamentação-etherchannel)
3. [Mapeamento das interfaces](#mapeamento-das-interfaces)
4. [Passo a passo](#passo-a-passo)
5. [Comandos usados](#comandos-usados)
6. [Verificação](#verificação)
7. [Testes de conectividade e redundância](#testes-de-conectividade-e-redundância)
8. [Evidências](#evidências)

## Visão geral da topologia

| Camada | Equipamentos |
|---|---|
| Borda | Roteador R-BORDA-WAN (Cisco 1841) |
| Núcleo | SW-NUCLEO-01, SW-NUCLEO-02 |
| Distribuição | SW-DIST-01, SW-DIST-02 |
| Acesso | SW-ACESSO-01 a SW-ACESSO-04 (Cisco 2960-24TT) |
| Dispositivos finais | PC1 a PC4, Laptop1 a Laptop4, Server1 |

![Topologia](diagrama.png)

## Fundamentação: EtherChannel

- **EtherChannel:** agrupa vários links físicos entre dois switches em um único link lógico (port-channel). O spanning tree enxerga o grupo como um só enlace, então nenhuma das portas é bloqueada e a capacidade soma a de todas elas.
- **Capacidade:** com portas Fast Ethernet (100 Mbps), 4 portas formam um canal de 400 Mbps e 2 portas formam um de 200 Mbps. Nesta atividade, os valores de 4 Gbps e 2 Gbps do enunciado representam a capacidade planejada para cada canal: 4 links no canal entre os núcleos e 2 links em cada canal entre núcleo e distribuição.
- **Redundância:** se um cabo do canal cair, o tráfego continua pelos demais, e o canal perde apenas uma parte da capacidade.
- **Modo `on`:** formação estática, sem protocolo de negociação. Precisa ser configurado **nos dois lados** do canal. Se o professor pedir LACP, use `mode active` nos dois lados.

## Objetivo

- Criar um canal entre os dois núcleos (4 portas) e um canal entre cada núcleo e a sua distribuição (2 portas).
- Manter os port-channels em trunk, permitindo as VLANs 10 e 20.
- Validar a conectividade de ponta a ponta e a redundância com um cabo desconectado.

## Mapeamento das interfaces

### Port-channels

| Link | Portas | Canal | Capacidade |
|---|---|---|---|
| SW-NUCLEO-01 ↔ SW-NUCLEO-02 | NUCLEO-01: Fa6/1, Fa7/1, Fa8/1, Fa9/1 · NUCLEO-02: Fa0/1, Fa6/1, Fa8/1, Fa9/1 | Po1 | 4 Gbps |
| SW-NUCLEO-01 ↔ SW-DIST-01 | NUCLEO-01: Fa1/1, Fa2/1 · DIST-01: Fa4/1, Fa5/1 | Po2 | 2 Gbps |
| SW-NUCLEO-02 ↔ SW-DIST-02 | NUCLEO-02: Fa1/1, Fa2/1 · DIST-02: Fa4/1, Fa5/1 | Po2 | 2 Gbps |

O número do grupo (`channel-group`) tem significado local em cada switch. Por isso, o Po2 do SW-NUCLEO-01 e o Po2 do SW-NUCLEO-02 não precisam ser o mesmo canal, desde que cada um feche com o seu par.

### Fora dos canais

- Fa5/1 do SW-NUCLEO-01: Server1
- Fa0/1 do SW-NUCLEO-01 e Fa7/1 do SW-NUCLEO-02: R-BORDA-WAN
- Links distribuição → acesso: um cabo cada, mantidos como trunk simples (Atividade 07)

### VLANs permitidas nos trunks dos canais

| ID | Nome |
|---|---|
| 10 | ALUNOS |
| 20 | VISITANTES |

## Passo a passo

### Passo 1: confirmar as portas

Em cada switch de núcleo e distribuição, confirme em qual porta está cada cabo:

```
show cdp neighbors
```

Um `channel-group` na porta errada gera um canal que não fecha.

### Passo 2: SW-NUCLEO-01

```
enable
configure terminal
interface range fastEthernet 6/1 , fastEthernet 7/1 , fastEthernet 8/1 , fastEthernet 9/1
 channel-group 1 mode on
 exit
interface range fastEthernet 1/1 , fastEthernet 2/1
 channel-group 2 mode on
 exit
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 exit
interface port-channel 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 end
copy running-config startup-config
```

### Passo 3: SW-NUCLEO-02

```
enable
configure terminal
interface range fastEthernet 0/1 , fastEthernet 6/1 , fastEthernet 8/1 , fastEthernet 9/1
 channel-group 1 mode on
 exit
interface range fastEthernet 1/1 , fastEthernet 2/1
 channel-group 2 mode on
 exit
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 exit
interface port-channel 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 end
copy running-config startup-config
```

### Passo 4: SW-DIST-01 e SW-DIST-02 (mesmo bloco)

```
enable
configure terminal
interface range fastEthernet 4/1 , fastEthernet 5/1
 channel-group 2 mode on
 exit
interface port-channel 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 end
copy running-config startup-config
```

### Passo 5: aguardar a convergência

Configure os quatro switches em sequência, sem pausas longas, porque o modo `on` precisa estar nos dois lados. Depois espere cerca de 1 minuto até o spanning tree estabilizar e os cabos ficarem verdes na topologia.

## Comandos usados

| Comando | Função |
|---|---|
| `interface range <portas>` | Aplica a mesma configuração a várias portas |
| `channel-group <n> mode on` | Coloca a porta no grupo `n`, de forma estática |
| `interface port-channel <n>` | Entra na configuração da interface lógica do canal |
| `switchport mode trunk` | Define o port-channel como trunk |
| `switchport trunk allowed vlan 10,20` | Permite somente as VLANs 10 e 20 |
| `show etherchannel summary` | Resume os canais, o estado e as portas de cada um |
| `show interfaces trunk` | Lista as portas e port-channels em trunk |

## Verificação

Em cada switch de núcleo e distribuição:

```
show etherchannel summary
show interfaces trunk
```

Resultado esperado:

| Switch | `show etherchannel summary` | `show interfaces trunk` |
|---|---|---|
| SW-NUCLEO-01 | `Po1(SU)` com 4 portas `(P)` e `Po2(SU)` com 2 portas `(P)` | `Po1` e `Po2` em `trunking`, VLANs 10 e 20 |
| SW-NUCLEO-02 | `Po1(SU)` com 4 portas `(P)` e `Po2(SU)` com 2 portas `(P)` | `Po1` e `Po2` em `trunking`, VLANs 10 e 20 |
| SW-DIST-01 | `Po2(SU)` com `Fa4/1(P)` e `Fa5/1(P)` | `Po2` em `trunking`, VLANs 10 e 20 |
| SW-DIST-02 | `Po2(SU)` com `Fa4/1(P)` e `Fa5/1(P)` | `Po2` em `trunking`, VLANs 10 e 20 |

Legenda das flags: `S` = Layer 2, `U` = em uso, `P` = porta no port-channel. `Number of channel-groups in use` deve ser 2 nos núcleos e 1 nas distribuições.

### Problemas comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| Flag `SD` ou porta `(D)` | O outro lado não foi configurado, ou as portas têm modo/VLAN diferentes | Conferir o `channel-group` nos dois extremos |
| Flag `(I)` | A porta não entrou no canal | Verificar o número do grupo e o `mode on` dos dois lados |
| Porta em `err-disabled` | Configuração inconsistente durante a montagem | `shutdown` e `no shutdown` na porta, depois de acertar os dois lados |
| Comando rejeitado | O modelo do switch não suporta EtherChannel | Trocar o switch por um 2960 ou 3560 |
| Canal formado, mas sem trunk | Falta `switchport mode trunk` no port-channel | Reaplicar o bloco da interface `port-channel` |

## Testes de conectividade e redundância

Use **Desktop > Command Prompt** em cada dispositivo.

| # | Origem | Comando | Esperado |
|---|---|---|---|
| 1 | PC1 | `ping 192.168.10.4` | Responde (VLAN 10, ramos opostos) |
| 2 | Laptop1 | `ping 192.168.20.4` | Responde (VLAN 20, ramos opostos) |
| 3 | PC1 | `tracert 192.168.10.4` | Um salto (camada 2) |
| 4 | PC1 | `ping 192.168.20.1` | Falha (VLANs diferentes) |

O primeiro ping pode dar timeout por causa do ARP; repita uma vez antes de concluir.

### Teste de redundância

1. No PC1, rode `ping -t 192.168.10.4` e deixe o comando em execução.
2. Na topologia, apague **um** dos cabos de um dos canais (por exemplo, um dos quatro entre os núcleos).
3. O esperado é perder 1 ou 2 pacotes e voltar a responder, porque o tráfego segue pelos demais cabos do canal.
4. Registre o resultado, reconecte o cabo nas mesmas portas e pare o ping com `Ctrl+C`.
5. Confirme com `show etherchannel summary` que todas as portas voltaram para `(P)`.

## Evidências

- [ ] `show etherchannel summary` dos quatro switches de núcleo e distribuição
- [ ] `show interfaces trunk` dos quatro switches de núcleo e distribuição
- [ ] Topologia com todos os cabos entre núcleos e entre núcleo e distribuição verdes
- [ ] Ping entre dispositivos da mesma VLAN em ramos opostos
- [ ] `tracert` com um salto
- [ ] Ping entre VLANs diferentes falhando
- [ ] `ping -t` com um cabo do canal desconectado, mostrando poucos pacotes perdidos
- [ ] `show etherchannel summary` depois de reconectar o cabo

## Autor

João Arthur Amorim Maciel
