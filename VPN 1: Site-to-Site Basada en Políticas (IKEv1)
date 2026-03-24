# 🛡️ VPN 1: Site-to-Site Basada en Políticas (IKEv1)

## 📖 Descripción
Esta es la primera de una serie de 9 implementaciones de VPNs. En este laboratorio, se configura una **VPN Site-to-Site IPsec basada en políticas** utilizando **IKEv1**. La comunicación se establece entre dos routers (R1 y R2) a través de un router ISP simulado. 

El tráfico interesante (aquel que debe ser encriptado) se define mediante una Lista de Control de Acceso (ACL) extendida y se aplica a las interfaces físicas externas mediante un **Crypto Map**.

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

**Fase 1 (ISAKMP):**
* **Encriptación:** AES 256
* **Hash:** SHA-256
* **Autenticación:** Pre-Shared Key (PSK)
* **Grupo Diffie-Hellman:** 14

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
 exit
end
write memory
