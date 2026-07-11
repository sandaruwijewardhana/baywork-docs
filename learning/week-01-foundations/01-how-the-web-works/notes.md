# Week 1 · Topic 1 — How the Web Works
### Client–server model · request/response · what a "server" really is

---

## 0. What you must be able to do after this topic

- Explain, in one sentence and in five minutes, how the web works.
- Say precisely **what a server is** (and destroy the "special big computer" myth).
- Define **client** and **server** and give three examples of each.
- Walk through the **request–response cycle** step by step, from memory.
- Describe what is inside an HTTP **request** and a **response** (the parts, not yet every header).
- Explain what it means for a server to **"listen on a port."**
- Explain why HTTP is **stateless** and why that matters.
- Draw the whole thing on a whiteboard and narrate it.

If you can do all of the above out loud, you pass. The `self-test.md` checks each one.

---

## 1. The one-sentence truth (lead with this)

> **The web is just programs on different computers sending each other text messages over a network: one side (the client) asks for something, the other side (the server) answers.**

Everything else in this topic is detail hanging off that sentence. Memorize it.

---

## 2. Bust the illusion: what a "server" actually is

Beginners picture a server as a mysterious, giant, humming machine in a data center. **Wrong mental model.** Here is the truth, bottom-up:

**A server is a *program* that runs continuously and waits for incoming requests, then sends back responses.**

- It is **software**, not hardware. `node index.js` running Express *is* a server. Nginx is a server. Postgres is a server.
- The **computer** that runs that program is *also* loosely called "a server" — but that's a nickname for the machine hosting the program.
- **Your own laptop can be a server.** When you run `docker compose up` and Express listens on port 3001, your laptop is now a web server. Nothing special happened to the hardware.
- "Server-grade hardware" (rack machines, ECC memory, redundant power) is about **reliability and uptime**, not a different *kind* of computer. A server is defined by its **role** (it serves), not its chassis.

**Minimum definition — a server does three things:**
1. **Listens** for incoming connections on a network address (an IP + a port).
2. **Accepts** a request when one arrives.
3. **Responds** — runs some code and sends data back.

**A client is the opposite role:** the program that **initiates** the request and consumes the response.

| Role | Who plays it | What it does |
|------|-------------|--------------|
| **Client** | Web browser, mobile app, `curl`, Postman, *another server* | **Initiates** the request, waits for the answer, uses it |
| **Server** | Express/Node app, Nginx, Postgres, an API | **Waits**, accepts requests, sends responses |

**Key insight:** *client* and *server* are **roles, not machines.** The same computer — even the same program — can be a client in one interaction and a server in another. Your Express API is a **server** to the browser, but a **client** to the Postgres database.

---

## 3. The client–server model

The web runs on the **client–server model**. Its rules:

1. **The client always speaks first.** In classic HTTP, the server never contacts the client out of the blue — it only ever *replies* to a request. (WebSockets and Server-Sent Events change this later; ignore them for now.)
2. **Request-driven.** No request → no response. The server sits idle, listening, until asked.
3. **One server, many clients.** A single server handles many clients at once (concurrency). Thousands of browsers can hit the same server.
4. **The network is the boundary.** Client and server are separate programs, usually on separate computers, that can *only* communicate by sending messages across the network. They share no memory, no variables.

**Contrast — peer-to-peer (P2P):** in P2P (e.g. BitTorrent) every node is both client and server to the others; there's no central "answerer." The web is overwhelmingly client–server, not P2P.

---

## 4. The request–response cycle (the heartbeat of the web)

This is the single most important sequence in web development. Learn it cold:

```
   CLIENT (browser)                              SERVER (your Express app)
        |                                                  |
   (1)  | build a REQUEST (method + URL + headers + body)  |
        |                                                  |
   (2)  | find the server's address  ──DNS──► name → IP    |   (name lookup; detail in Topic on DNS)
        |                                                  |
   (3)  | open a connection to  IP:port  (TCP handshake)   |   (transport; detail in Week 11)
        |                                                  |
   (4)  | ───────────── send the REQUEST ───────────────► |
        |                                                  |  (5) receive, run code,
        |                                                  |      maybe query the database,
        |                                                  |      build a RESPONSE
        |                                                  |
   (6)  | ◄──────────── send the RESPONSE ─────────────── |  (status + headers + body)
        |                                                  |
   (7)  | render / use the response                        |
        |                                                  |
   (8)  | connection closed or kept alive for reuse        |
```

Narrate it in words (interview version):
> "The client builds a request, resolves the server's name to an IP via DNS, opens a TCP connection to that IP and port, and sends the request. The server receives it, runs its code — often hitting a database — builds a response with a status code, headers, and a body, and sends it back over the same connection. The client then renders or uses the response, and the connection is closed or reused."

At this stage you don't need the DNS/TCP internals (those are dedicated later topics) — but you must know **where they sit** in the cycle.

---

## 5. What's inside a request and a response

Both are just **plain text** (for HTTP/1.1) with a strict shape. This is why the web is so debuggable — you can read it.

### 5.1 An HTTP request

```
GET /login.html HTTP/1.1          ← request line: METHOD  PATH  VERSION
Host: baywork.lk                  ← headers (key: value), one per line
User-Agent: Mozilla/5.0
Accept: text/html
                                  ← a BLANK line marks the end of headers
(optional body goes here)         ← body: present for POST/PUT/PATCH, empty for GET
```

The four parts of a request:
1. **Method** — the *verb*, the intent: `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove).
2. **Path (URL path)** — *what* resource: `/login.html`, `/api/customers`.
3. **Headers** — metadata about the request: which host, what format the client accepts, auth tokens, cookies.
4. **Body** — the *payload* (only for methods that send data, e.g. a JSON object on `POST`).

### 5.2 An HTTP response

```
HTTP/1.1 200 OK                   ← status line: VERSION  STATUS-CODE  REASON
Content-Type: text/html           ← headers
Content-Length: 1024
                                  ← blank line
<!doctype html>...                ← body: the actual content (HTML, JSON, an image, ...)
```

The three parts of a response:
1. **Status code** — did it work? `200` OK, `201` Created, `301`/`302` redirect, `400` bad request, `401` unauthorized, `404` not found, `500` server error. (Full status-code study is a later Week-1 bullet.)
2. **Headers** — metadata: `Content-Type` (what kind of data the body is), `Content-Length`, `Set-Cookie`, caching directives.
3. **Body** — the actual content: an HTML page, a JSON object, an image's bytes, etc.

**The header that changes everything: `Content-Type`.** The *same bytes* are treated completely differently depending on this header — `text/html` gets rendered as a page, `application/json` gets parsed as data, `image/png` gets drawn as an image. (You saw this exact idea earlier with `express.static`.)

---

## 6. Frontend vs backend, in this model

- **Frontend = client-side.** The code that runs **in the browser**: HTML, CSS, JavaScript. It builds and sends requests, and renders responses.
- **Backend = server-side.** The code that runs **on the server**: your Node/Express app plus the database. It receives requests, runs business logic, talks to the database, and returns responses.
- **The network is the wall between them.** They never share variables or memory. Their *only* channel is requests and responses — almost always **HTTP carrying JSON** for APIs.

This is why, in your BayWork app, `console/js/api.js` (frontend) can only reach the database by sending an HTTP request to Express (backend), which then queries Postgres. The browser can never touch Postgres directly.

---

## 7. What "a server listens on a port" means (bottom-up)

When we say a server "listens on port 3001," here's the physical reality:

- A running computer has **one (or a few) IP addresses** — its identity on the network (e.g. `192.168.1.10`, or `127.0.0.1` for itself).
- The IP gets a message *to the machine*. But a machine runs many programs. **Which program should receive it?** That's what a **port** answers.
- A **port** is just a number (0–65535) that identifies a specific program's "mailbox" on that machine.
- A server program asks the operating system: *"give me port 3001; deliver anything that arrives there to me."* That's **binding + listening**.
- The pair **`IP:port`** (e.g. `127.0.0.1:3001`) is a complete address for *a specific program on a specific machine*. This pair is called a **socket address**.
- Well-known defaults: **HTTP = port 80**, **HTTPS = port 443**, Postgres = 5432. When you type `http://baywork.lk` with no port, the browser assumes **80** (or **443** for `https`).

You don't need TCP/socket internals yet (Week 11) — but you must be able to say: *"A server listens on a port; the IP finds the machine, the port finds the program, and IP:port together is a socket address."*

---

## 8. Statelessness (why the web forgets you)

**HTTP is stateless: each request is completely independent. By default, the server remembers nothing about previous requests from the same client.**

- Request #2 has no memory of request #1. The server treats every request as a stranger knocking for the first time.
- **Why it matters:** if the server forgets you between requests, how does it know you're logged in on the second page? It can't — from HTTP alone. That's exactly why we invented **cookies** and **tokens** (JWT): they make the client *re-present its identity on every single request*. (Full auth is Week 9 — but the *reason* it exists is statelessness.)
- **Benefit:** statelessness is what lets **one server, or many identical servers behind a load balancer**, handle any request interchangeably. No request depends on server memory, so you can scale horizontally.

---

## 9. End-to-end trace: "what happens when I load a page?" (this-topic level)

You type `https://baywork.lk/login` and press Enter:

1. The browser (client) decides it needs to **GET** the resource at that URL.
2. It resolves `baywork.lk` to an IP address (**DNS** — covered in its own topic).
3. It opens a connection to that **IP on port 443** (HTTPS) and secures it (TLS — later).
4. It sends an **HTTP request**: `GET /login HTTP/1.1`, `Host: baywork.lk`, plus headers.
5. The **server** receives it, runs code, and returns a **response**: `200 OK`, `Content-Type: text/html`, and the HTML body.
6. The browser reads `Content-Type: text/html` and **renders** the page.
7. While parsing, it finds `<link href="styles.css">` and `<script src="app.js">` and fires **more requests** for each — every asset is its own request–response cycle.
8. `app.js` later calls `fetch('/api/session')` → **another** request → the server replies with JSON → the page updates.

That's the entire web: one page load is *many* request–response cycles stacked together. (The deeper "type a URL and press Enter" answer — DNS, TCP handshake, TLS, rendering pipeline — is built up across later topics. Here you own the **client asks / server answers** skeleton.)

---

## 10. Mental models & analogies (use these in interviews)

- **Restaurant:** you (client) give an order (request) to the waiter; the kitchen (server) cooks and returns your dish (response). You must order first; the kitchen never sends food to strangers unprompted. Many customers, one kitchen.
- **Letters, not a phone call:** each HTTP request is like mailing a self-contained letter — it must include everything needed (address, return address, contents), because the server keeps no memory of your last letter (statelessness).
- **Receptionist analogy for ports:** the IP address gets your letter to the *building*; the port number is the *room/department* it must be delivered to.

---

## 11. Common misconceptions (interview traps)

| Myth | Reality |
|------|---------|
| "A server is a special/huge computer." | A server is a **program that listens and responds**; the hardware is ordinary. Your laptop can be one. |
| "The server pushes data to me whenever it wants." | Classic HTTP is **client-initiated only**. The server just replies. (WebSockets/SSE add push — later.) |
| "Frontend and backend are one program." | They're **separate programs across the network**, talking only via requests/responses. |
| "`localhost` is somewhere on the internet." | `localhost` / `127.0.0.1` is **your own machine** talking to itself. Nothing leaves the computer. |
| "The URL and the server are the same thing." | The URL is an **address**; DNS maps its hostname to the server's IP. |
| "GET vs POST is about security." | It's about **intent/semantics** (read vs create) and where data goes (URL vs body), not secrecy. |

---

## 12. Glossary (be able to define each in one line)

- **Client** — program that initiates a request and consumes the response (browser, app, curl).
- **Server** — program that listens for requests and returns responses.
- **Request** — a message from client to server: method + path + headers + optional body.
- **Response** — a message from server to client: status code + headers + body.
- **HTTP** — the text-based rules (protocol) clients and servers use to talk over the web.
- **URL** — the address of a resource (`https://host:port/path?query`).
- **IP address** — the numeric identity of a machine on a network.
- **Port** — a number identifying a specific program on a machine.
- **Socket address** — `IP:port`, a complete address of a program on a machine.
- **DNS** — the system that maps a hostname (`baywork.lk`) to an IP address.
- **Host** — the machine (or its name) that a resource lives on.
- **localhost / 127.0.0.1** — a name/address that always means "this same machine."
- **Frontend / backend** — client-side code (browser) / server-side code (server + DB).
- **Stateless** — each request is independent; the server keeps no memory between requests by default.

---

## 13. Interview soundbites (memorize these crisp answers)

- **"What is a server?"** → "A program that runs continuously, listens on a network port, and responds to incoming requests. The machine that runs it is loosely called a server too, but a server is a *role*, not special hardware."
- **"Explain the client–server model."** → "The client initiates a request; the server listens, processes it, and returns a response. The client always speaks first, one server serves many clients, and they communicate only over the network."
- **"Walk me through the request–response cycle."** → "Client builds a request, resolves the host to an IP via DNS, opens a connection to IP:port, sends the request; the server processes it — often querying a database — and returns a status code, headers, and body; the client renders or uses it."
- **"Frontend vs backend?"** → "Frontend runs in the browser (HTML/CSS/JS) and sends requests; backend runs on the server (app + database) and returns responses. The network separates them; they talk over HTTP, usually with JSON."
- **"Why is HTTP stateless and why does that matter?"** → "Each request is independent — the server remembers nothing by default. That's why we need cookies/tokens to carry identity on every request, and it's what lets us scale across many interchangeable servers."
- **"What does it mean for a server to listen on a port?"** → "It asks the OS to deliver any network data arriving on that port number to it. The IP finds the machine; the port finds the program; together IP:port is a socket address."

---

*Next: do every task in `exercises.md`, then grade yourself with `self-test.md`. You only pass when you can answer all of §0 out loud.*
