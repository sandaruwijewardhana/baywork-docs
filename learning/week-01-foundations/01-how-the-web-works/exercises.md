# Exercises — How the Web Works
### Do these by hand. No AI, no copy-paste of answers. Write everything down.

> You need only a **browser** and a **terminal** (Git Bash or PowerShell — both have `curl` on Windows 10+).
> No coding knowledge required for Week 1. The goal is to *observe the request–response cycle happening for real.*

---

## Part A — See real requests in the browser (DevTools)

**A1.** Open any website (e.g. `https://example.com`). Press **F12** → click the **Network** tab → **reload** the page.
- Click the very first request in the list (the document itself).
- Write down: the **Request Method**, the **Status Code**, the **Request URL**, and the value of the response **`Content-Type`** header.

**A2.** In that same Network tab, count how many requests the page made to load fully (look at the bottom bar: "N requests").
- Write one sentence explaining **why a single page load produced many requests** (link back to §9 of the notes).

**A3.** Find one request whose response `Content-Type` is **not** `text/html` (e.g. a CSS, JS, image, or font file).
- Write down its `Content-Type`. Explain in one line why the browser treats it differently from the HTML.

**A4.** Reload again and watch the **Status** column. Find any request with a status that is **not** `200`.
- If you find a `301`/`302` (redirect), `304` (not modified), or `404`, write it down and note what it means. (If all are 200, try a URL you know is wrong, e.g. `https://example.com/nope`.)

---

## Part B — Make requests yourself with `curl` (be the client)

**B1.** In your terminal, run:
```bash
curl -v https://example.com
```
- Lines starting with `>` are the **request** you sent. Lines starting with `<` are the **response** you got back.
- Write down: (a) the **request line** (the `>` line with `GET`), (b) the **status line** (the `<` line with `HTTP/... 200`), and (c) any two **response headers**.

**B2.** Run:
```bash
curl -I https://example.com
```
(`-I` fetches **headers only**, no body.)
- Which HTTP **method** does `-I` use? (Look it up if unsure — it's not GET.) Write down what that method is for.

**B3.** Ask for something that doesn't exist:
```bash
curl -i https://example.com/this-page-does-not-exist
```
- Write down the **status code** you got and what it means.

**B4.** (If your BayWork app runs) Start it (`docker compose up`) and hit your own server:
```bash
curl -v http://localhost:3001/api/health
```
- What is the **`Content-Type`** of the response? What does the body look like?
- Explain what `localhost` means here and why nothing left your machine.

---

## Part C — Written / from-memory exercises (close the notes first)

**C1.** On paper, **draw the request–response cycle** (the 8 steps) from memory. Then open the notes and check what you missed.

**C2.** Write your own **one-sentence definition of a server** — without looking. Then compare to §2. Rewrite it until it's tight.

**C3.** List **everything that happens** from pressing Enter on a URL to seeing the page, at this topic's level (client asks / server answers). Number the steps.

**C4.** Fill in the blanks from memory (then check):
- The client always speaks ______.
- IP address finds the ______; port finds the ______.
- HTTP is ______, so the server remembers nothing between requests by default.
- The response header that decides whether bytes are rendered, parsed, or drawn is ______.
- Frontend and backend can only communicate via ______ and ______.

**C5.** For each item, label it **CLIENT** or **SERVER** (or *both, depending on interaction*):
1. Chrome browser
2. Your Express API when the browser calls it
3. Your Express API when it queries Postgres
4. `curl`
5. PostgreSQL
6. A mobile app calling your API

---

## Part D — Teach-back (the real test)

**D1.** Set a 5-minute timer. Out loud (or to a friend / rubber duck), explain **how the web works** — cover: what a server is, the client–server model, the request–response cycle, and statelessness. No notes.

**D2.** Have someone (or your future self) ask you the six **interview soundbites** from §13. Answer each cold.

> You are done with this topic when D1 and D2 feel easy. If you stumble, note *which* part, re-read that section, and repeat tomorrow.

---

## Answers to the closed-book blanks (C4) — check only after trying

<details>
<summary>Reveal C4 answers</summary>

- The client always speaks **first**.
- IP address finds the **machine**; port finds the **program**.
- HTTP is **stateless**, so the server remembers nothing between requests by default.
- The response header that decides how bytes are treated is **`Content-Type`**.
- Frontend and backend can only communicate via **requests** and **responses** (HTTP messages).

</details>

<details>
<summary>Reveal C5 answers</summary>

1. Chrome — **client**
2. Express API answering the browser — **server**
3. Express API querying Postgres — **client** (it initiates the request to the DB)
4. `curl` — **client**
5. PostgreSQL — **server**
6. Mobile app calling your API — **client**

(Note #2 and #3: the *same* Express program is a server in one interaction and a client in another — proving "client/server are roles, not machines.")

</details>
