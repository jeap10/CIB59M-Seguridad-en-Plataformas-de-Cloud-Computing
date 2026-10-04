# 🌐 Laboratorio: Conectividad entre Redes Virtuales en Azure (VNet Peering + Bastion + NAT Gateway)

Este repositorio documenta la implementación de una arquitectura de red en Microsoft Azure que conecta dos redes virtuales (VNets) mediante **peering**, con acceso remoto seguro a través de **Azure Bastion** y salida a Internet controlada mediante **NAT Gateway**.

## 📐 Arquitectura

La arquitectura consiste en dos redes virtuales (VNET1 y VNET2), cada una con una máquina virtual Windows, conectadas entre sí mediante peering para permitir comunicación privada bidireccional.

![Diagrama de arquitectura]

<img width="856" height="508" alt="Lab_1" src="https://github.com/user-attachments/assets/06f11c8e-2ffa-4ea3-a6f0-649277eea425" />



| Componente | Descripción |
|---|---|
| **VNET1** | Red virtual principal, contiene la VM `V1`, el NSG1, la subred de Bastion y el NAT Gateway |
| **VNET2** | Red virtual secundaria, contiene la VM `V2` y el NSG2 |
| **Peering** | Conecta VNET1 y VNET2 para tráfico privado bidireccional |
| **Azure Bastion** | Permite acceso RDP seguro a las VMs sin exponerlas con IP pública |
| **NAT Gateway** | Da salida a Internet a V1 sin necesidad de IP pública propia |

---

## 📦 Paso 0: Creación del Resource Group

Antes de crear cualquier recurso, se creó un **Resource Group** (grupo de recursos), que funciona como el "contenedor lógico" donde viven organizados todos los componentes del laboratorio (VNets, VMs, NSGs, Bastion, NAT Gateway, etc.).

1. En el portal de Azure, buscar **"Resource groups"** > **"+ Create"**.
2. Seleccionar la suscripción correspondiente.
3. Asignar un nombre descriptivo (ej. `CIB59-Palacios`).
4. Elegir la región (ej. `East US`) — **todos los recursos del laboratorio deben crearse en la misma región** para evitar problemas de compatibilidad, especialmente con el peering y el NAT Gateway.
5. Clic en **"Review + create"** y luego **"Create"**.

![Creación del Resource Group]
<img width="1917" height="965" alt="Captura de pantalla 2026-10-04 000745" src="https://github.com/user-attachments/assets/8a36720b-8f7e-46ec-925e-f1022815d618" />

<img width="1917" height="965" alt="Captura de pantalla 2026-10-04 000806" src="https://github.com/user-attachments/assets/74bc72ca-1338-46e1-ac15-dfa8d42f7442" />

<img width="1917" height="911" alt="Captura de pantalla 2026-10-04 000819" src="https://github.com/user-attachments/assets/fcbbe1eb-af70-4dec-8ea0-99cc7fbf7321" />

<img width="1917" height="912" alt="Captura de pantalla 2026-10-04 000835" src="https://github.com/user-attachments/assets/36b6abf5-1123-4910-b88a-11ecc2585967" />


> **Nota:** Tener todo organizado bajo un mismo Resource Group facilita mucho la limpieza al finalizar el laboratorio — basta con eliminar el grupo completo para borrar todos los recursos asociados de una sola vez.

---

## 🧱 Paso 1: Creación de las redes virtuales

Se crearon dos redes virtuales con rangos de direcciones IP distintos para evitar conflictos de direccionamiento:

- **VNET1**: `10.1.0.0/16` (subred `10.1.0.0/24`)
- **VNET2**: `10.2.0.0/16` (subred `10.2.0.0/24`)

![Creación de VNET1](images/01-crear-vnet1.png)
![Creación de VNET2](images/01-crear-vnet2.png)

---

## 🖥️ Paso 2: Despliegue de las máquinas virtuales

Se desplegó la VM `V1` dentro de VNET1 y la VM `V2` dentro de VNET2. Ninguna de las dos cuenta con IP pública, ya que el acceso remoto se gestiona íntegramente mediante Azure Bastion.

![Despliegue de V1](images/02-crear-v1.png)
![Despliegue de V2](images/02-crear-v2.png)

---

## 🔒 Paso 3: Configuración de los Network Security Groups (NSGs)

Se configuraron `NSG1` y `NSG2` para permitir el tráfico necesario entre ambas VNets, incluyendo el protocolo **ICMP** (necesario para las pruebas de ping) y **RDP** (puerto 3389) para el acceso vía Bastion.

![Reglas de NSG1](images/03-nsg1-reglas.png)
![Reglas de NSG2](images/03-nsg2-reglas.png)

---

## 🛡️ Paso 4: Creación de la subred para Azure Bastion

Antes de desplegar Bastion, se creó una subred dedicada y obligatoria llamada exactamente `AzureBastionSubnet` dentro de VNET1, con un tamaño mínimo de `/26`.

> **Nota:** Bastion requiere su propia subred exclusiva, separada de donde residen las VMs.

![Subred AzureBastionSubnet](images/04-subred-bastion.png)

---

## 🚪 Paso 5: Despliegue de Azure Bastion

Con la subred lista, se desplegó el recurso de Azure Bastion (SKU Basic) dentro de VNET1. Esto permite conectarse por RDP a las VMs directamente desde el navegador, sin exponerlas a Internet.

![Despliegue de Bastion](images/05-crear-bastion.png)
![Bastion desplegado correctamente](images/05-bastion-succeeded.png)

---

## 🔗 Paso 6: Configuración del Peering entre VNET1 y VNET2

Se estableció el peering entre ambas redes virtuales, configurándolo en ambos sentidos simultáneamente desde el formulario de Azure. El estado final debe mostrar **"Connected"** en ambos lados.

![Formulario de creación del peering](images/06-crear-peering.png)
![Peering en estado Connected](images/06-peering-connected.png)

---

## 🔥 Paso 7: Habilitación de ICMP en el firewall de Windows

Por defecto, el firewall interno de Windows Server/Windows 11 bloquea el tráfico ICMP entrante, incluso si el NSG de Azure lo permite. Fue necesario habilitarlo explícitamente **en ambas VMs**, conectado vía Bastion:

```bash
netsh advfirewall firewall add rule name="ICMPv4" protocol=icmpv4:8,any dir=in action=allow
```

![Comando ejecutado en V1](images/07-firewall-v1.png)
![Comando ejecutado en V2](images/07-firewall-v2.png)

---

## 🏓 Paso 8: Prueba de conectividad (Ping bidireccional)

Se validó la comunicación entre ambas VMs utilizando sus direcciones IP privadas:

**Desde V1 hacia V2:**
```
ping 10.2.0.4
```
![Ping exitoso V1 a V2](images/08-ping-v1-a-v2.png)

**Desde V2 hacia V1:**
```
ping 10.1.0.4
```
![Ping exitoso V2 a V1](images/08-ping-v2-a-v1.png)

✅ Resultado: **0% de pérdida de paquetes en ambas direcciones**, confirmando que el peering, los NSGs y los firewalls locales están correctamente configurados.

---

## 🌍 Paso 9: Creación del NAT Gateway

Para proporcionar salida a Internet a `V1` sin asignarle una IP pública directa, se creó un recurso **NAT Gateway**, asociado a una IP pública estándar y vinculado a la subred principal de VNET1.

![Configuración del NAT Gateway](images/09-crear-nat.png)
![NAT Gateway asociado a la subred](images/09-nat-subred.png)

---

## 🔌 Paso 10: Prueba de salida a Internet

Conectado a V1 por Bastion, se verificó el acceso a Internet exitosamente a través del NAT Gateway, navegando desde el explorador (Edge) dentro de la VM.

![Prueba de navegación en V1](images/10-prueba-internet.png)

---

## ✅ Resultados finales

| Prueba | Resultado |
|---|---|
| Peering VNET1 ↔ VNET2 | ✅ Connected (ambos sentidos) |
| Ping V1 → V2 | ✅ 0% pérdida |
| Ping V2 → V1 | ✅ 0% pérdida |
| Acceso RDP vía Bastion | ✅ Funcional para V1 y V2 |
| Salida a Internet vía NAT Gateway | ✅ Funcional |

---

## 🧠 Conceptos clave

- **VNet Peering**: conecta dos redes virtuales de Azure a nivel de red, permitiendo comunicación directa por IP privada a través de la backbone interna de Microsoft, sin pasar por Internet.
- **Azure Bastion**: servicio PaaS que permite conexión RDP/SSH segura a VMs sin exponerlas con IP pública, reduciendo la superficie de ataque.
- **NAT Gateway**: permite tráfico de salida a Internet desde una subred privada, sin necesidad de asignar IP pública a cada VM individualmente.
- **NSG (Network Security Group)**: firewall a nivel de red/subred de Azure, independiente del firewall interno del sistema operativo.

---

## 📁 Estructura del repositorio

```
├── README.md
└── images/
    ├── 00-diagrama-arquitectura.png
    ├── 00-crear-resource-group.png
    ├── 01-crear-vnet1.png
    ├── 01-crear-vnet2.png
    ├── 02-crear-v1.png
    ├── 02-crear-v2.png
    ├── 03-nsg1-reglas.png
    ├── 03-nsg2-reglas.png
    ├── 04-subred-bastion.png
    ├── 05-crear-bastion.png
    ├── 05-bastion-succeeded.png
    ├── 06-crear-peering.png
    ├── 06-peering-connected.png
    ├── 07-firewall-v1.png
    ├── 07-firewall-v2.png
    ├── 08-ping-v1-a-v2.png
    ├── 08-ping-v2-a-v1.png
    ├── 09-crear-nat.png
    ├── 09-nat-subred.png
    └── 10-prueba-internet.png
```

---

## 👤 Autor

Laboratorio realizado como parte del curso de Infraestructura de Nube.
