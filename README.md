# Semana 12 - Taller práctico de subnetting en Ubuntu

**Estudiante:** Esteban Alejandro Melo Borja  
**Asignatura:** Fundamentos de Redes  
**Grupo:** 2E 20262  
**Tema:** Diseño de subredes IPv4, Network Namespaces, interfaces `veth` y enrutamiento Linux  
**Entorno:** Ubuntu en una máquina virtual de VirtualBox

## 1. Objetivo

Diseñar y construir dos redes IPv4 en una única instalación de Ubuntu, utilizando los espacios de red `aula-a` (PC-A), `aula-b` (PC-B) y `aula-r` (router). Configurar direcciones IPv4, puertas de enlace y enrutamiento; comprobar con `ping` la conectividad local y remota; y diagnosticar la pérdida de comunicación cuando se desactiva `net.ipv4.ip_forward`.

Se desarrollaron dos escenarios: el inicial con redes `/26` y el segundo con redes `/27`.

## 2. Diseño y cálculo de subredes

### Escenario 1: prefijo /26

La máscara `/26` corresponde a `255.255.255.192`. Hay `32 - 26 = 6` bits de host, `2^6 = 64` direcciones totales por subred, **62 utilizables** y un salto de **64** direcciones.

| Subred | Red/prefijo | Hosts utilizables | Broadcast |
|---|---|---|---|
| A | `192.168.10.0/26` | `192.168.10.1` - `192.168.10.62` | `192.168.10.63` |
| B | `192.168.10.64/26` | `192.168.10.65` - `192.168.10.126` | `192.168.10.127` |

| Dispositivo | Namespace | Interfaz | Dirección IPv4 | Gateway |
|---|---|---|---|---|
| PC-A | `aula-a` | `veth-a` | `192.168.10.10/26` | `192.168.10.1` |
| Router, lado A | `aula-r` | `veth-ra` | `192.168.10.1/26` | No aplica |
| Router, lado B | `aula-r` | `veth-rb` | `192.168.10.65/26` | No aplica |
| PC-B | `aula-b` | `veth-b` | `192.168.10.70/26` | `192.168.10.65` |

### Escenario 2: prefijo /27 (Reto 3)

La máscara `/27` corresponde a `255.255.255.224`. Hay `32 - 27 = 5` bits de host, `2^5 = 32` direcciones totales por subred, **30 utilizables** y un salto de **32** direcciones.

| Subred | Red/prefijo | Hosts utilizables | Broadcast |
|---|---|---|---|
| A | `10.20.30.0/27` | `10.20.30.1` - `10.20.30.30` | `10.20.30.31` |
| B | `10.20.30.32/27` | `10.20.30.33` - `10.20.30.62` | `10.20.30.63` |

| Dispositivo | Namespace | Interfaz | Dirección IPv4 | Gateway |
|---|---|---|---|---|
| PC-A | `aula-a` | `veth-a` | `10.20.30.10/27` | `10.20.30.1` |
| Router, lado A | `aula-r` | `veth-ra` | `10.20.30.1/27` | No aplica |
| Router, lado B | `aula-r` | `veth-rb` | `10.20.30.33/27` | No aplica |
| PC-B | `aula-b` | `veth-b` | `10.20.30.40/27` | `10.20.30.33` |

## 3. Topología de red

La misma estructura de red se utilizó en los dos escenarios; únicamente cambiaron las direcciones IPv4 y las máscaras.

```text
      SUBRED A                         SUBRED B
       PC-A                             PC-B
      [aula-a]                        [aula-b]
       veth-a                          veth-b
          |                               |
          |    pareja veth      pareja veth|
          |                               |
       veth-ra                         veth-rb
             \\                       //
                [aula-r: ROUTER]
               Reenvío IPv4 (ip_forward)
```

El router tiene dos interfaces, una en cada subred. Los computadores no están unidos directamente entre sí.

## 4. Procedimiento y retos

1. **Reto 1 - Interpretación IPv4:** se calcularon redes, máscaras `/26`, direcciones de broadcast y gateways.
2. **Reto 2 - Comandos Linux:** se explicaron las operaciones de creación de namespaces, parejas `veth`, traslado de interfaces, configuración IPv4, rutas por defecto y activación de `ip_forward`. El detalle está en [`comandos.md`](comandos.md).
3. **Laboratorio inicial /26:** se crearon los namespaces `aula-a`, `aula-b` y `aula-r`, sus dos parejas `veth`, las direcciones IPv4 y los gateways. Se verificó el efecto de habilitar y deshabilitar el reenvío IPv4.
4. **Reto 3 - Reconstrucción /27:** se desmontó la topología `/26` y se reconstruyó con `10.20.30.0/27` y `10.20.30.32/27`; se repitieron la configuración y las pruebas.
5. **Reto 4 - Diagnóstico:** se desactivó `net.ipv4.ip_forward`, se comparó el `ping` remoto con el local y se restableció la conectividad activando de nuevo el reenvío.

**Nota:** los comandos de [`comandos.md`](comandos.md) documentan el proceso. No deben ejecutarse todos de una sola vez como si fueran un script.

## 5. Pruebas y resultados observados

Los resultados siguientes se documentaron mediante capturas reales de la terminal de Ubuntu.

| Escenario | Prueba | Resultado observado |
|---|---|---|
| `/26` | PC-A hacia gateway `192.168.10.1` | Correcta; 3 respuestas, 0 % de pérdida |
| `/26` | PC-B hacia gateway `192.168.10.65` | Correcta; 3 respuestas, 0 % de pérdida |
| `/26` | PC-A hacia PC-B con reenvío desactivado | Fallo de conectividad |
| `/26` | Comunicación PC-A ↔ PC-B con `ip_forward=1` | Correcta en ambas direcciones |
| `/27` | PC-A hacia gateway `10.20.30.1` | Correcta; 3 respuestas, 0 % de pérdida |
| `/27` | PC-A hacia PC-B con `ip_forward=0` | Fallida; 3 paquetes enviados, 0 recibidos, 100 % de pérdida |
| `/27` | PC-A hacia PC-B con `ip_forward=1` | Correcta; 3 respuestas, 0 % de pérdida |
| `/27` | PC-B hacia PC-A con `ip_forward=1` | Correcta; 3 respuestas, 0 % de pérdida |
| `/27` | Falla provocada y restauración del reenvío | Se observa falla remota, ping local exitoso y recuperación posterior |

### Pruebas principales de /27

```bash
# Ping local (PC-A -> router)
sudo ip netns exec aula-a ping -c 3 10.20.30.1

# Ping remoto (PC-A -> PC-B)
sudo ip netns exec aula-a ping -c 3 10.20.30.40

# Ping remoto de regreso (PC-B -> PC-A)
sudo ip netns exec aula-b ping -c 3 10.20.30.10
```

## 6. Diagnóstico de una falla (Reto 4)

**1. ¿Qué deja de funcionar?** El `ping` entre PC-A (`10.20.30.10`) y PC-B (`10.20.30.40`) falla con `ip_forward=0` porque los equipos pertenecen a redes diferentes y el router no reenvía sus paquetes. En la prueba /27 se observó 100 % de pérdida.

**2. ¿Qué sigue funcionando?** El `ping` desde PC-A hacia su gateway local (`10.20.30.1`) funciona, porque el paquete está dirigido a una interfaz del propio router. En la prueba se observaron tres respuestas y 0 % de pérdida.

**3. ¿Por qué sucede?** `net.ipv4.ip_forward=0` desactiva el reenvío de paquetes IPv4 entre las interfaces del router virtual, pero no impide que el router responda a paquetes destinados a sus propias direcciones IP.

**4. ¿Qué comando restablece el funcionamiento?**

```bash
sudo ip netns exec aula-r sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec aula-a ping -c 3 10.20.30.40
```

Se reactivó el reenvío y se verificó nuevamente el destino remoto. La captura de **Reto 3, Evidencia 07** también sirve como evidencia del **Reto 4**.

## 7. Evidencias

Las imágenes se extrajeron de las capturas **reales incluidas en el informe PDF**. Sus nombres indican el escenario y el orden original; no se modificaron los resultados de terminal.

| Escenario /26 | Verificación |
|---|---|
| [`26-01-namespaces.png`](evidencias/26-01-namespaces.png) | Creación de namespaces |
| [`26-02-interfaces.png`](evidencias/26-02-interfaces.png) | Parejas veth y asignación |
| [`26-03-ipv4.png`](evidencias/26-03-ipv4.png) | Direccionamiento IPv4 |
| [`26-04-rutas.png`](evidencias/26-04-rutas.png) | Rutas y gateways |
| [`26-05-ping-local.png`](evidencias/26-05-ping-local.png) | Conectividad hacia gateways |
| [`26-06-routing-exitoso.png`](evidencias/26-06-routing-exitoso.png) | Comunicación entre subredes |
| [`26-07-diagnostico.png`](evidencias/26-07-diagnostico.png) | Falla y recuperación |

| Escenario /27 | Verificación |
|---|---|
| [`27-01-namespaces.png`](evidencias/27-01-namespaces.png) | Reconstrucción de namespaces |
| [`27-02-interfaces.png`](evidencias/27-02-interfaces.png) | Parejas veth y asignación |
| [`27-03-ipv4.png`](evidencias/27-03-ipv4.png) | Direccionamiento IPv4 /27 |
| [`27-04-rutas.png`](evidencias/27-04-rutas.png) | Rutas y gateways |
| [`27-05-ping-fallido.png`](evidencias/27-05-ping-fallido.png) | Fallo remoto con `ip_forward=0` |
| [`27-06-routing-exitoso.png`](evidencias/27-06-routing-exitoso.png) | Comunicación bidireccional |
| [`27-07-diagnostico.png`](evidencias/27-07-diagnostico.png) | Diagnóstico y recuperación (también Reto 4) |

## 8. Conclusiones

- Las máscaras `/26` y `/27` determinan el tamaño de cada subred, los rangos utilizables y el broadcast.
- Los *network namespaces* permiten simular varias máquinas sin crear varias instalaciones de Ubuntu.
- Las parejas `veth` conectan cada computador con el router virtual.
- Una ruta por defecto permite al computador enviar tráfico fuera de su subred a su gateway.
- Que un equipo alcance su gateway no demuestra que pueda alcanzar otra red: para ello se necesita el reenvío IPv4 y una ruta de retorno adecuada.
- La comparación entre un ping local exitoso y uno remoto fallido permitió identificar `ip_forward=0` como causa de la falla y verificar la recuperación al activar `ip_forward=1`.

## 9. Archivos de entrega

- `README.md`: descripción, cálculos, resultados, diagnóstico y conclusiones.
- `comandos.md`: comandos utilizados y explicación de cada grupo.
- `evidencias/`: capturas reales de ambos escenarios.
- **Informe PDF:** versión final del documento elaborado por el estudiante, con los cuatro retos y sus evidencias (añadir al repositorio al momento de subir la entrega).

**Entrega:** publicar el repositorio en GitHub y enviar su enlace mediante EVA.
