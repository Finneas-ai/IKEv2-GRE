# VPN Site-to-Site GRE sobre IPsec (IKEv2)

Video de referencia: https://www.youtube.com/watch?v=e4mLPgtiU6Q

## Descripción

Igual que el escenario anterior, pero la negociación se realiza con **IKEv2** (`crypto ikev2 proposal/policy/keyring/profile`) en lugar de `crypto isakmp`. El túnel `Tunnel0` sigue siendo GRE (`tunnel mode gre ip`) y se protege mediante `crypto map` con ACL que selecciona el tráfico GRE entre los peers, referenciando el `ikev2-profile`.



## Topología

```
LAN A (10.6.82.0/25) -- Peer A --GRE Tunnel0 (30.6.82.0/30), protegido por IPsec IKEv2-- ISP -- Peer B -- LAN B (10.6.82.128/25)
```

## Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara            |
|-------------|----------|--------------------------|
| ISP         | e0/0     | 20.6.82.1 /30            |
| ISP         | e0/1     | 20.6.82.5 /30            |
| Peer A      | e0/0     | 20.6.82.2 /30            |
| Peer A      | e0/1     | 10.6.82.1 /25 (LAN A)    |
| Peer A      | Tunnel0  | 30.6.82.1 /30 (GRE)      |
| Peer B      | e0/0     | 20.6.82.6 /30            |
| Peer B      | e0/1     | 10.6.82.129 /25 (LAN B)  |
| Peer B      | Tunnel0  | 30.6.82.2 /30 (GRE)      |
| PC1 (VPCS)  | -        | 10.6.82.10/25, gw 10.6.82.1   |
| PC2 (VPCS)  | -        | 10.6.82.138/25, gw 10.6.82.129 |

## Parámetros IKEv2 / IPsec

| Parámetro   | Valor                              |
|-------------|-------------------------------------|
| encryption  | aes-cbc-256                        |
| integrity   | sha256                             |
| group (DH)  | 14                                  |
| keyring PSK | CLAVE0682                          |
| ACL protegida | tráfico GRE entre IPs públicas de los peers |
| tunnel mode | gre ip                              |

## Configuración

### ISP

```
interface e0/0
 ip address 20.6.82.1 255.255.255.252
 no shutdown
interface e0/1
 ip address 20.6.82.5 255.255.255.252
 no shutdown
```

### Peer A

```
interface e0/0
 ip address 20.6.82.2 255.255.255.252
 no shutdown
interface e0/1
 ip address 10.6.82.1 255.255.255.128
 no shutdown
ip route 0.0.0.0 0.0.0.0 20.6.82.1

crypto ikev2 proposal PROP0682
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL0682
 proposal PROP0682

crypto ikev2 keyring KEY0682
 peer R2
  address 20.6.82.6
  pre-shared-key CLAVE0682

crypto ikev2 profile PROF0682
 match identity remote address 20.6.82.6 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY0682

crypto ipsec transform-set TS0682 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended GRE-ACL
 permit gre host 20.6.82.2 host 20.6.82.6

crypto map MAPGRE 10 ipsec-isakmp
 set peer 20.6.82.6
 set transform-set TS0682
 set ikev2-profile PROF0682
 match address GRE-ACL

interface Tunnel0
 ip address 30.6.82.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 20.6.82.6
 tunnel mode gre ip

interface e0/0
 crypto map MAPGRE

ip route 10.6.82.128 255.255.255.128 Tunnel0
```

### Peer B

```
interface e0/0
 ip address 20.6.82.6 255.255.255.252
 no shutdown
interface e0/1
 ip address 10.6.82.129 255.255.255.128
 no shutdown
ip route 0.0.0.0 0.0.0.0 20.6.82.5

crypto ikev2 proposal PROP0682
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL0682
 proposal PROP0682

crypto ikev2 keyring KEY0682
 peer R1
  address 20.6.82.2
  pre-shared-key CLAVE0682

crypto ikev2 profile PROF0682
 match identity remote address 20.6.82.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY0682

crypto ipsec transform-set TS0682 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended GRE-ACL
 permit gre host 20.6.82.6 host 20.6.82.2

crypto map MAPGRE 10 ipsec-isakmp
 set peer 20.6.82.2
 set transform-set TS0682
 set ikev2-profile PROF0682
 match address GRE-ACL

interface Tunnel0
 ip address 30.6.82.2 255.255.255.252
 tunnel source e0/0
 tunnel destination 20.6.82.2
 tunnel mode gre ip

interface e0/0
 crypto map MAPGRE

ip route 10.6.82.0 255.255.255.128 Tunnel0
```

### Hosts finales (VPCS)

```
PC1: ip 10.6.82.10/25 10.6.82.1
PC2: ip 10.6.82.138/25 10.6.82.129
```

## Verificación

```
show interface tunnel 0
show crypto ikev2 sa
show crypto ipsec sa
show crypto map
ping 10.6.82.138 source 10.6.82.10
```
