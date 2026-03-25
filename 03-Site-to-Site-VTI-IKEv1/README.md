# 🛡️ VPN 3: Site-to-Site Basada en Enrutamiento (VTI) con IKEv1

## 📖 Descripción
En este laboratorio se configura una **VPN Site-to-Site basada en enrutamiento** utilizando **IKEv1** y una Interfaz de Túnel Virtual (**VTI**). 

A diferencia de las VPNs basadas en políticas (que utilizan ACLs y Crypto Maps), este enfoque crea una interfaz lógica dedicada (`Tunnel0`). Todo el tráfico que la tabla de enrutamiento envíe hacia esta interfaz es automáticamente encriptado. Este diseño es el estándar moderno, ya que es más escalable, simplifica la configuración y permite correr protocolos de enrutamiento dinámico (como OSPF o EIGRP) sobre el túnel IPsec.

*Nota de emulación:* Se ha aplicado el comando `no ip cef` en los routers de los extremos para evitar problemas conocidos de reenvío en interfaces VTI dentro de entornos virtualizados (GNS3 / PNETLab).

## 🗺️ Topología y Direccionamiento

| Dispositivo | Interfaz | Dirección IP / Máscara | Descripción |
| :--- | :--- | :--- | :--- |
| **ISP** | G1/0 | 11.79.1.1 / 30 | Enlace hacia R1 |
| **ISP** | G2/0 | 11.79.2.1 / 30 | Enlace hacia R2 |
| **R1** | G1/0 (WAN) | 11.79.1.2 / 30 | Enlace hacia ISP |
| **R1** | G2/0 (LAN) | 10.24.79.1 / 24 | Red Local R1 |
| **R1** | Tunnel 0 | 172.16.79.1 / 30 | Interfaz VTI |
| **R2** | G2/0 (WAN) | 11.79.2.2 / 30 | Enlace hacia ISP |
| **R2** | G1/0 (LAN) | 20.24.11.1 / 24 | Red Local R2 |
| **R2** | Tunnel 0 | 172.16.79.2 / 30 | Interfaz VTI |

## 🔐 Parámetros Criptográficos

**Fase 1 (ISAKMP):**
* **Encriptación:** AES 256
* **Hash:** SHA-256
* **Autenticación:** Pre-Shared Key (PSK)
* **Grupo Diffie-Hellman:** 14

**Fase 2 (IPSec):**
* **Protocolo:** ESP
* **Transform-Set:** AES 256 / SHA-256 HMAC
* **Modo:** Tunnel
* **Aplicación:** Perfil IPsec asociado directamente a la interfaz de túnel.

---

## ⚙️ Configuraciones

<details>
<summary><b>1. Router ISP</b></summary>

```cisco
conf t
host ISP

interface GigabitEthernet1/0
 description Hacia R1
 ip address 11.79.1.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2/0
 description Hacia R2
 ip address 11.79.2.1 255.255.255.252
 no shutdown
 exit
end
write memory
```
<details>
<summary><b>2. R1</b></summary>

```cisco
en
conf t
host R1

! Configuración de interfaces físicas
interface GigabitEthernet1/0
 description WAN a ISP
 ip address 11.79.1.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2/0
 ip address 10.24.79.1 255.255.255.0
 no shutdown
 exit

! Desactivar CEF (Workaround para emuladores)
no ip cef

! Ruta por defecto hacia el ISP
ip route 0.0.0.0 0.0.0.0 11.79.1.1

! Fase 1: IKEv1
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 exit
crypto isakmp key abi1179 address 11.79.2.2

! Fase 2: IPSec (Sin ACL ni Crypto Map)
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! Perfil IPSec (Amarra la criptografía al túnel)
crypto ipsec profile PROF-VTI
 set transform-set TS-VTI
 exit

! Creación de la Interfaz de Túnel (VTI)
interface Tunnel0
 ip address 172.16.79.1 255.255.255.252
 tunnel source GigabitEthernet1/0
 tunnel destination 11.79.2.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI
 no shutdown
 exit

! Enrutamiento: Todo lo que vaya a la LAN de R2, mételo al túnel
ip route 20.24.11.0 255.255.255.0 Tunnel0
end
write memory
```

<details>
<summary><b>3. R2</b></summary>

```cisco
enable
configure terminal
hostname R2

! Configuración de interfaces físicas
interface GigabitEthernet2/0
 description WAN a ISP
 ip address 11.79.2.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet1/0
 description LAN R2
 ip address 20.24.11.1 255.255.255.0
 no shutdown
 exit

! Desactivar CEF (Workaround para emuladores)
no ip cef

! Ruta por defecto hacia el ISP
ip route 0.0.0.0 0.0.0.0 11.79.2.1

! Fase 1: IKEv1
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 exit
crypto isakmp key abi1179 address 11.79.1.2

! Fase 2: IPSec
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! Perfil IPSec
crypto ipsec profile PROF-VTI
 set transform-set TS-VTI
 exit

! Creación de la Interfaz de Túnel (VTI)
interface Tunnel0
 ip address 172.16.79.2 255.255.255.252
 tunnel source GigabitEthernet2/0
 tunnel destination 11.79.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI
 no shutdown
 exit

! Enrutamiento: Todo lo que vaya a la LAN de R1, mételo al túnel
ip route 10.24.79.0 255.255.255.0 Tunnel0
end
write memory
```

<details>
<summary><b>4. PCs</b></summary>

```cisco
! PC en LAN R1
ip 10.24.79.50 255.255.255.0 10.24.79.1

! PC en LAN R2
ip 20.24.11.50 255.255.255.0 20.24.11.1
