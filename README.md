# 📌 RoomMakerBack

API em **Spring Boot** para gerenciamento de **salas virtuais**, utilizando **MongoDB**, autenticação via **JWT** e **WebSockets** para comunicação em tempo real.

O sistema permite que usuários **criem, procurem, entrem, saiam e excluam salas**. Cada sala pode ser de diferentes categorias, como **Jogo da Velha**, **Jokenpô** ou só **Bate-papo**, e todas possuem um **chat em tempo real via WebSocket**. O dono também pode escolher ou não uma senha para entrar na sala.

Também é possível recuperar senha da conta por email.

Veja a aplicação completa hospedada [aqui](https://room-maker-front.vercel.app/)

Veja o código do Front-End [aqui](https://github.com/Gustavoksbr/RoomMakerFront)

---

## Rodando com Docker

```bash
docker compose up --build
```

> Por padrão o `.env` aponta para o MongoDB Atlas. Para usar o container local, altere `ROOMMAKER_MONGODB_URI` para `mongodb://mongodb:27017/roommaker`.

---

## Endpoints HTTP

Base URL: `http://localhost:8080`

Rotas protegidas exigem o header `Authorization: Bearer <token>`.

---

### Utilitários

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| GET | `/ping` | Não | Health check — retorna `"pong"` |
| GET | `/teapot` | Não | Retorna erro 418 I'm a teapot |

---

### Autenticação e Usuários

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| POST | `/cadastro` | Não | Cria um novo usuário |
| POST | `/login` | Não | Autentica e retorna JWT |
| POST | `/login2fa` | Não | Segunda etapa do login com 2FA |
| POST | `/usuario/esquecisenha` | Não | Envia código de recuperação por email |
| POST | `/usuario/novasenha` | Não | Redefine a senha com o código recebido |
| PUT | `/usuario/doisfatores` | Sim | Habilita/desabilita autenticação de dois fatores |
| GET | `/usuarios?substring=` | Não | Lista usuários filtrando por substring do username |
| GET | `/datanascimento` | Sim | Retorna a data de nascimento do usuário autenticado |

#### POST `/cadastro`
```json
// Request
{
  "username": "string (max 15, apenas letras e números)",
  "password": "string (min 3, max 64)",
  "email": "string",
  "descricao": "string (opcional)",
  "dataNascimento": "YYYY-MM-DD (opcional)"
}
// Response
{ "token": "string" }
```

#### POST `/login`
```json
// Request
{ "username": "string", "password": "string" }
// Response
{ "token": "string" }
```

#### POST `/login2fa`
```json
// Request
{ "email": "string", "codigo": "string" }
// Response
{ "token": "string" }
```

#### POST `/usuario/esquecisenha`
```json
// Request
{ "email": "string" }
// Response: 200 OK (sem corpo)
```

#### POST `/usuario/novasenha`
```json
// Request
{ "email": "string", "codigo": "string", "password": "string (min 3, max 64)" }
// Response
{ "token": "string" }
```

#### GET `/usuarios?substring=`
```json
// Response
[{ "username": "string", "email": "string", "descricao": "string" }]
```

---

### Salas

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| GET | `/salas` | Não | Lista salas com filtros opcionais |
| GET | `/salas/_convidado?usernameParticipante=` | Não | Lista salas onde o usuário é participante |
| GET | `/salas/_dono?usernameDono=` | Não | Lista salas onde o usuário é dono |
| GET | `/salas/{usernameDono}/{nomeSala}` | Não | Retorna detalhes de uma sala |
| POST | `/salas` | Sim | Cria uma nova sala |
| POST | `/salas/{usernameDono}/{nomeSala}` | Sim | Entra em uma sala (com ou sem senha) |
| DELETE | `/salas/{usernameDono}/{nomeSala}` | Sim | Exclui uma sala (apenas o dono) |
| DELETE | `/salas/{usernameDono}/{nomeSala}/{usernameParticipante}` | Sim | Sai de uma sala |

#### GET `/salas?usernameDono=&nome=&categoria=`
```json
// Response
[{
  "id": "string",
  "usernameDono": "string",
  "nome": "string",
  "categoria": "string",
  "qtdCapacidade": 0,
  "disponivel": true,
  "publica": true,
  "usernameParticipantes": ["string"]
}]
```

#### POST `/salas`
```json
// Request
{
  "nome": "string (max 15, apenas letras e números)",
  "categoria": "string (ex: chat, tictactoe, jokenpo, whoistheimpostor)",
  "senha": "string (opcional — omitir para sala pública)",
  "qtdCapacidade": 2
}
// Response: SalaResponse (mesmo formato acima)
```

#### POST `/salas/{usernameDono}/{nomeSala}`
```json
// Request
{ "senha": "string (enviar vazio se sala pública)" }
// Response: SalaResponse
```

---

### Who Is The Impostor (HTTP)

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| GET | `/categorias/whoistheimpostor/{usernameDono}/{nomeSala}/mostrar` | Sim | Retorna o estado atual do jogo para o usuário autenticado |

#### GET `/categorias/whoistheimpostor/{usernameDono}/{nomeSala}/mostrar`
```json
// Response
{
  "partidaSendoJogada": true,
  "impostorDaPartidaPassada": "string",
  "cartaDaPartidaPassada": {},
  "votosPorVotadosDaPartidaPassada": {},
  "jogadoresDaPartidaPassada": ["string"],
  "estaNaPartida": true,
  "isImpostor": false,
  "carta": {},
  "jogadoresNaPartida": ["string"],
  "quantidadeVotos": 0,
  "votado": "string"
}
```

---

## WebSockets (STOMP)

### Conexão

- Endpoint de conexão: `ws://localhost:8080/socket`
- Prefixo de envio (cliente → servidor): `/app`
- Prefixo de subscrição (servidor → cliente): `/topic`
- Autenticação: enviar o JWT no header `Authorization` durante o handshake STOMP

### Padrão de tópico de resposta

Cada mensagem enviada pelo servidor é entregue individualmente para cada usuário da sala:

```
/topic/sala/{usernameDono}/{nomeSala}/{username}/{tipo}
```

---

### Chat

#### Enviar — carregar histórico do chat
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/chat
Payload : (sem payload)
```
```json
// Resposta no tópico /topic/sala/{usernameDono}/{nomeSala}/{username}/chat
{
  "usernameDono": "string",
  "nomeSala": "string",
  "messages": [{
    "ordem": 0,
    "message": "string",
    "from": "string",
    "to": null,
    "timestamp": "ISO-8601"
  }]
}
```

#### Enviar — nova mensagem
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/chat/message
Payload : { "message": "string (max 200 chars)", "to": null }
```
```json
// Resposta no tópico /topic/sala/{usernameDono}/{nomeSala}/{username}/chat
// (broadcast para todos os participantes + dono)
{
  "ordem": 0,
  "message": "string",
  "from": "string",
  "to": null,
  "timestamp": "ISO-8601"
}
```

---

### Jogo da Velha (TicTacToe)

#### Enviar — entrar/iniciar partida
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/tictactoe
Payload : (sem payload)
```
```json
// Resposta no tópico /topic/sala/{usernameDono}/{nomeSala}/{username}/tictactoe
// Aguardando oponente:
{ "x": null, "o": null, "posicao": "_________", "status": "WAITING" }

// Partida iniciada:
{
  "nomeSala": "string",
  "usernameDono": "string",
  "usernameOponente": "string",
  "jogoAtual": { "x": "string", "o": "string", "posicao": "string", "status": "string" },
  "historico": [],
  "tamanhoHistorico": 0
}
```

#### Enviar — fazer lance
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/tictactoe/lance
Payload : { "lance": 0 }   // posição de 0 a 8 no tabuleiro
```
```json
// Resposta no tópico /topic/sala/{usernameDono}/{nomeSala}/{username}/tictactoe
{
  "x": "string",
  "o": "string",
  "posicao": "string (9 chars: X, O ou _)",
  "status": "WAITING | X_WON | O_WON | DRAW | IN_PROGRESS"
}
```

---

### Jokenpô (Jokenpo)

#### Enviar — carregar estado da sala
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/jokenpo
Payload : (sem payload)
```

#### Enviar — fazer lance
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/jokenpo/lance
Payload : { "lance": "PEDRA | PAPEL | TESOURA" }
```
```json
// Resposta no tópico /topic/sala/{usernameDono}/{nomeSala}/{username}/jokenpo
{
  "numero": 1,
  "lanceDono": "PEDRA | PAPEL | TESOURA | ESPERANDO | SEGREDO",
  "lanceOponente": "PEDRA | PAPEL | TESOURA | ESPERANDO | SEGREDO",
  "status": "WAITING | DONO_GANHOU | OPONENTE_GANHOU | EMPATE"
}
```
> Enquanto o status for `WAITING`, o lance do adversário aparece como `SEGREDO`.

---

### Who Is The Impostor (WebSocket)

#### Enviar — iniciar partida
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/whoistheimpostor/comecar
Payload : (sem payload)
```

#### Enviar — terminar partida
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/whoistheimpostor/terminar
Payload : (sem payload)
```

#### Enviar — votar
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/whoistheimpostor/votar
Payload : { "voto": "string (username do suspeito)" }
```

#### Enviar — cancelar voto
```
Destino : /app/sala/{usernameDono}/{nomeSala}/{username}/whoistheimpostor/cancelarVoto
Payload : (sem payload)
```

> As respostas de todos os eventos acima chegam no tópico `/topic/sala/{usernameDono}/{nomeSala}/{username}/whoistheimpostor` com o formato `WhoIsTheImpostorResponse` (ver endpoint HTTP de mostrar acima).
