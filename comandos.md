# comandos.md - Taller práctico de subnetting en Ubuntu (Semana 12)

**Estudiante:** Esteban Alejandro Melo Borja  
**Objetivo:** Documentar las instrucciones y explicar qué cambia cada grupo de comandos.

Los comandos se ejecutaron **individualmente** en Ubuntu para construir y verificar dos topologías. Este archivo es documentación, **no es un script para ejecutar completo**. Las interfaces del Ubuntu principal y la red del equipo anfitrión no deben modificarse.

## 1. Consulta inicial de Ubuntu

```bash
ip -V
ip -br addr
ip -4 route
sudo ip netns list
```

- `ip -V` informa la versión de la herramienta `ip`.
- `ip -br addr` muestra un resumen de las interfaces y direcciones de red.
- `ip -4 route` muestra la tabla de rutas IPv4 del entorno principal.
- `ip netns list` consulta los namespaces existentes antes de crearlos.

## 2. Escenario /26 - creación de namespaces

```bash
sudo ip netns add aula-a
sudo ip netns add aula-b
sudo ip netns add aula-r
sudo ip netns list
```

Cada `ip netns add` crea un espacio de red independiente dentro del mismo sistema Linux. `aula-a` representa PC-A, `aula-b` representa PC-B y `aula-r` representa el router virtual. El último comando verifica los tres.

**Evidencia:** `evidencias/26-01-namespaces.png`.

## 3. Escenario /26 - interfaces Ethernet virtuales

```bash
sudo ip link add veth-a type veth peer name veth-ra
sudo ip link add veth-b type veth peer name veth-rb
sudo ip link set veth-a netns aula-a
sudo ip link set veth-ra netns aula-r
sudo ip link set veth-b netns aula-b
sudo ip link set veth-rb netns aula-r
sudo ip -n aula-a link
sudo ip -n aula-b link
sudo ip -n aula-r link
```

`ip link add ... type veth peer` crea dos extremos de una conexión Ethernet virtual. `ip link set ... netns` mueve cada extremo al namespace que le corresponde. `ip -n ... link` consulta las interfaces dentro de un namespace específico. El router contiene `veth-ra` y `veth-rb`.

**Evidencia:** `evidencias/26-02-interfaces.png`.

## 4. Escenario /26 - direcciones IPv4 y activación

```bash
sudo ip -n aula-a addr add 192.168.10.10/26 dev veth-a
sudo ip -n aula-r addr add 192.168.10.1/26 dev veth-ra
sudo ip -n aula-b addr add 192.168.10.70/26 dev veth-b
sudo ip -n aula-r addr add 192.168.10.65/26 dev veth-rb
sudo ip -n aula-a link set lo up
sudo ip -n aula-b link set lo up
sudo ip -n aula-r link set lo up
sudo ip -n aula-a link set veth-a up
sudo ip -n aula-b link set veth-b up
sudo ip -n aula-r link set veth-ra up
sudo ip -n aula-r link set veth-rb up
sudo ip -n aula-a -br addr
sudo ip -n aula-b -br addr
sudo ip -n aula-r -br addr
```

`addr add` asigna una dirección IPv4 y su prefijo a una interfaz. `link set ... up` activa la interfaz. Las consultas finales confirman las cuatro direcciones y que los enlaces están activos.

**Evidencia:** `evidencias/26-03-ipv4.png`.

## 5. Escenario /26 - pruebas locales y rutas

```bash
sudo ip netns exec aula-a ping -c 3 192.168.10.1
sudo ip netns exec aula-b ping -c 3 192.168.10.65
sudo ip netns exec aula-a ping -c 3 192.168.10.70
sudo ip -n aula-a route add default via 192.168.10.1
sudo ip -n aula-b route add default via 192.168.10.65
sudo ip -n aula-a route
sudo ip -n aula-b route
sudo ip -n aula-r route
```

`ping -c 3` envía tres solicitudes ICMP para comprobar conectividad. Las dos primeras pruebas son locales, desde cada computador hacia su interfaz de gateway. El ping a otro segmento no puede completarse hasta disponer de enrutamiento adecuado. `route add default via` agrega el gateway por defecto a las tablas de cada computador; `route` permite inspeccionarlas.

**Evidencias:** `26-04-rutas.png`, `26-05-ping-local.png`.

## 6. Escenario /26 - habilitar y verificar el routing

```bash
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=0
sudo ip netns exec aula-a ping -c 3 192.168.10.70
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec aula-a ping -c 3 192.168.10.70
sudo ip netns exec aula-b ping -c 3 192.168.10.10
```

`ip netns exec aula-r` ejecuta el comando dentro del router virtual. `sysctl -w net.ipv4.ip_forward=0` prohíbe el reenvío entre redes; `=1` lo permite. Con el reenvío activado se verificó la comunicación entre los dos computadores en ambas direcciones.

**Evidencia:** `evidencias/26-06-routing-exitoso.png`.

## 7. Escenario /26 - diagnóstico y recuperación

```bash
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=0
sudo ip netns exec aula-a ping -c 3 192.168.10.70
sudo ip netns exec aula-a ping -c 3 192.168.10.1
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec aula-a ping -c 3 192.168.10.70
```

Al desactivar el reenvío, el destino remoto falla, pero el gateway responde. Restablecer `ip_forward=1` recupera la comunicación entre subredes.

**Evidencia:** `evidencias/26-07-diagnostico.png`.

## 8. Cambio de escenario: limpiar /26

```bash
sudo ip netns del aula-a
sudo ip netns del aula-b
sudo ip netns del aula-r
sudo ip netns list
```

`ip netns del` elimina los namespaces del primer escenario y permite reconstruir la topología con nuevas direcciones. La eliminación se hizo para iniciar el laboratorio `/27`.

## 9. Reto 3 - reconstruir las interfaces de /27

```bash
sudo ip netns add aula-a
sudo ip netns add aula-b
sudo ip netns add aula-r
sudo ip netns list
sudo ip link add veth-a type veth peer name veth-ra
sudo ip link add veth-b type veth peer name veth-rb
sudo ip link set veth-a netns aula-a
sudo ip link set veth-ra netns aula-r
sudo ip link set veth-b netns aula-b
sudo ip link set veth-rb netns aula-r
sudo ip -n aula-a link
sudo ip -n aula-b link
sudo ip -n aula-r link
```

Se volvieron a crear los tres espacios de red y sus enlaces, conservando la estructura de la topología anterior. Los nombres de dispositivos permanecen iguales.

**Evidencias:** `27-01-namespaces.png` y `27-02-interfaces.png`.

## 10. Reto 3 - direccionamiento /27

```bash
sudo ip -n aula-a addr add 10.20.30.10/27 dev veth-a
sudo ip -n aula-r addr add 10.20.30.1/27 dev veth-ra
sudo ip -n aula-b addr add 10.20.30.40/27 dev veth-b
sudo ip -n aula-r addr add 10.20.30.33/27 dev veth-rb
sudo ip -n aula-a link set lo up
sudo ip -n aula-b link set lo up
sudo ip -n aula-r link set lo up
sudo ip -n aula-a link set veth-a up
sudo ip -n aula-b link set veth-b up
sudo ip -n aula-r link set veth-ra up
sudo ip -n aula-r link set veth-rb up
sudo ip -n aula-a -br addr
sudo ip -n aula-b -br addr
sudo ip -n aula-r -br addr
```

Se configuraron cuatro direcciones en dos redes `/27`: `10.20.30.0/27` y `10.20.30.32/27`. Cada una tiene 32 direcciones en total, de las que 30 son utilizables. La comprobación mostró las interfaces activas.

**Evidencia:** `evidencias/27-03-ipv4.png`.

## 11. Reto 3 - gateways y rutas

```bash
sudo ip -n aula-a route add default via 10.20.30.1
sudo ip -n aula-b route add default via 10.20.30.33
sudo ip -n aula-a route
sudo ip -n aula-b route
sudo ip -n aula-r route
```

PC-A envía tráfico remoto al gateway `10.20.30.1` y PC-B al gateway `10.20.30.33`. El router ve ambas redes porque están conectadas directamente mediante `veth-ra` y `veth-rb`.

**Evidencia:** `evidencias/27-04-rutas.png`.

## 12. Reto 3 - pruebas de conectividad

```bash
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=0
sudo ip netns exec aula-a ping -c 3 10.20.30.1
sudo ip netns exec aula-a ping -c 3 10.20.30.40
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec aula-a ping -c 3 10.20.30.40
sudo ip netns exec aula-b ping -c 3 10.20.30.10
```

**Resultados reales:** con `ip_forward=0`, PC-A pudo llegar a su gateway (3 de 3 respuestas), pero no a PC-B (0 de 3). Con `ip_forward=1`, las pruebas entre PC-A y PC-B funcionaron en ambos sentidos (3 de 3 respuestas en cada prueba).

**Evidencias:** `27-05-ping-fallido.png` y `27-06-routing-exitoso.png`.

## 13. Reto 4 - diagnóstico de una falla en /27

```bash
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=0
sudo ip netns exec aula-a ping -c 3 10.20.30.40
sudo ip netns exec aula-a ping -c 3 10.20.30.1
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec aula-a ping -c 3 10.20.30.40
```

- **Prueba que falla:** PC-A hacia PC-B, porque el router no reenvía IPv4.
- **Prueba que funciona:** PC-A hacia `10.20.30.1`, su puerta de enlace local.
- **Causa:** `ip_forward=0` deshabilita el tráfico entre interfaces del router, no los paquetes destinados a la interfaz local.
- **Solución:** `sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1`.

**Evidencia:** `evidencias/27-07-diagnostico.png` (reutilizada también en el Reto 4 del PDF).

## 14. Limpieza opcional al terminar la entrega

Los siguientes comandos **eliminan la topología**, por lo que deben ejecutarse **solo si se decide limpiar la máquina virtual después de guardar todas las capturas**. No es necesario repetirlos para la entrega.

```bash
sudo ip netns list
sudo ip netns del aula-a
sudo ip netns del aula-b
sudo ip netns del aula-r
sudo ip netns list
```

---

## Reto 2 - explicación específica de seis instrucciones

| Instrucción | Elemento que modifica y finalidad |
|---|---|
| `sudo ip netns add aula-a` | Crea el namespace `aula-a`, con sus propias interfaces y rutas. |
| `sudo ip link add veth-a type veth peer name veth-ra` | Crea una pareja de interfaces Ethernet virtuales conectadas entre sí. |
| `sudo ip link set veth-a netns aula-a` | Traslada `veth-a` al namespace de PC-A. |
| `sudo ip -n aula-a addr add 192.168.10.10/26 dev veth-a` | Asigna la IPv4 y el prefijo `/26` a la interfaz `veth-a`. |
| `sudo ip -n aula-a route add default via 192.168.10.1` | Define el gateway por defecto de PC-A para llegar a otras redes. |
| `sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1` | Habilita el reenvío de paquetes IPv4 en el router virtual. |

Las evidencias del laboratorio y los resultados reales se detallan en [`README.md`](README.md) y en las imágenes de la carpeta `evidencias/`.
