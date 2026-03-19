# 📄 README — VPN Client-to-Site (Topología 2)
### Proyecto de Redes | PNETLab | Seguridad Perimetral con FortiGate y Cisco IOS





---

## 1. Objetivo de la Red

El objetivo de esta práctica es implementar y verificar un túnel **VPN IPsec Site-to-Site** entre un firewall **FortiGate (FGT-Server)** y un **Router Cisco vIOS (Router-Cliente)**, simulando la comunicación segura entre una sede central y un sitio remoto a través de un enlace WAN simulado en **PNETLab**.

La VPN garantiza:
- **Confidencialidad** del tráfico entre ambas LAN mediante cifrado ESP-DES.
- **Autenticación** por clave precompartida (PSK).
- **Integridad** mediante SHA-HMAC.
- **Perfect Forward Secrecy (PFS)** con Diffie-Hellman Grupo 5.

---

## 2. Topología de Red

### Diagrama Lógico

<img width="1023" height="830" alt="image" src="https://github.com/user-attachments/assets/e2be7713-d0a1-4f3e-9953-6e3021b192f3" />




### Descripción de Roles

| Dispositivo       | Rol                    | Plataforma       |
|-------------------|------------------------|------------------|
| FortiGate (FGT)   | Firewall / VPN Server  | FortiOS (PNETLab)|
| Cisco vIOS        | Router / VPN Cliente   | IOS (PNETLab)    |
| VPC (Local)       | PC de prueba LAN local | VPCS (PNETLab)   |
| VPC (Remota)      | PC de prueba LAN remota| VPCS (PNETLab)   |

---

## 3. Esquema de Direccionamiento IP

### Red WAN — Enlace de Internet Simulado (`172.20.11.0/30`)

| Dispositivo              | Interfaz    | Dirección IP    |
|--------------------------|-------------|-----------------|
| Router Cisco (vIOS)      | Gi0/0       | 172.20.11.1 /30 |
| FortiGate (FGT-Server)   | port1       | 172.20.11.2 /30 |

### Red LAN Servidor — Sitio Central (`10.20.11.0/24`)

| Dispositivo              | Interfaz    | Dirección IP      |
|--------------------------|-------------|-------------------|
| FortiGate Gateway        | port2       | 10.20.11.254 /24  |
| PC Local (VPC)           | eth0        | 10.20.11.10 /24   |

### Red LAN Cliente — Sitio Remoto (`10.24.65.0/24`)

| Dispositivo              | Interfaz    | Dirección IP      |
|--------------------------|-------------|-------------------|
| Router Cisco Gateway     | Gi0/1       | 10.24.65.254 /24  |
| PC Remota (VPC)          | eth0        | 10.24.65.10 /24   |

---

## 4. Configuraciones Utilizadas

### 4.1 Router Cisco vIOS — `Router-Cliente`

```
configure terminal
hostname Router-Cliente

! === INTERFACES ===
interface GigabitEthernet0/0
 ip address 172.20.11.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet0/1
 ip address 10.24.65.254 255.255.255.0
 no shutdown
 exit

! === RUTA PREDETERMINADA ===
ip route 0.0.0.0 0.0.0.0 172.20.11.2

! === FASE 1 — ISAKMP Policy ===
crypto isakmp policy 10
 encr des
 hash sha
 authentication pre-share
 group 5
 exit

crypto isakmp key Matricula20241165 address 172.20.11.2

! === FASE 2 — Transform Set e IPsec ===
crypto ipsec transform-set TS esp-des esp-sha-hmac
 mode tunnel
 exit

! === ACL de tráfico interesante ===
access-list 100 permit ip 10.24.65.0 0.0.0.255 10.20.11.0 0.0.0.255

! === CRYPTO MAP ===
crypto map CMAP 10 ipsec-isakmp
 set peer 172.20.11.2
 set transform-set TS
 match address 100
 set pfs group5
 exit

! === Aplicar mapa en WAN ===
interface GigabitEthernet0/0
 crypto map CMAP
 exit

end
write memory
```

---

### 4.2 FortiGate — `FGT-Server`

```
! === HOSTNAME ===
config system global
 set hostname FGT-Server
end

! === INTERFACES ===
config system interface
 edit port1
  set mode static
  set ip 172.20.11.2 255.255.255.252
  set allowaccess ping
 next
 edit port2
  set mode static
  set ip 10.20.11.254 255.255.255.0
  set allowaccess ping
 next
end

! === VPN IPSEC — FASE 1 ===
config vpn ipsec phase1-interface
 edit "VPN-Clientes"
  set interface "port1"
  set ike-version 1
  set mode main
  set peertype any
  set net-device disable
  set proposal aes128-sha1
  set dhgrp 5
  set remote-gw 172.20.11.1
  set psksecret Matricula20241165
 next
end

! === VPN IPSEC — FASE 2 ===
config vpn ipsec phase2-interface
 edit "VPN-Clientes"
  set phase1name "VPN-Clientes"
  set proposal aes128-sha1
  set dhgrp 5
  set src-subnet 10.20.11.0 255.255.255.0
  set dst-subnet 10.24.65.0 255.255.255.0
  set auto-negotiate enable
 next
end

! === RUTAS ESTÁTICAS ===
config router static
 edit 1
  set device port1
  set gateway 172.20.11.1
 next
 edit 2
  set dst 10.24.65.0 255.255.255.0
  set device "VPN-Clientes"
 next
end

! === POLÍTICAS DE FIREWALL ===
config firewall policy
 edit 1
  set name "LAN_to_VPN"
  set srcintf "port2"
  set dstintf "VPN-Clientes"
  set srcaddr "all"
  set dstaddr "all"
  set action accept
  set schedule "always"
  set service "ALL"
 next
 edit 2
  set name "VPN_to_LAN"
  set srcintf "VPN-Clientes"
  set dstintf "port2"
  set srcaddr "all"
  set dstaddr "all"
  set action accept
  set schedule "always"
  set service "ALL"
 next
end
```

---

## 5. Parámetros del Túnel VPN IPsec

| Parámetro            | Valor configurado         |
|----------------------|---------------------------|
| Tipo de VPN          | IPsec Site-to-Site        |
| Modo IKE             | Main Mode (IKEv1)         |
| Cifrado (Fase 1)     | AES-128 / DES             |
| Hash (Fase 1)        | SHA-1                     |
| Autenticación        | Pre-Shared Key (PSK)      |
| Clave PSK            | `Matricula20241165`       |
| Grupo Diffie-Hellman | Grupo 5 (1536-bit)        |
| Cifrado (Fase 2)     | ESP-DES / AES-128         |
| Hash (Fase 2)        | SHA-HMAC / SHA-1          |
| PFS                  | Habilitado — Grupo 5      |
| Peer FortiGate       | 172.20.11.2               |
| Peer Router Cisco    | 172.20.11.1               |

---

## 6. Verificación del Funcionamiento

### Desde el Router Cisco

Para verificar el estado del túnel:

```bash
# Ver sesiones IKE activas
show crypto isakmp sa

# Ver asociaciones IPsec establecidas
show crypto ipsec sa

# Probar conectividad hacia la LAN del servidor
ping 10.20.11.10 source 10.24.65.254
```

### Desde VPC Remota

```bash
# Ping desde PC remota hacia PC local a través del túnel VPN
ping 10.20.11.10
```

### Desde FortiGate

```bash
# Diagnóstico del túnel VPN
diagnose vpn ike gateway list
diagnose vpn tunnel list
```

---

## 7. Flujo del Tráfico VPN

```
PC Remota (10.24.65.10)
      │
      ▼
Router-Cliente (Gi0/1: 10.24.65.254)
      │  [Tráfico cifrado con ESP]
      ▼  [Túnel IPsec activo sobre WAN]
      │
Router-Cliente (Gi0/0: 172.20.11.1) ◄──── CRYPTO MAP CMAP aplicado aquí
      │
      ▼ ──────── WAN 172.20.11.0/30 ────────────►
                                                  │
                                    FGT-Server (port1: 172.20.11.2)
                                                  │ [Descifrado ESP]
                                                  ▼
                                    FGT-Server (port2: 10.20.11.254)
                                                  │
                                                  ▼
                                          PC Local (10.20.11.10)
```

---

## 8. Entorno de Laboratorio

| Componente       | Detalle                                  |
|------------------|------------------------------------------|
| Plataforma       | PNETLab (PNETLab VM sobre VMware/VBox)   |
| Firewall         | FortiGate (FortiOS) — imagen PNETLab     |
| Router           | Cisco vIOS — imagen GNS3/PNETLab         |
| PCs de prueba    | VPCS integrado en PNETLab                |
| Protocolo VPN    | IPsec (IKEv1) Site-to-Site               |
| Enlace WAN       | Punto a punto simulado /30               |

---

## 9. Notas Adicionales

- El PSK `Matricula20241165` debe coincidir **exactamente** en ambos extremos del túnel.
- El FortiGate utiliza **Fase 1 con AES-128** y el router Cisco usa **DES** — asegurarse de que las propuestas sean compatibles o unificarlas para evitar negociación fallida.
- La política de firewall en FortiGate **debe permitir el tráfico en ambas direcciones** (LAN→VPN y VPN→LAN).
- El `crypto map` en el router Cisco **debe estar aplicado en la interfaz WAN (Gi0/0)**, no en la LAN.
- La ACL 100 define el **tráfico interesante** que activa el túnel VPN.

---

*Documentación generada para el proyecto de Seguridad en Redes — Configuración VPN IPsec Client-to-Site en PNETLab.*
