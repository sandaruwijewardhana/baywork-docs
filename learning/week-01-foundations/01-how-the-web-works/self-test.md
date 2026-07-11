# Self-Test — How the Web Works
### Interview-style. Answer CLOSED-BOOK and out loud/in writing BEFORE looking at the key.

Score: give yourself 1 point per question only if your answer matches the key's key ideas.
**Pass = 25/28 and you can say the six soundbites cold.** Anything you miss → re-read that section, retry tomorrow.

---

## Section 1 — Core definitions
1. What is a **server**? (Don't say "a computer.")
2. What is a **client**? Give three examples.
3. Are "client" and "server" **machines** or **roles**? Justify with one example.
4. Define the **client–server model** in two sentences.
5. What is an HTTP **request** made of? (Name the parts.)
6. What is an HTTP **response** made of? (Name the parts.)

## Section 2 — The cycle
7. Walk through the **request–response cycle** step by step.
8. In classic HTTP, **who speaks first** — client or server? Why does that matter?
9. Why does loading **one web page** produce **many** requests?
10. Your browser's JS calls `fetch('/api/customers')`. Is that a new request–response cycle? What does the server send back, typically?

## Section 3 — Servers, ports, addresses
11. What does it mean for a server to **"listen on a port"**?
12. What's the difference between an **IP address** and a **port**?
13. What is a **socket address**?
14. What port does `http://` default to? What about `https://`?
15. What does **`localhost`** (127.0.0.1) mean? Does traffic to it leave your machine?

## Section 4 — Frontend / backend / statelessness
16. Define **frontend** and **backend** and name the boundary between them.
17. In BayWork, can the browser query PostgreSQL directly? Explain.
18. What does it mean that **HTTP is stateless**?
19. Given statelessness, how does a server know you're **logged in** on your second request?
20. Give one **benefit** of statelessness for scaling.

## Section 5 — Headers, methods, status (intro)
21. What does the **`Content-Type`** response header control? Give two examples.
22. Name four HTTP **methods** and the intent of each.
23. What do these status codes mean: **200, 301, 404, 500**?
24. Is choosing **GET vs POST** about security? If not, what is it about?

## Section 6 — Traps & reasoning
25. True/False: "A server must be special, powerful hardware." Explain.
26. True/False: "A server can push data to the browser any time it wants." Explain (classic HTTP).
27. Your Express API calls the Postgres database. In *that* interaction, is your API a client or a server? Why?
28. A friend says "the URL *is* the server." Correct them precisely.

---

# ANSWER KEY (look only after answering)

<details>
<summary>Reveal answer key</summary>

1. A **program** that runs continuously, **listens** on a network port, accepts incoming requests, and returns **responses**. The hosting machine is loosely called a server too, but "server" is a **role**, not special hardware.
2. The program that **initiates** a request and consumes the response. Examples: web browser, mobile app, `curl`/Postman, or another server calling an API.
3. **Roles**, not machines. Example: your Express app is a **server** to the browser but a **client** to Postgres — same program, different role per interaction.
4. The client initiates a request; the server listens, processes it, and returns a response. The client always speaks first, one server serves many clients, and they communicate only over the network.
5. **Method** (GET/POST/…), **path/URL**, **headers**, and an optional **body**.
6. **Status code**, **headers**, and a **body**.
7. Client builds a request → resolves host to IP via **DNS** → opens a connection to **IP:port** → sends the request → server **processes** it (runs code, maybe queries the DB) → server returns **status + headers + body** → client **renders/uses** it → connection closed or reused.
8. The **client** speaks first. It matters because the server is passive — it only ever *replies*; no request means no response. (This is why "push" needs extra tech like WebSockets/SSE.)
9. The HTML document references other resources (CSS, JS, images, fonts); **each is its own request–response cycle**. Plus later JS `fetch` calls add more.
10. Yes — a full new cycle. The server typically returns **JSON** (with `Content-Type: application/json`), which the JS parses and uses to update the page.
11. The server asks the OS to **deliver any network data arriving on that port number to it** — i.e. it binds to the port and waits. 
12. **IP address** identifies the **machine** on the network; **port** identifies the **specific program** on that machine.
13. **`IP:port`** — a complete address for a specific program on a specific machine (e.g. `127.0.0.1:3001`).
14. `http://` defaults to **port 80**; `https://` defaults to **port 443**.
15. `localhost`/`127.0.0.1` means **"this same machine."** Traffic to it **does not leave the computer** — the machine talks to itself (loopback).
16. **Frontend** = client-side code running in the **browser** (HTML/CSS/JS). **Backend** = server-side code (app + database). The **network** is the boundary; they talk only via HTTP requests/responses.
17. **No.** The browser can only send HTTP requests to the Express **backend**, which then queries Postgres. The DB is never exposed to the browser.
18. Each request is **independent**; by default the server keeps **no memory** of previous requests from the same client.
19. The client must **re-present its identity on every request** — via a **cookie or token** (e.g. JWT) sent with each request. (Full detail later.)
20. Because no request depends on server memory, **any server (or many identical servers behind a load balancer)** can handle any request → easy **horizontal scaling**.
21. It tells the client **how to interpret the response body**. Examples: `text/html` → render as a page; `application/json` → parse as data; `image/png` → draw as an image.
22. **GET** (read), **POST** (create), **PUT/PATCH** (update), **DELETE** (remove). (Any four with correct intent.)
23. **200** OK (success); **301** moved permanently (redirect); **404** not found; **500** internal server error.
24. **Not** about security. It's about **intent/semantics** (read vs create) and **where data goes** (GET → in the URL/query; POST → in the body). Neither is encrypted by itself — that's HTTPS's job.
25. **False.** A server is defined by its **role** (it listens and responds). Ordinary hardware — even your laptop — can be a server. Powerful/redundant hardware is about **reliability/uptime**, not being a "server."
26. **False** for classic HTTP — it only **replies** to client requests. Server-initiated push requires **WebSockets** or **Server-Sent Events** (later topics).
27. A **client** — because in that interaction your API **initiates** the request to the database and waits for the response. Roles depend on who initiates.
28. The **URL is an address**, not the server. It contains a **hostname** that **DNS** resolves to the server's **IP address**; the server is the program listening at that IP and port.

</details>

---

## Mastery gate
- [ ] Scored ≥ 25/28 closed-book
- [ ] Can recite the 6 interview soundbites (notes §13) cold
- [ ] Can draw + narrate the request–response cycle on a blank page
- [ ] Explained the whole topic out loud in 5 minutes without notes

When all four are checked, mark Topic 1 complete in `../../README.md` and move to Topic 2.
