# 🛡️ VPN 2: Site-to-Site Basada en Políticas (IKEv2)

## 📖 Descripción
Esta es la segunda implementación de la serie. En este laboratorio, evolucionamos la topología anterior configurando una **VPN Site-to-Site basada en políticas**, pero esta vez dando el salto a **IKEv2**.

A diferencia de IKEv1, IKEv2 utiliza una estructura jerárquica y modular más robusta para la Fase 1, separando la configuración en cuatro componentes clave:
1. **Propuesta (Proposal):** Define los algoritmos de encriptación, integridad y grupo DH.
2. **Política (Policy):** Asocia una o más propuestas.
3. **Llavero (Keyring):** Almacena de forma segura las claves precompartidas (PSK) por cada peer.
4. **Perfil (Profile):** Amarra la identidad local y remota, los métodos de autenticación y el llavero.

El tráfico a encriptar sigue siendo clasificado mediante una ACL extendida y aplicado a la interfaz física con un Crypto Map.

## 🗺️ Topología y Direccionamiento

| Dispositivo | Interfaz | Dirección IP / Máscara | Descripción |
| :--- | :--- | :--- | :--- |
| **ISP** | G1/0 | 11.79.1.1 / 30 | Enlace hacia R1 |
| **ISP** | G2/0 | 11.79.2.1 / 30 | Enlace hacia R2 |
| **R1** | G1/0 (WAN) | 11.79.1.2 / 30 | Enlace hacia ISP |
| **R1** | G2/0 (LAN) | 10.24.79.1 / 24 | Red Local R1 |
| **R2** | G2/0 (WAN) | 11.79.2.2 / 30 | Enlace hacia ISP |
| **R2** | G1/0 (LAN) | 20.24.11.1 / 24 | Red Local R2 |

## 🔐 Parámetros Criptográficos

**Fase 1 (IKEv2):**
* **Encriptación:** AES-CBC 256
* **Integridad (Hash):** SHA-256
* **Grupo Diffie-Hellman:** 14
* **Autenticación:** Pre-Shared Key (PSK) mediante Keyring y Profile.

**Fase 2 (IPSec):**
* **Protocolo:** ESP
* **Transform-Set:** AES 256 / SHA-256 HMAC
* **Modo:** Tunnel

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

<details>
<summary><b>2. R1</b></summary>

en
conf t
host R1

! Configuración de Interfaces
interface GigabitEthernet1/0
 description WAN a ISP
 ip address 11.79.1.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2/0
 ip address 10.24.79.1 255.255.255.0
 no shutdown
 exit

! Ruta por defecto hacia el ISP
ip route 0.0.0.0 0.0.0.0 11.79.1.1

! --- FASE 1: IKEv2 ---
! 1. Propuesta IKEv2
crypto ikev2 proposal IKEv2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! 2. Política IKEv2
crypto ikev2 policy IKEv2-POL
 proposal IKEv2-PROP
 exit

! 3. Keyring (Llavero de contraseñas)
crypto ikev2 keyring IKEv2-KEY
 peer R2
  address 11.79.2.2
  pre-shared-key abi1179
 exit
 exit

! 4. Perfil IKEv2 (Amarra todo lo anterior)
crypto ikev2 profile IKEv2-PROF
 match identity remote address 11.79.2.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2-KEY
 exit

! --- FASE 2: IPSec ---
! 5. Transform Set
crypto ipsec transform-set TS-IKEv2 esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! 6. ACL para Tráfico Interesante (De R1 LAN a R2 LAN)
ip access-list extended VPN-TRAFFIC
 permit ip 10.24.79.0 0.0.0.255 20.24.11.0 0.0.0.255
 exit

! 7. Crypto Map
crypto map MAP-IKEv2 10 ipsec-isakmp
 set peer 11.79.2.2
 set transform-set TS-IKEv2
 set ikev2-profile IKEv2-PROF
 match address VPN-TRAFFIC
 exit

! 8. Aplicar a la WAN
interface GigabitEthernet1/0
 crypto map MAP-IKEv2
 exit
end
write memory


<details>
<summary><b>3. R2</b></summary>

enable
configure terminal
hostname R2

! Configuración de Interfaces
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

! Ruta por defecto hacia el ISP
ip route 0.0.0.0 0.0.0.0 11.79.2.1

! --- FASE 1: IKEv2 ---
! 1. Propuesta IKEv2
crypto ikev2 proposal IKEv2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! 2. Política IKEv2
crypto ikev2 policy IKEv2-POL
 proposal IKEv2-PROP
 exit

! 3. Keyring (Llavero de contraseñas)
crypto ikev2 keyring IKEv2-KEY
 peer R1
  address 11.79.1.2
  pre-shared-key abi1179
 exit
 exit

! 4. Perfil IKEv2
crypto ikev2 profile IKEv2-PROF
 match identity remote address 11.79.1.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2-KEY
 exit

! --- FASE 2: IPSec ---
! 5. Transform Set
crypto ipsec transform-set TS-IKEv2 esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! 6. ACL para Tráfico Interesante (De R2 LAN a R1 LAN)
ip access-list extended VPN-TRAFFIC
 permit ip 20.24.11.0 0.0.0.255 10.24.79.0 0.0.0.255
 exit

! 7. Crypto Map
crypto map MAP-IKEv2 10 ipsec-isakmp
 set peer 11.79.1.2
 set transform-set TS-IKEv2
 set ikev2-profile IKEv2-PROF
 match address VPN-TRAFFIC
 exit

! 8. Aplicar a la WAN
interface GigabitEthernet2/0
 crypto map MAP-IKEv2
 exit
end
write memory

<details>
<summary><b>4. PCs</b></summary>

! PC en LAN R1
ip 10.24.79.50 255.255.255.0 10.24.79.1

! PC en LAN R2
ip 20.24.11.50 255.255.255.0 20.24.11.1



 exit
end
write memory
