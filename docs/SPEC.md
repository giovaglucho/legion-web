Documento técnico maestro: juego web estilo Legion TD 2 (MERN)
Versión 1.0 · 12 de octubre de 2026 · Nombre provisorio del juego: Legión Web
0. Cómo usar este documento
Este documento es el contrato de construcción del juego. Está escrito para seguirse en orden, fase por fase, por una persona o por un asistente de programación que lo lea completo antes de empezar.
Secciones 1 a 10: definen QUÉ se construye (reglas, arquitectura, datos, protocolos). Son la fuente de verdad. Si el plan por fases las contradice, mandan estas secciones.
Fases 0 a 13: definen CÓMO y EN QUÉ ORDEN. Cada fase tiene objetivo, tareas, archivos, criterios de aceptación (CA) y pruebas. No se avanza hasta cumplir todos los CA de la fase actual.
Números del juego (costos, vidas, daños): son valores iniciales guardados en archivos de datos, no en el código. Se ajustan en la Fase 11 con simulaciones.
Convención: DEBE = obligatorio · DEBERÍA = recomendado · PUEDE = opcional.
Las decisiones abiertas están en la última sección, cada una con un valor por defecto. Si no se cambian, se usa el valor por defecto.
1. Visión, objetivos y alcance
1.1 Visión
Un juego de estrategia 1 contra 1 que se juega en el navegador, inspirado en Legion TD 2. Cada jugador construye una defensa en su carril, resiste oleadas automáticas y decide cuánto invertir en defensa y cuánto en enviar mercenarios al rival para ganar más ingresos. No hay control en tiempo real: se decide en la fase de construcción y el combate se resuelve solo. Las cuentas, el ranking Elo, las temporadas y los replays dan motivos para volver.
1.2 Objetivos medibles
O1. Un usuario nuevo se registra y juega su primera partida (contra bot) en menos de 3 minutos.
O2. Una partida completa de 20 rondas dura entre 20 y 35 minutos.
O3. Partidas justas: el servidor es la autoridad; ningún cliente puede alterar oro, vida ni resultados.
O4. Ranking Elo persistente con leaderboard, divisiones, temporadas, perfil, historial y replays.
O5. Una sola instancia pequeña soporta al menos 200 partidas simultáneas (se mide en la Fase 11).
O6. Reconexión: si un jugador se desconecta, vuelve a su partida en curso sin perder el estado.
1.3 Dentro del MVP (DEBE)
Registro, login, sesión persistente y cierre de sesión.
Partida 1v1 online completa (20 rondas) con las reglas de la sección 2.
Modo práctica contra bot (sin Elo) y bot de relleno cuando no hay rival en cola.
Matchmaking por rating, reconexión y manejo de abandono.
Elo, divisiones, leaderboard, perfil, historial de partidas, replays y temporadas.
Tutorial mínimo, interfaz completa y responsive (escritorio primero).
Seguridad básica, tests automáticos, despliegue y monitoreo.
1.4 Fuera del MVP (backlog)
Modos 2v2 y 4v4, mejoras de guerreros, chat, amigos, espectadores en vivo, cosméticos o pagos, torneos, app nativa.
1.5 Principios de diseño
Servidor autoritativo: el cliente solo envía intenciones; el servidor valida todo.
Simulación pura y determinista, compartida: el mismo código de combate corre en servidor y cliente.
Datos antes que código: unidades, oleadas y economía viven en archivos de datos versionados.
Primero jugable offline, luego online: el juego se prueba contra un bot antes de agregar red.
Cada fase termina en algo que se puede ejecutar y probar.
2. Diseño del juego (reglas del MVP)
2.1 Resumen de una partida
Dos jugadores. Cada uno tiene un carril propio (grilla de 10 por 18 celdas), un rey con 100 de vida y oro. La partida dura 20 rondas. Cada ronda tiene una fase de construcción y una de combate.
En construcción se compran y venden guerreros en la grilla y se envían mercenarios al rival.
En combate atacan tu carril la oleada del juego y los mercenarios que envió el rival. Tus guerreros pelean solos.
Todo enemigo que atraviesa la defensa y llega al rey le quita vida (fuga).
Gana quien deja al rival sin vida o, tras la ronda 20, quien conserva más vida.
Enviar mercenarios cuesta oro pero sube tu ingreso por ronda. La decisión central es defender más o invertir en economía y presión.
2.2 Tablero y constantes
Una celda mide 1 unidad de mundo. El centro de la celda (c, r) es (c + 0.5, r + 0.5).
Constante	Valor
Ancho del carril	10 celdas (x de 0 a 10)
Largo del carril	18 celdas (y de 0 a 18)
Zona de construcción	filas 8 a 17 (100 celdas, un guerrero por celda)
Punto de aparición enemigo	y = 0
Rey	y = 18; un enemigo que llega a y ≥ 18 es una fuga
Frecuencia de simulación	20 ticks por segundo (dt = 0.05 s)
Duración máxima de combate	1800 ticks (90 s)
Límite de guerreros por jugador	40
2.3 Fases y tiempos
Fase de la ronda	Duración	Qué pasa
Construcción	60 s en la ronda 1, 40 s desde la ronda 2	Se cobra el ingreso, se compran y venden guerreros, se envían mercenarios. Si ambos pulsan Listo, termina antes.
Combate	hasta 90 s	Cada carril se simula de forma independiente. Termina cuando no quedan enemigos vivos o a los 90 s.
Resumen	5 s	Se aplica el daño a los reyes y se muestra el marcador.
Los temporizadores son del servidor. El servidor da 1.5 s de gracia por latencia a las acciones que llegan justo al cierre.
2.4 Economía
Oro inicial: 300.
Ingreso base: 25 por ronda desde la ronda 2, más el ingreso que aportan los mercenarios enviados. Se paga al comenzar cada fase de construcción.
Vender un guerrero devuelve el 100% si se compró en esa misma ronda y el 70% si se compró antes.
Los mercenarios no se venden. Su ingreso extra es permanente para quien los envía.
No hay interés ni botín por matar enemigos.
Máximo de envíos de mercenarios por jugador y por ronda: 12.
2.5 Tipos de ataque y armadura
Tres tipos de ataque contra tres tipos de armadura. Cada armadura tiene una debilidad, un punto neutro y una resistencia.
Ataque \ Armadura	Ligera	Media	Pesada
Físico	1.00	0.75	1.25
Perforante	1.25	1.00	0.75
Mágico	0.75	1.25	1.00
Fórmula de daño: daño final = daño × multiplicador del tipo × 100 / (100 + 6 × valor de armadura). El daño final mínimo es 1.
2.6 Guerreros (unidades defensoras)
id	Nombre	Costo	Vida	Daño	Ataques/s	Rango	Ataque	Armadura (valor)	Velocidad
guardian	Guardián	50	450	14	1.0	1	Físico	Pesada (4)	2.5
arquero	Arquero	60	180	20	1.2	6	Perforante	Ligera (0)	2.5
lancero	Lancero	55	280	15	1.1	2	Perforante	Media (2)	2.5
mago	Mago	80	170	34	0.8	5	Mágico	Ligera (0)	2.5
caballero	Caballero	90	600	26	0.9	1	Físico	Media (3)	3.0
sanador	Sanador	70	160	cura 18	1.0	5	Mágico	Ligera (0)	2.5
ballesta	Ballesta	120	220	70	0.5	8	Perforante	Media (1)	2.0
golem	Gólem	150	1300	22	0.7	1	Físico	Pesada (6)	2.0
2.7 Mercenarios (se envían al rival)
Cada envío genera una cantidad de unidades en el carril del rival durante el combate de esa misma ronda.
id	Nombre	Costo	Unidades	Ingreso extra	Vida	Daño	Ataques/s	Rango	Ataque	Armadura	Velocidad	Fuga c/u
rata	Manada de ratas	20	4	+2	90	8	1.0	1	Físico	Ligera (0)	2.4	1
trasgo	Trasgos saqueadores	40	2	+4	260	14	1.0	1	Perforante	Media (2)	2.2	2
ogro	Ogro	80	1	+7	900	30	0.8	1	Físico	Pesada (4)	1.8	6
dragon	Dragón joven	160	1	+14	1800	40	0.8	4	Mágico	Media (3)	2.0	12
2.8 Criaturas de oleada
id	Nombre	Vida base	Daño base	Ataques/s	Rango	Ataque	Armadura	Velocidad	Fuga
esbirro	Esbirro	100	10	1.0	1	Físico	Ligera (0)	2.0	1
goblin_arquero	Goblin arquero	80	12	1.0	5	Perforante	Ligera (0)	2.0	1
bestia	Bestia	220	16	0.9	1	Físico	Media (2)	2.4	1
hechicero	Hechicero oscuro	140	20	0.8	5	Mágico	Media (1)	1.8	1
coloso	Coloso	600	28	0.7	1	Físico	Pesada (5)	1.5	2
troll_jefe	Troll jefe	800	40	0.8	1	Físico	Pesada (6)	1.6	10
hechicero_jefe	Hechicero jefe	1100	60	0.8	6	Mágico	Media (3)	1.6	10
golem_jefe	Gólem jefe	1800	70	0.6	1	Físico	Pesada (8)	1.4	15
dragon_jefe	Dragón jefe	2500	110	0.7	5	Mágico	Media (5)	1.8	25
Escalado por ronda n: vida × 1.14^(n−1) y daño × 1.08^(n−1) para las criaturas de oleada. Los mercenarios no escalan.
Oleadas (20):
esbirro × 6
esbirro × 8
goblin_arquero × 8
bestia × 6
troll_jefe × 1 + esbirro × 4
hechicero × 8
bestia × 10
coloso × 4
goblin_arquero × 12
hechicero_jefe × 1 + hechicero × 6
esbirro × 18
bestia × 12
coloso × 6
hechicero × 12
golem_jefe × 1 + coloso × 3
goblin_arquero × 16
bestia × 16
coloso × 8
hechicero × 16
dragon_jefe × 1 + coloso × 6
2.9 Reglas de combate
Aparición: primero entran las criaturas de la oleada y luego los mercenarios del rival, una unidad cada 6 ticks (0.3 s) desde y = 0. La columna es 0.5 + (índice mod 10).
Objetivo de un guerrero: el enemigo vivo más cercano dentro de su radio de aggro (rango + 2). Empate: menor uid.
Objetivo de un enemigo: el guerrero vivo más cercano dentro de su aggro (rango + 1.5). Si no hay, avanza recto hacia el rey (aumenta y).
Movimiento: si el objetivo está fuera de rango, la unidad se mueve hacia él a su velocidad; si está en rango, se detiene y ataca. Un guerrero no se aleja más de 4 celdas de su celda base; sin objetivo, vuelve a ella.
Ataque: cooldown = redondeo(20 / ataques por segundo) ticks. El daño es instantáneo (los proyectiles son solo visuales). El daño de un tick se acumula y se aplica al final del tick, así el orden de procesamiento no altera el resultado.
Sanador: no daña. En cada ataque cura 18 de vida al aliado con menor porcentaje de vida dentro de su rango (sin pasar de su vida máxima).
Sin colisiones: las unidades no bloquean el paso ni se empujan. Pueden superponerse visualmente.
Fuga: el enemigo que llega a y ≥ 18 desaparece y resta su valor de fuga al rey.
Fin del combate: no quedan enemigos vivos, o se cumplen 1800 ticks (los enemigos vivos restantes cuentan como fugas).
2.10 Victoria, empate, rendición y desconexión
Pierde el jugador cuyo rey llega a 0 de vida. Si ambos llegan a 0 en la misma ronda, gana el que tenga la vida resultante menos negativa; si es igual, empate.
Tras la ronda 20, gana quien tenga más vida de rey; si es igual, empate (puntaje Elo 0.5).
Rendición: disponible en cualquier momento; cuenta como derrota.
Desconexión: hay 60 s de gracia para volver. Si no vuelve, pierde por abandono. Durante la gracia, la partida se pausa solo si ambos están desconectados; si no, el reloj sigue y el desconectado no puede actuar.
Un abandono en las primeras 2 rondas cuenta igual como derrota con Elo normal, para que irse no sea una forma de evitar perder rating.
3. Arquitectura general
3.1 Vista de conjunto
```
\[ Navegador: React + PixiJS ]  --HTTPS REST-->  \[ API Express ]
        |                                            |
        +--- Socket.IO (WebSocket) -->  \[ Servidor de partidas ]  (mismo proceso Node)
                                                     |
                                              \[ MongoDB ]

packages/shared (TypeScript): motor de simulación, reglas, datos, tipos y esquemas
```
3.2 Decisiones técnicas
Tema	Decisión	Motivo
Lenguaje	TypeScript estricto en todo el repo	Los tipos compartidos evitan desacuerdos entre cliente y servidor
Repositorio	Monorepo con npm workspaces: apps/client, apps/server, packages/shared	El motor se comparte sin publicar paquetes
Cliente	React + Vite; PixiJS solo para dibujar el tablero; Zustand para estado	La interfaz es React; el tablero necesita un canvas rápido
Servidor	Node LTS + Express + Socket.IO	Estándar MERN con tiempo real
Base de datos	MongoDB + Mongoose	Es la M de MERN; los documentos encajan con partidas y replays
Validación	Zod en cada entrada (REST y sockets)	Nunca confiar en datos del cliente
Autenticación	JWT de acceso corto + refresh token en cookie httpOnly; argon2id	Seguro y compatible con sockets
Pruebas	Vitest, Supertest, mongodb-memory-server, Playwright	Cubren motor, API, sockets y flujo completo
Calidad	ESLint + Prettier + GitHub Actions	Todo cambio pasa lint, tipos y tests
Logs	pino	Logs estructurados y baratos
Despliegue	Cliente estático; servidor en un servicio con WebSockets persistentes; MongoDB Atlas	Las funciones serverless no sostienen sockets
Las versiones exactas se fijan al instalar (Fase 0) y se registran en package.json. Se usan las versiones LTS o estables vigentes en ese momento.
3.3 Flujo de una ronda (modelo de re-simulación determinista)
El servidor inicia la construcción: cobra ingresos y emite `phase:build:start` con el estado.
Cada cliente envía acciones (comprar, vender, enviar, listo). El servidor las valida con las funciones puras de shared y responde con el estado actualizado.
Al cerrar la construcción (tiempo agotado o ambos listos), el servidor congela el estado, genera una semilla y arma un `LaneInput` por carril. El carril de cada jugador incluye los mercenarios enviados por el rival.
El servidor ejecuta `simulateLane` para ambos carriles de inmediato. Ese es el resultado oficial: daño al rey, fugas, duración y huella (hash).
El servidor emite `phase:combat:start` con los `LaneInput` y la huella oficial. No envía posiciones. Cada cliente ejecuta la misma simulación localmente para animar y compara su huella con la oficial.
El servidor espera la duración simulada más 2 s, aplica el daño oficial a los reyes, emite `round:end`, comprueba victoria y pasa a la ronda siguiente.
Ventajas: tráfico mínimo, replays gratis (se guardan solo las entradas), sin sincronizar posiciones, difícil de hacer trampa. Riesgo: que un cliente se desincronice. Mitigación: la huella detecta la diferencia y, si no coincide, el cliente muestra el resultado oficial sin animación exacta. El servidor manda siempre.
4. Motor de simulación (packages/shared)
4.1 API pública
```ts
export type LaneInput = {
  dataVersion: string;
  seed: number;                 // entero de 32 bits
  round: number;                // 1..20
  fighters: { uid: number; typeId: string; x: number; y: number }\[];
  wave: { typeId: string; count: number }\[];
  mercs: { mercId: string }\[];  // enviados por el rival en esta ronda
};
export type LaneResult = {
  ticks: number;                // duración simulada
  kingDamage: number;           // suma de fugas
  leaks: { tick: number; typeId: string; damage: number }\[];
  survivors: number\[];          // uids de guerreros vivos
  hash: string;                 // huella del estado final
};
export function simulateLane(input: LaneInput, onFrame?: (frame: Frame) => void): LaneResult;
```
`onFrame` permite al cliente dibujar cada tick. El servidor no lo usa.
4.2 Orden de cada tick
Aparición de unidades pendientes.
Elección de objetivos.
Movimiento.
Ataques (se acumulan daños y curas).
Aplicación de daños y curas; eliminación de muertos.
Fugas.
Comprobación de fin de combate.
4.3 Reglas de determinismo (DEBE)
Prohibido usar Math.random, Date.now, performance.now, Math.sin, Math.cos, Math.pow, Math.exp y el orden de claves de objetos. Se usan arrays ordenados por uid.
Solo se usan sumas, restas, multiplicaciones, divisiones, Math.sqrt, Math.floor, Math.ceil, Math.round y comparaciones. Estas operaciones dan el mismo resultado en cualquier motor JavaScript (IEEE 754).
Las potencias del escalado (1.14^(n−1), 1.08^(n−1)) se precalculan con multiplicaciones repetidas y se guardan en una tabla de 20 valores.
Las distancias se comparan al cuadrado siempre que sea posible.
Se incluye un generador propio (mulberry32) alimentado con la semilla. El MVP casi no lo usa, pero la API lo reserva para variaciones futuras.
La huella es FNV-1a sobre (tick final, uid, vida × 100 redondeada, x × 1000 redondeada, y × 1000 redondeada) de todas las entidades vivas, ordenadas por uid.
Test dorado: un `LaneInput` fijo produce una huella fija guardada en el repo. Si cambia sin cambiar `dataVersion`, CI falla.
4.4 Rendimiento
Meta: simular una ronda completa (hasta 150 entidades y 1800 ticks) en menos de 100 ms en Node. Si se supera, se agrega una grilla espacial para buscar objetivos.
5. Reglas de partida y máquina de estados (packages/shared)
5.1 Tipos principales
```ts
type PlacedFighter = { uid: number; typeId: string; x: number; y: number; boughtRound: number };
type PlayerState = {
  userId: string;
  kingHp: number;
  gold: number;
  income: number;               // ingreso por ronda actual
  fighters: PlacedFighter\[];
  pendingSends: { mercId: string }\[];
  ready: boolean;
};
type MatchState = {
  dataVersion: string;
  seed: number;
  round: number;                // 1..20
  phase: 'build' | 'combat' | 'summary' | 'ended';
  players: \[PlayerState, PlayerState];
  phaseEndsAt: number;          // milisegundos epoch del servidor
  winner?: 0 | 1 | 'draw';
};
type Action =
  | { t: 'place'; typeId: string; x: number; y: number }
  | { t: 'sell'; uid: number }
  | { t: 'send'; mercId: string }
  | { t: 'ready' };
```
5.2 Funciones puras (sin red, sin base de datos, sin reloj)
Función	Qué hace
createMatch(userIds, seed)	Crea el estado inicial: ronda 1, 300 de oro, reyes con 100
applyAction(state, playerIdx, action)	Valida y aplica una acción en construcción; devuelve el nuevo estado o un código de error
startBuildPhase(state)	Cobra ingresos (desde la ronda 2) y reinicia los listos
buildLaneInputs(state)	Arma los dos `LaneInput` de la ronda, cruzando los mercenarios
applyRoundResult(state, results)	Resta las fugas a los reyes y limpia los envíos pendientes
checkVictory(state)	Devuelve ganador, empate o ninguno según 2.10
5.3 Códigos de error de validación
`NOT\_IN\_BUILD\_PHASE`, `ALREADY\_READY`, `UNKNOWN\_TYPE`, `OUT\_OF\_ZONE`, `CELL\_OCCUPIED`, `NOT\_ENOUGH\_GOLD`, `SUPPLY\_LIMIT`, `UNIT\_NOT\_FOUND`, `TOO\_MANY\_SENDS`.
Regla de oro: todas las funciones son puras y se prueban sin servidor. El servidor y el bot usan exactamente las mismas funciones.
6. Protocolo de red (REST y Socket.IO)
6.1 API REST (prefijo /api)
Método	Ruta	Auth	Descripción
POST	/api/auth/register	no	Body: username, email, password. Crea la cuenta, devuelve access token y fija la cookie de refresh
POST	/api/auth/login	no	Body: emailOrUsername, password
POST	/api/auth/refresh	cookie	Rota el refresh token y devuelve un access token nuevo
POST	/api/auth/logout	cookie	Revoca el refresh token
GET	/api/me	sí	Perfil propio, rating y partida activa si existe
PATCH	/api/me	sí	Preferencias (sonido, idioma)
GET	/api/leaderboard	no	Parámetros: season, page, limit. Ranking paginado
GET	/api/users/:username	no	Perfil público y estadísticas
GET	/api/users/:username/matches	no	Historial paginado
GET	/api/matches/:id	no	Datos de la partida y entradas para replay
GET	/api/game-data	no	Datos del juego con dataVersion (el cliente los cachea)
GET	/api/health	no	Estado del servidor y de la base de datos
Formato de error: `{ error: { code, message, details? } }` con el código HTTP correcto (400 validación, 401 sin sesión, 403 prohibido, 404, 409 duplicado, 429 límite, 500).
6.2 Socket.IO
El handshake envía `auth: { token: accessToken }`. El servidor rechaza la conexión si el token es inválido o expiró. Cada partida usa la sala `match:<id>`.
Cliente → servidor
Evento	Payload	Respuesta
queue:join	mode: ranked o practice	ack con ok o error
queue:leave	vacío	ack
match:join	matchId (tras match:found o al reconectar)	el servidor envía match:state
action	matchId, seq, action (tipo Action)	ack con ok, estado nuevo o código de error
match:surrender	matchId	ack
Servidor → cliente
Evento	Payload
queue:status	segundos de espera y ventana de rating actual
match:found	matchId, oponente (username, rating)
match:state	vista completa del estado (filtrada) y serverTime
state:update	estado nuevo tras una acción propia o del rival
phase:build:start	round, endsAt, estado
phase:combat:start	round, los dos LaneInput, resultados oficiales (ticks, kingDamage, hash)
round:end	round, vida de ambos reyes
match:end	winner, cambios de rating (antes, después, delta), matchId
opponent:status	connected y, si no, graceEndsAt
error	code, message
Información visible: cada jugador ve la grilla del rival durante la construcción. Los envíos pendientes de mercenarios del rival permanecen ocultos hasta el inicio del combate. La vista de estado se filtra en el servidor.
Reglas de red (DEBE):
Cada acción lleva un número `seq` creciente. El servidor ignora repetidos y responde con el resultado anterior (idempotencia ante reintentos).
Límite de 20 acciones por segundo por jugador. Si se supera de forma sostenida, se desconecta al socket.
El reloj lo manda el servidor. El cliente calcula el desfase con serverTime solo para mostrar la cuenta regresiva; nunca decide nada con él.
7. Modelo de datos (MongoDB)
7.1 Colecciones
users
username (3 a 16 caracteres: letras, números y guion bajo), usernameLower (único), email en minúsculas (único), passwordHash.
createdAt, lastLoginAt, role (player o admin), banned (booleano), settings (sound, language).
rating: elo (inicia en 1000), peak, games, wins, losses, draws, streak (racha actual), provisionalGames.
refreshTokens
userId, tokenHash, familyId, expiresAt (índice TTL), revokedAt, userAgent, createdAt.
matches
mode (ranked, practice, bot), season, seed, dataVersion, startedAt, endedAt.
players: dos entradas con userId (nulo si es bot), username, eloBefore, eloAfter, finalKingHp, result (win, loss, draw).
rounds: lista con n, inputs (los dos LaneInput), results (resultados oficiales), kingHpAfter y las acciones de cada jugador con su marca de tiempo.
winner, endReason (kingDead, completed, surrender, abandon), ratingApplied (booleano).
seasons
number (único), startsAt, endsAt, status (active o closed).
seasonRatings
seasonNumber, userId, elo, peak, games, wins, losses, division, finalRank. Índice único (seasonNumber, userId).
7.2 Índices
users: usernameLower único, email único, `{ 'rating.elo': -1 }` para el leaderboard.
matches: `{ 'players.userId': 1, endedAt: -1 }` para historial y `{ season: 1, endedAt: -1 }`.
refreshTokens: índice TTL sobre expiresAt y índice sobre familyId.
7.3 Tamaño y retención
Una partida con replay pesa entre 60 y 100 KB, muy por debajo del límite de 16 MB por documento. Se guarda todo en el MVP. En la Fase 12 se agrega una tarea que compacta (borra los inputs) de las partidas con más de 90 días.
8. Autenticación y seguridad
8.1 Cuentas y sesiones
Contraseña de 8 caracteres o más, almacenada con argon2id. Nunca se registra ni se devuelve.
Access token: JWT de 15 minutos con sub y role.
Refresh token: valor aleatorio de 256 bits; en la base solo se guarda su hash. Dura 30 días y se rota en cada uso. Si se presenta uno ya usado, se revoca toda su familia y el usuario debe volver a entrar.
Cookie de refresh: httpOnly, Secure, SameSite=Lax (Strict si cliente y API comparten dominio), ruta /api/auth.
Sockets: el token se valida en el handshake. Si expira, el cliente refresca por REST, actualiza `socket.auth` y reconecta; el servidor devuelve al jugador a su partida.
8.2 Controles obligatorios
Control	Valor inicial
Límite de login	10 intentos cada 15 minutos por IP y usuario
Límite de registro	5 cuentas por hora por IP
Límite REST general	100 solicitudes por minuto por IP
Límite de acciones por socket	20 por segundo por jugador
Tamaño máximo de body	100 KB
Cabeceras	helmet activado
CORS	lista blanca desde CLIENT_ORIGIN, sin comodín cuando hay credenciales
Validación	Zod en toda entrada; se rechazan claves que empiecen con $ o contengan punto
Secretos	Solo en variables de entorno; .env nunca se sube al repositorio
8.3 Anti-abuso del ranking
Un mismo par de jugadores que juega más de 3 partidas ranked en 24 horas recibe solo el 50% del cambio de Elo desde la cuarta.
Entrada al leaderboard desde la partida ranked número 5.
Elo mínimo de 100.
Las acciones rechazadas por validación se cuentan y se registran para detectar clientes modificados.
Detección de cuentas múltiples y sistema de reportes quedan en el backlog.
8.4 Privacidad
No se guarda la IP más de 30 días. Los logs nunca incluyen contraseñas ni tokens. El email no es público. Antes de abrir al público se publican términos de uso y política de privacidad. Verificación de email y recuperación de contraseña se implementan en la Fase 10 (requieren un proveedor de correo; ver decisiones abiertas).
9. Matchmaking, Elo, divisiones y temporadas
9.1 Elo
Rating inicial: 1000.
Esperado: E = 1 / (1 + 10^((R_rival − R_propio) / 400)).
Nuevo rating: R' = R + K × (S − E), con S = 1 (victoria), 0.5 (empate) o 0 (derrota). Se redondea a entero.
K = 40 durante las primeras 10 partidas ranked del jugador y K = 24 después. Como los K pueden diferir, el cambio no es exactamente simétrico; es intencional para que los nuevos converjan rápido.
Ejemplo: 1000 contra 1000 con K = 40 da +20 al ganador y −20 al perdedor.
Aplicación única: al terminar una partida ranked, el servidor marca `ratingApplied` con una actualización atómica condicionada (solo si estaba en falso). Así el Elo se aplica una sola vez aunque haya reintentos o reinicios. No requiere transacciones.
9.2 Modos y bots
Ranked: solo contra humanos; cambia el Elo.
Práctica: contra bot; no cambia el Elo. Sirve de tutorial y de relleno cuando hay poca gente.
Si en ranked no aparece rival en 25 segundos, se ofrece al jugador jugar contra un bot sin Elo o seguir esperando. Nunca se asigna un bot en una partida ranked.
9.3 Cola de matchmaking
Cola en memoria para ranked. Cada segundo se ordena por rating y se empareja.
Ventana de búsqueda = min(100 + 20 × segundos de espera, 600). Dos jugadores se emparejan si la diferencia de rating cabe en la ventana de ambos.
Si dos jugadores jugaron entre sí hace menos de 10 minutos, se prefiere otro rival durante 30 segundos.
Si el servidor se reinicia, la cola se pierde; el cliente detecta la reconexión y vuelve a encolarse.
9.4 Divisiones
División	Rango de Elo
Bronce	menos de 900
Plata	900 a 1099
Oro	1100 a 1299
Platino	1300 a 1499
Diamante	1500 a 1699
Maestro	1700 o más
9.5 Temporadas
Duración inicial: 8 semanas, configurable con SEASON_LENGTH_DAYS.
Al cerrar: se guarda un `seasonRating` por jugador con división y posición final, y se hace un reinicio suave: nuevo Elo = 1000 + (Elo − 1000) × 0.5, con 5 partidas provisionales.
Una tarea programada idempotente revisa cada hora si la temporada venció (y también al arrancar el servidor).
Los jugadores reciben una insignia de perfil según su división final.
9.6 Motivos para volver (incluidos en el MVP)
Racha de victorias, mejor Elo histórico, insignias de temporada, estadísticas personales (unidad más usada, porcentaje de victorias), posición en el leaderboard con su variación semanal y replays compartibles por URL.
10. Estructura del repositorio y convenciones
```
legion-web/
  package.json              (workspaces)
  tsconfig.base.json
  docker-compose.yml        (MongoDB para desarrollo)
  .env.example
  .github/workflows/ci.yml
  packages/shared/
    src/data/               fighters.json, mercs.json, creeps.json, waves.json, rules.json
    src/schemas/            esquemas Zod y tipos
    src/sim/                simulateLane, rng, hash, targeting
    src/rules/              createMatch, applyAction, startBuildPhase, ...
    src/bots/               bot heurístico
    src/elo/                cálculo de Elo y divisiones
    test/
  apps/server/
    src/config/  src/db/  src/models/  src/auth/  src/routes/
    src/sockets/  src/match/ (MatchRoom, MatchManager)
    src/matchmaking/  src/jobs/  src/middleware/
    src/app.ts  src/index.ts
    test/
  apps/client/
    src/api/  src/net/  src/store/  src/game/ (renderer del tablero)
    src/pages/  src/components/  src/styles/
    e2e/
```
Convenciones
Ninguna regla ni número del juego fuera de packages/shared. Ningún número mágico: todo viene de src/data.
Prohibido `any`. TypeScript estricto.
Una rama por fase (`fase-3-reglas`), commits convencionales (`feat:`, `fix:`, `test:`), CI en verde obligatorio para unir a main.
Cada función pura de shared lleva tests.
Variables de entorno (.env.example)
Variable	Uso
NODE_ENV	development, test o production
PORT	Puerto de la API
MONGODB_URI	Cadena de conexión
JWT_ACCESS_SECRET	Secreto largo y aleatorio para firmar access tokens
CLIENT_ORIGIN	Origen permitido por CORS
LOG_LEVEL	info, debug, etc.
SEASON_LENGTH_DAYS	Duración de la temporada (56 por defecto)
PARTE III. PLAN POR FASES
Cada fase indica objetivo, tareas, criterios de aceptación (CA) y pruebas. El orden está pensado para tener algo jugable lo antes posible y reducir riesgos: primero el corazón (motor y reglas), luego el cliente, luego la red.
Fase 0. Entorno y repositorio
Objetivo: un repositorio con monorepo, lint, tipos, tests y CI funcionando, sin nada del juego todavía.
Tareas
Instalar Node.js LTS, Git, un editor y MongoDB para desarrollo (docker-compose o una cuenta gratuita de MongoDB Atlas).
Crear el repositorio con la estructura de la sección 10 y npm workspaces.
Configurar tsconfig.base.json (modo estricto), ESLint con typescript-eslint y Prettier.
Configurar Vitest en shared y server, con un test trivial en cada uno.
Crear el cliente con Vite + React que muestre una página de bienvenida.
Crear un servidor Express mínimo con GET /api/health.
Scripts en la raíz: dev (cliente y servidor a la vez), build, test, lint y typecheck.
Crear el workflow de GitHub Actions: instalar, lint, typecheck, test y build en cada push y pull request.
Escribir .env.example y un README con los pasos para levantar todo.
CA: `npm run dev` levanta cliente y servidor; /api/health responde ok; lint, typecheck y tests pasan; CI en verde.
Fase 1. Datos y esquemas del juego
Objetivo: todos los datos de la sección 2 en archivos JSON versionados y validados.
Tareas
Crear fighters.json, mercs.json, creeps.json, waves.json y rules.json con los valores de la sección 2. rules.json incluye las constantes (oro inicial, ingreso, límites, tamaño del tablero, tickRate, matriz de tipos, fórmula de armadura) y `dataVersion: 1.0.0`.
Crear esquemas Zod para cada archivo y derivar los tipos TypeScript con `z.infer`.
Crear `loadGameData()` que valida y congela los datos (Object.freeze) y precalcula las tablas de escalado de 20 valores.
Agregar el script `npm run validate:data`.
Pruebas: los datos cargan; los ids son únicos; toda referencia de waves existe en creeps; todos los números son finitos y positivos donde corresponde; un dato inválido produce un error claro.
CA: cambiar un número en un JSON cambia el juego sin tocar código; los tests de datos pasan.
Fase 2. Motor de simulación
Objetivo: `simulateLane` completo, determinista y probado, siguiendo las secciones 2.9 y 4.
Tareas
Implementar el generador mulberry32 y la huella FNV-1a.
Implementar entidades, aparición escalonada, elección de objetivos, movimiento con correa, ataques con cooldown, curación, fugas y fin de combate, en el orden de tick de la sección 4.2.
Agregar la regla de ESLint que prohíbe Math.random, Date.now y funciones trigonométricas dentro de src/sim.
Crear el script `npm run sim:cli` que lee un LaneInput desde un JSON y muestra el resultado (y opcionalmente un resumen por tick) para depurar.
Escribir tests unitarios por regla con casos pequeños que se pueden calcular a mano (un guerrero contra un esbirro, un sanador con un herido, un arquero contra un coloso, una fuga simple).
Escribir el test dorado, el test de repetición (misma entrada produce la misma salida 100 veces) y el test de rendimiento.
CA: todos los tests pasan; la huella dorada es estable; una ronda de 150 entidades corre en menos de 100 ms; ninguna API prohibida en src/sim.
Fase 3. Reglas de partida y Elo (puro, sin red)
Objetivo: una partida completa que se puede jugar y probar solo con funciones puras.
Tareas
Implementar `createMatch`, `applyAction`, `startBuildPhase`, `buildLaneInputs`, `applyRoundResult` y `checkVictory` (sección 5).
Implementar src/elo: probabilidad esperada, actualización con K variable, piso de 100, reducción al 50% por par repetido (la función recibe el contador como dato) y cálculo de división.
Escribir tests de cada código de error, de la venta al 100% y al 70%, del límite de 40 guerreros, del cobro de ingresos, del cruce de mercenarios entre carriles y del empate.
Escribir una prueba de integración sin red: dos bots aleatorios juegan 20 rondas y la partida termina con un resultado válido. En cada paso se verifican invariantes: oro nunca negativo, no más de 40 guerreros, sin celdas duplicadas y vidas de reyes coherentes.
Escribir tests de Elo con valores conocidos: 1000 contra 1000 da ±20 con K = 40; 1200 contra 1000 da una probabilidad esperada cercana a 0.76 para el de 1200.
CA: todos los tests pasan; la prueba de bots aleatorios corre 500 partidas sin violar invariantes.
Fase 4. Cliente jugable offline contra un bot
Objetivo: jugar una partida completa en el navegador contra un bot, sin servidor. Es el primer momento en que el juego se puede probar con las manos.
Tareas
Bot heurístico en shared/bots con tres dificultades (fácil, normal, difícil). Lógica base: en las primeras rondas compra guerreros baratos y equilibrados; reserva una parte del oro para mercenarios; elige guerreros según el tipo de armadura de la oleada siguiente (contraataque de tipos); envía mercenarios de forma proporcional al oro disponible. La dificultad cambia el porcentaje de gasto en economía y la calidad del contraataque.
Pantalla de partida: tablero PixiJS con la grilla de 10 por 18 y la zona de construcción resaltada; panel de tienda con los 8 guerreros (costo y tipos); panel de mercenarios; HUD con oro, ingreso, ronda, cuenta regresiva y vida de ambos reyes; botones Listo y Rendirse.
Interacción: elegir un guerrero y hacer clic en una celda para colocarlo; vender con clic derecho o botón; tooltips con estadísticas y ventajas de tipo.
Reproducción del combate: usar `simulateLane` con `onFrame`; barras de vida, destello al atacar, números de daño y velocidad x1 y x2.
Vista del rival: pestaña o miniatura con el carril del oponente.
Flujo local de rondas con las funciones puras y el mismo `MatchState` que usará el servidor.
Pantalla de fin de partida.
Arte provisorio con formas y emojis; el arte final llega en la Fase 9.
Pruebas: componentes clave con Vitest y Testing Library; un test e2e con Playwright que juega 3 rondas contra el bot.
CA: una partida completa contra el bot es jugable sin errores; el cliente no contiene números de juego; el combate visto produce la misma huella que `simulateLane` sin dibujar.
Fase 5. Backend base y autenticación
Objetivo: cuentas de usuario seguras y sesión persistente.
Tareas
Conexión a MongoDB con Mongoose y reintentos; configuración validada con Zod a partir de las variables de entorno.
Middlewares: helmet, CORS con lista blanca, límite de body de 100 KB, rate limit, id de solicitud, pino-http y un manejador de errores central con el formato de la sección 6.1.
Modelos `User` y `RefreshToken` con los índices de la sección 7.
Rutas de register, login, refresh, logout y GET /api/me con argon2id, rotación de refresh tokens y detección de reutilización (sección 8.1).
Tests con Supertest y mongodb-memory-server: registro correcto, usuario o email duplicado, contraseña incorrecta, rotación y reutilización del refresh, logout y límite de intentos.
En el cliente: páginas de login y registro, store de sesión, renovación automática del access token y rutas protegidas.
CA: un usuario se registra, cierra el navegador, vuelve y sigue con sesión; reutilizar un refresh token revoca la familia; los tests pasan.
Fase 6. Tiempo real y partidas online
Objetivo: dos jugadores reales juegan una partida completa a través del servidor autoritativo.
Tareas
Servidor Socket.IO con middleware de handshake que valida el JWT. Un solo socket activo por usuario: una conexión nueva reemplaza a la anterior.
MatchRoom: guarda el `MatchState` y los temporizadores. Su método de acciones valida con `applyAction` y respeta la idempotencia por `seq`. Al cerrar la construcción llama a `buildLaneInputs` y `simulateLane` para los dos carriles, emite `phase:combat:start`, espera la duración simulada más 2 s, aplica `applyRoundResult` y `checkVictory`, y pasa a la ronda siguiente.
MatchManager: crea y destruye salas y mantiene los mapas matchId a sala y userId a matchId.
Reconexión: al conectarse, si el usuario tiene una partida activa se le envía `match:state` y se avisa al rival con `opponent:status`. Hay 60 s de gracia; si vence, derrota por abandono.
Rendición con `match:surrender`.
Cliente: capa net/socket.ts con reconexión automática. Se reemplaza el flujo local de la Fase 4 por los eventos del servidor, manteniendo la misma interfaz. El combate se reproduce con los `LaneInput` recibidos y se compara la huella con la oficial; si difiere, se muestra el resultado oficial sin animación exacta.
Persistencia mínima: al terminar, guardar la partida en la colección matches con rondas y acciones (el Elo llega en la Fase 8).
Recuperación ante caídas (DEBERÍA): guardar un snapshot del `MatchState` al final de cada ronda y restaurar las salas activas con menos de 5 minutos al arrancar el servidor.
Pruebas: integración con dos clientes automatizados de socket.io-client jugando una partida completa contra un servidor real con base en memoria; desconexión y reconexión; acción duplicada; acción inválida; tiempo agotado de construcción; ambos Listo; abandono.
CA: dos navegadores reales juegan 20 rondas; cerrar y reabrir la pestaña devuelve a la partida; un cliente que envía acciones inválidas o repetidas no logra ninguna ventaja.
Fase 7. Matchmaking y modos de juego
Objetivo: que cualquier usuario entre a la web, pulse Jugar y encuentre una partida.
Tareas
Cola ranked según la sección 9.3, con un ciclo de emparejamiento cada segundo.
Eventos `queue:join`, `queue:leave`, `queue:status` y `match:found`. La semilla de la partida se genera en el servidor con `crypto.randomInt`.
Modo práctica contra bot en el servidor: el bot actúa como un jugador más dentro de MatchRoom, con un retraso aleatorio de 3 a 15 s para parecer humano.
Oferta de bot a los 25 s de espera en ranked (sin Elo).
Interfaz de lobby con botones Ranked y Práctica, contador de espera y botón Cancelar.
Restricciones: un usuario no puede estar en cola y en partida a la vez ni en dos partidas.
Pruebas: emparejamiento por ventana, ampliación de la ventana con el tiempo, doble cola, cancelación y una prueba de carga con 200 usuarios simulados encolando.
CA: dos usuarios nuevos en ranked se emparejan y juegan; uno solo recibe la oferta de bot a los 25 s; las pruebas de carga pasan.
Fase 8. Elo, ranking, perfil y replays
Objetivo: el sistema que da motivos para volver.
Tareas
Al terminar una partida ranked, aplicar el Elo con la marca `ratingApplied` (sección 9.1) y actualizar victorias, derrotas, empates, racha, mejor Elo y partidas provisionales. Aplicar la reducción del 50% por par repetido.
Guardar eloBefore y eloAfter en la partida y emitir `match:end` con los cambios.
Servicio de temporadas: crear la temporada activa al arrancar si no existe y programar el cierre idempotente de la sección 9.5.
Endpoints de leaderboard, perfil público, historial y partida (replay).
Interfaz: leaderboard con paginación, búsqueda por nombre y tu posición destacada; perfil con Elo, división, gráfico de evolución de las últimas 30 partidas, racha, estadísticas e insignias; historial con enlaces a replays.
Visor de replay: reproduce cada ronda con las entradas guardadas usando `simulateLane`, con controles de ronda y velocidad. URL compartible: /replay/:matchId.
Pantalla de fin de partida con el cambio de Elo animado y el botón Jugar otra vez.
Pruebas: el Elo se aplica una sola vez aunque dos procesos finalicen la misma partida; abandono; empate; cierre de temporada con reloj simulado; paginación; el replay reproduce la misma huella que la partida original.
CA: tras una partida ranked ambos jugadores ven su nuevo Elo; el leaderboard lo refleja; el replay se reproduce idéntico.
Fase 9. Interfaz y experiencia de usuario
Objetivo: un producto claro, atractivo y fácil de aprender.
Tareas
Páginas: landing con propuesta y botón Jugar, login y registro, lobby, partida, leaderboard, perfil, replay, Cómo jugar, ajustes y 404.
Tutorial interactivo de 3 rondas guiadas contra un bot: colocar un guerrero, entender tipos de ataque y armadura, enviar un mercenario.
Arte y audio: reemplazar las formas provisorias por sprites (pack con licencia libre o propio), animaciones de ataque, muerte y fuga, efectos de sonido y música con control de volumen. Las licencias y créditos se anotan en CREDITS.md.
Claridad táctica: indicador de ventaja o desventaja de tipo al elegir un guerrero; resumen de ronda con daño y fugas; vista previa de la oleada siguiente (tipo de armadura y cantidad).
Responsive: escritorio como objetivo principal; tablets y móviles en horizontal con controles táctiles (tocar para colocar, mantener para vender).
Accesibilidad: contraste suficiente, tamaño de texto, navegación por teclado en menús, iconos por tipo para no depender solo del color y opción de reducir animaciones.
Estados de carga, errores de red, aviso de reconexión y mensajes claros.
Textos en un archivo de traducciones (español primero, listo para sumar inglés).
Rendimiento: carga diferida de PixiJS, atlas de sprites y control del tamaño del bundle. Objetivo: 60 FPS con 150 entidades en un equipo medio.
CA: un jugador nuevo completa el tutorial y su primera partida sin ayuda externa; accesibilidad de Lighthouse de 90 o más en las páginas que no son de juego; 60 FPS en el equipo de referencia.
Fase 10. Seguridad, cuenta y anti-abuso
Objetivo: que el juego resista a usuarios malintencionados y cumpla lo mínimo para abrirse al público.
Tareas
Verificación de email y recuperación de contraseña con tokens de un solo uso y expiración corta (requiere proveedor de correo).
Controles de la sección 8.2 completos, con tests.
Anti-abuso del ranking de la sección 8.3.
Registro de acciones rechazadas y alerta ante patrones anómalos.
`npm audit` en CI y Dependabot para actualizaciones.
Revisión básica contra OWASP: inyección NoSQL, XSS (prohibir `dangerouslySetInnerHTML`), CSRF (SameSite y cabecera personalizada en el refresh) y errores que no filtren detalles internos.
Páginas de términos y privacidad, y borrado de cuenta (las partidas del usuario se anonimizan).
Moderación básica de nombres de usuario (lista de palabras prohibidas), rol admin y endpoint para banear.
Pruebas: intentos maliciosos (inyección, tokens falsos, exceso de solicitudes); ninguna ruta devuelve `passwordHash`.
CA: todos los controles tienen un test; no quedan vulnerabilidades altas en `npm audit`.
Fase 11. Balance, pruebas completas y carga
Objetivo: un juego equilibrado, probado de punta a punta y capaz de sostener la carga prevista.
Tareas
Torneo automático de bots (`npm run sim:tournament`): miles de partidas entre dificultades y estrategias. El reporte incluye porcentaje de victoria por estrategia, uso de cada guerrero y mercenario, duración media de partida, ronda en que suelen terminar y fugas por oleada.
Metas iniciales de balance: ningún guerrero con uso mayor al 35% ni menor al 5% en estrategias óptimas; ninguna estrategia pura que gane más del 60% contra una mezcla; ventaja del primer jugador menor al 52%; entre el 40% y el 70% de las partidas terminan antes de la ronda 20.
Ajustar los JSON y subir `dataVersion` en cada cambio. Los datos de cada versión se archivan en src/data/versions/<versión>/ y el servidor entrega la versión pedida, para que los replays viejos se reproduzcan con los datos con los que se jugaron. Si cambian las reglas del motor (no solo los datos), se sube también `engineVersion`; los replays de motores anteriores se marcan como legados y muestran solo el resultado.
E2E completo con Playwright: registro, cola, partida completa, Elo y leaderboard.
Pruebas de carga: un script crea N partidas simultáneas con clientes automáticos y mide CPU, memoria, latencia de ack y tiempo de simulación. Meta: 200 partidas simultáneas (400 sockets) con latencia p95 de ack menor a 200 ms y CPU menor al 70% en la instancia elegida.
Optimizaciones si no se cumple la meta: mover `simulateLane` a worker_threads, grilla espacial para objetivos y compresión de mensajes.
Pruebas de caos: reiniciar el servidor a mitad de partidas y cortar la red de los clientes.
Beta interna con 5 a 10 testers y una lista de errores priorizada.
CA: metas de balance cumplidas o documentadas como excepción; la carga cumple los objetivos; los tests e2e pasan en CI.
Fase 12. Despliegue y operación
Objetivo: el juego funcionando en internet de forma segura y operable.
Tareas
Tres entornos (local, staging y producción), cada uno con su base de datos y sus secretos.
MongoDB Atlas con réplicas, usuario de aplicación con permisos mínimos, lista de IPs permitidas y backups automáticos activados.
Cliente: build estático en un hosting de archivos estáticos con CDN. Servidor: servicio con WebSockets persistentes, una sola instancia al inicio. Las salas viven en memoria; escalar a varias instancias requiere sticky sessions, un adaptador de Redis y una cola distribuida (backlog).
Dominio propio con HTTPS. Se usan subdominios hermanos (app.dominio y api.dominio) para que las cookies SameSite funcionen; CORS ajustado a esos orígenes.
CI/CD: al unir a main se despliega a staging; la promoción a producción es manual tras pasar un e2e de humo.
Observabilidad: logs de pino centralizados, /api/health vigilado por un monitor de disponibilidad, métricas (partidas activas, jugadores en cola, tiempo de simulación, errores) con alertas, y un servicio de errores como Sentry en cliente y servidor.
Apagado ordenado: con SIGTERM el servidor deja de aceptar partidas nuevas, guarda snapshots y espera a que terminen las activas (máximo 5 minutos), para desplegar sin cortar partidas.
Tarea de retención que compacta los replays de más de 90 días.
Runbook con los pasos para reiniciar, restaurar un backup, cerrar una temporada a mano, banear a un usuario y revertir un despliegue.
Los planes y precios de los proveedores cambian: se verifican al momento de elegir. Existen opciones gratuitas o de bajo costo para empezar, con límites.
CA: despliegue desde cero en menos de una hora siguiendo el runbook; una partida real de punta a punta en producción; un backup restaurado con éxito en staging.
Fase 13. Lanzamiento, medición y retención
Objetivo: pasar de «funciona» a «la gente vuelve».
Tareas
Beta cerrada por invitación (50 a 100 jugadores) con canal de feedback.
Métricas: registros, partidas por usuario, retención a 1, 7 y 30 días, tiempo en cola, porcentaje de partidas contra bot, abandonos y duración de partida. Con analítica respetuosa de la privacidad (sin datos personales).
Corregir los 10 problemas principales y rebalancear con datos reales.
Lanzamiento público: landing, anuncio en comunidades de estrategia y Legion TD, y un Discord de soporte.
Primera temporada oficial con insignias.
Plan de contenido: nuevos guerreros y oleadas en cada temporada, cambiando solo JSON y versión.
CA: retención a 7 días medida; tiempo medio de cola menor a 30 s en horas pico (o plan de relleno con bots); sin errores críticos durante 7 días seguidos.
PARTE IV. CALIDAD, RIESGOS Y CIERRE
Estrategia de pruebas
Nivel	Herramienta	Qué cubre	Cuándo corre
Unitarias	Vitest	Motor, reglas, Elo, bots	Cada commit
Determinismo	Test dorado	Huella fija del motor	Cada commit
API	Supertest + mongodb-memory-server	Auth, rutas, errores	Cada commit
Sockets	socket.io-client + servidor real	Partida completa, reconexión, trampas	Cada pull request
E2E	Playwright	Flujo completo de usuario	Humo en cada PR; completo cada noche
Carga	Script propio	200 partidas simultáneas	Antes de cada release
Seguridad	npm audit y tests de ataques	Dependencias y abusos	Cada pull request
Meta de cobertura: 90% de líneas en packages/shared y 80% en apps/server.
Riesgos y mitigaciones
Riesgo	Impacto	Mitigación
Desincronización entre simulación de cliente y servidor	Animación incorrecta	Huella oficial, resultado del servidor como verdad, reglas de determinismo y test dorado
Poca población de jugadores	Colas largas y poca retención	Práctica con bots, ventana de búsqueda amplia, eventos de temporada, invitaciones
Balance defectuoso	Partidas aburridas o decididas de antemano	Torneos de bots, datos versionados y parches rápidos
Trampas	Ranking sin valor	Servidor autoritativo, validación total y límites de ritmo
Caída del servidor único	Partidas interrumpidas	Snapshots, apagado ordenado, reinicio automático y plan de escalado
Alcance excesivo	El proyecto no termina	MVP definido, backlog separado y fases con criterios de aceptación
Arte o audio con licencia dudosa	Problemas legales	Solo assets con licencia clara y CREDITS.md
Costos de hosting	Presupuesto	Empezar con una instancia pequeña y medir
Backlog después del MVP
Modo 2v2 con carriles compartidos y reparto de mercenarios.
Mejoras de guerreros (dos niveles) y habilidades activas.
Chat, amigos y partidas privadas por invitación.
Espectador en vivo y torneos.
Recompensas cosméticas por temporada.
App instalable (PWA) y soporte móvil completo.
Escalado horizontal (adaptador de Redis, sticky sessions, cola distribuida).
Detección de cuentas múltiples y sistema de reportes.
Glicko-2 en lugar de Elo.
Más mapas y modos.
Criterios de aceptación globales del proyecto
El proyecto está terminado cuando todo esto es verdadero:
Un usuario nuevo se registra, completa el tutorial y juega su primera partida en menos de 3 minutos.
Dos usuarios reales juegan una partida ranked completa y ambos ven su cambio de Elo.
Leaderboard, perfil, historial y replays funcionan.
Cerrar la pestaña a mitad de partida y volver restaura la partida.
Ningún cliente modificado logra ventaja (hay tests que lo prueban).
200 partidas simultáneas con latencia p95 de ack menor a 200 ms.
Todos los tests pasan en CI y se cumplen las metas de cobertura.
Está en producción con HTTPS, backups y monitoreo.
Hay una temporada activa y el cierre de temporada fue probado.
Términos de uso y política de privacidad publicados.
ANEXOS
Anexo A. Dependencias entre fases y entregables
Fase	Depende de	Tamaño	Entregable ejecutable
0	Nada	S	Repositorio con CI en verde
1	0	S	Datos del juego validados
2	1	L	Simulador de combate por línea de comandos
3	2	M	Partida completa simulada por tests
4	3	XL	Juego jugable contra bot en el navegador
5	0 (puede ir en paralelo con 2 a 4)	M	Registro e inicio de sesión funcionando
6	3, 4 y 5	XL	Partida online 1v1 entre dos navegadores
7	6	M	Botón Jugar con cola y bots
8	6 y 7	L	Elo, ranking, perfil y replays
9	4 (se completa tras la 8)	L	Producto pulido con tutorial
10	5 (puede solaparse con 8 y 9)	M	Cuentas seguras y anti-abuso
11	8	L	Juego balanceado, probado y medido
12	11	M	Juego en producción
13	12	M	Beta, lanzamiento y retención
Tamaños relativos: S pequeño, M mediano, L grande, XL muy grande.
Anexo B. Guía de trabajo para quien implemente (persona o asistente)
Leer completas las secciones 1 a 10 y la fase actual antes de escribir código.
Trabajar una tarea por vez y escribir los tests junto con el código.
Antes de cerrar una fase: ejecutar lint, typecheck y tests, y revisar los CA uno por uno informando cuáles se cumplen.
No inventar reglas. Si algo no está definido, anotar la pregunta y usar el valor por defecto de las decisiones abiertas.
No agregar dependencias nuevas sin justificarlas en el README.
Todo cambio de reglas o números se hace primero en src/data (o en este documento) y nunca de forma dispersa en el código.
Commits pequeños y una rama por fase.
Si un test dorado cambia, es un cambio de reglas: se sube `dataVersion` o `engineVersion` y se explica el motivo.
Anexo C. Ejemplo mínimo de LaneInput
```ts
const input: LaneInput = {
  dataVersion: '1.0.0',
  seed: 123456,
  round: 1,
  fighters: \[
    { uid: 1, typeId: 'guardian', x: 4.5, y: 10.5 },
    { uid: 2, typeId: 'arquero', x: 5.5, y: 12.5 },
  ],
  wave: \[{ typeId: 'esbirro', count: 6 }],
  mercs: \[],
};
```
Anexo D. Glosario
Tick: paso de simulación de 0.05 s (20 por segundo).
Guerrero: unidad defensora que compra y coloca el jugador.
Mercenario: unidad que se envía al carril del rival a cambio de oro y que sube el ingreso propio.
Fuga: enemigo que llega al rey y le resta vida.
LaneInput: todos los datos necesarios para simular el combate de un carril en una ronda.
Huella (hash): valor corto que resume el estado final de una simulación; sirve para comprobar que dos simulaciones coinciden.
Autoritativo: el servidor es quien decide; el cliente solo propone y muestra.
Determinista: misma entrada, misma salida, siempre.
Partidas provisionales: primeras partidas ranked de un jugador, con mayor cambio de Elo.
Replay: reproducción de una partida a partir de sus entradas guardadas.
Decisiones abiertas (preguntas para vos, con valor por defecto)
Si no respondés alguna, se usa el valor por defecto.
#	Decisión	Valor por defecto
1	¿TypeScript o JavaScript puro?	TypeScript (los tipos compartidos evitan errores entre cliente y servidor)
2	¿Nombre del juego?	Legión Web (provisorio)
3	¿Modo del MVP: solo 1v1 o ya apuntar a 2v2?	Solo 1v1; el 2v2 va al backlog
4	¿Idioma de la interfaz?	Español primero, inglés después
5	¿Proveedor de correo para verificar email y recuperar contraseña?	Se decide en la Fase 10; hasta entonces el registro no verifica el email
6	¿Dónde alojar el servidor?	Un servicio con WebSockets persistentes; se decide en la Fase 12
7	¿Dónde alojar el cliente?	Un hosting estático con CDN; se decide en la Fase 12
8	¿Arte y audio propios, de un pack con licencia libre o generados?	Pack con licencia libre y créditos en CREDITS.md
9	¿Duración de la temporada?	8 semanas
10	¿Login con Google o Discord?	No en el MVP
11	¿Presupuesto mensual de hosting?	Plan gratuito o de bajo costo para empezar
12	¿Dispositivos objetivo?	Escritorio primero; móvil en horizontal como secundario
13	¿Cómo se revisan los avances?	Al terminar cada fase se verifican los criterios de aceptación antes de seguir
