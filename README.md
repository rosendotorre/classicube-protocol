https://www.npmjs.com/package/classicube-protocol

classicube-protocol

Implementación simple del protocolo de red de ClassiCube / Minecraft Classic en Node.js.

Esta librería permite crear clientes, servidores, proxies o herramientas de red que utilicen el protocolo original de Minecraft Classic compatible con ClassiCube.

---

✨ Características

- Implementación del protocolo Minecraft Classic
- Compatible con ClassiCube
- Basado en Node.js
- Sin dependencias pesadas
- Fácil de integrar en proyectos existentes

Casos de uso:

- servidores Classic
- bots
- sniffers de paquetes
- proxies
- herramientas de debugging

---

📦 Instalación

npm install classicube-protocol

---

🚀 Uso básico

Ejemplo simple para conectarse a un servidor y leer paquetes.

const net = require("net")
const protocol = require("classicube-protocol")

const client = net.connect(25565, "localhost")

client.on("connect", () => {
    console.log("Conectado al servidor")
})

client.on("data", (data) => {
    const packet = protocol.parsePacket(data)
    console.log(packet)
})

---

📡 Estructura del protocolo

El protocolo Classic utiliza IDs de paquetes donde cada uno representa una acción específica.

ID| Nombre| Descripción
0x00| Identification| Login del jugador
0x01| Ping| Keep Alive
0x02| Level Initialize| Inicio del mapa
0x03| Level Data Chunk| Datos del mapa
0x04| Level Finalize| Fin de transferencia del mapa

Cada paquete comienza con:

[ Packet ID ][ Packet Data ]

---

🧠 Leer ID de paquete manualmente

Ejemplo básico usando Buffer de Node.js.

const id = buffer.readUInt8(0)

console.log("Packet ID:", id)

---

🖥 Crear un servidor simple

Ejemplo mínimo de servidor TCP.

const net = require("net")

const server = net.createServer((socket) => {

    console.log("Jugador conectado")

    socket.on("data", (data) => {
        console.log("Paquete recibido:", data)
    })

    socket.on("close", () => {
        console.log("Jugador desconectado")
    })

})

server.listen(25565, () => {
    console.log("Servidor escuchando en puerto 25565")
})

---

🧪 Casos de uso

Esta librería puede utilizarse para:

- crear servidores Minecraft Classic
- desarrollar bots automáticos
- analizar tráfico de red
- crear proxies
- depurar el protocolo

---

📂 Proyecto relacionado

Cliente moderno compatible con este protocolo:

ClassiCube

---

🤝 Contribuciones

Las contribuciones son bienvenidas.

Puedes ayudar:

- reportando bugs
- mejorando documentación
- agregando ejemplos
- optimizando el parser del protocolo

---

📜 Licencia

MIT License
