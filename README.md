<p align="center">
  <img src="https://readme-typing-svg.demolab.com?font=Fira+Code&weight=600&size=28&duration=3000&pause=800&color=3B82F6&center=true&vCenter=true&width=600&lines=%E2%94%8C%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%90;%E2%94%82++WebVerse+Pro+Labs++%E2%94%82;%E2%94%94%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%80%E2%94%98" alt="WebVerse Pro Labs" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-WebVerse%20Pro-3B82F6?style=for-the-badge&logo=google-cloud&logoColor=white" alt="Platform" />
  <img src="https://img.shields.io/badge/Focus-Web%20Security-EC4899?style=for-the-badge&logo=owasp&logoColor=white" alt="Focus" />
  <img src="https://img.shields.io/badge/Machines-Pwned-10B981?style=for-the-badge&logo=hack-the-box&logoColor=white" alt="Machines" />
</p>

<br />

<p align="center">
  <i>⚔️ &nbsp; No spoilers. No flags. Just the craft. &nbsp; ⚔️</i>
</p>

---

## 🧭 What Is This?

A growing archive of **technical writeups** for machines on
[**WebVerse Pro Labs**](https://dashboard.webverselabs-pro.com) —
a hands-on platform for web security practitioners.

Every guide here follows one rule:

> **Cover the technique, not just the answer.**

You'll find the methodology, the dead ends, the "why," and the
remediation — never a copy-paste flag. If you want the win,
put in the work. These notes are here to help you *learn*, not
to help you skip.

---

## ⚡ The Philosophy

<table>
<tr>
<td width="50%" valign="top">

### 🔍 What You'll Find Here

- Step-by-step **attack chains**, from recon to root
- **Diagrams** explaining the architecture before the exploit
- **Raw payloads** with byte-level walkthroughs
- **Remediation** advice — what the box got wrong, and how to fix it
- Links to **RFCs, CVEs, and research** that back the techniques

</td>
<td width="50%" valign="top">

### 🚫 What You Won't Find

- Literal flags or credentials
- "Just run this script" walkthroughs
- Spoilers that rob you of the *aha* moment
- Answers without explanation

</td>
</tr>
</table>

---

## 🗺️ Machine Index

<p align="center">
  <i>Each machine teaches a technique. Here's the syllabus.</i>
</p>

### 📡 Spread <kbd>Medium</kbd>

> *"A weekend CSV tool whose ops console is blocked at an edge proxy — but the app trusts the network."*

| | |
|---|---|
| **Technique** | CL.TE HTTP Request Smuggling |
| **Concepts** | Proxy bypass, Content-Length vs. Transfer-Encoding desync, keep-alive connection poisoning |
| **Services** | Gateway (edge proxy) + Gunicorn/Flask |
| **Lesson** | Blocking a path at the proxy is not access control |
| **Writeup** | [`spread-writeup.md`](./spread-writeup.md) |
| **Lab** | [WebVerse Pro →](https://dashboard.webverselabs-pro.com/labs/spread) |

<br />

---

```
 ┌──────────────────────────────────────────────────────────┐
 │                    MORE COMING SOON                       │
 │  ▰▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱   │
 └──────────────────────────────────────────────────────────┘
```

---

## 🧰 The Toolkit

These are the tools, references, and mental models I lean on across machines.
They're worth having in your own bag.

<table>
<tr>
<td>

**🔬 Testing**
- [Burp Suite](https://portswigger.net/burp)
- [Caido](https://caido.io)
- [curl + raw sockets](https://curl.se)

**📡 Recon**
- [Nmap](https://nmap.org)
- [FFuF](https://github.com/ffuf/ffuf)
- [httpx](https://github.com/projectdiscovery/httpx)

</td>
<td>

**📚 Reference**
- [PortSwigger Research](https://portswigger.net/research)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [RFC 7230 — HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc7230)
- [HackTricks](https://book.hacktricks.xyz)

**🧠 Mental Models**
- Trust boundaries are where bugs live
- The proxy is not the app
- Desync thrives on ambiguity

</td>
</tr>
</table>

---

## 📂 Repository Map

```
WebVerse-Pro-/
│
├── README.md                  ← you are here
│
├── spread/
│   ├── spread-writeup.md      ← CL.TE smuggling walkthrough
│   └── assets/                ← diagrams, screenshots
│
├── <next-box>/
│   └── ...
│
└── templates/
    └── writeup-template.md    ← blank guide for new machines
```

---

## ⚠️ Disclaimer

These writeups are for **educational purposes only**. Every technique
described here was tested in an isolated lab environment explicitly
designed for security training.

- Don't test these techniques against systems you don't own or have
  explicit permission to test.
- WebVerse Pro Labs provides the infrastructure — use it.
- Understanding how attacks work makes you a better defender.
  Use that knowledge responsibly.

---

<p align="center">
  <br />
  <img src="https://img.shields.io/badge/made%20with-%E2%99%A5%20%2B%20raw%20sockets-red?style=flat-square" alt="made with" />
  <br /><br />
  <a href="https://dashboard.webverselabs-pro.com">
    <img src="https://img.shields.io/badge/WebVerse_Pro-Labs-3B82F6?style=for-the-badge&logo=google-cloud&logoColor=white" alt="WebVerse Pro" />
  </a>
  <br /><br />
  <sub>© 2026 · <a href="https://github.com/tonyb760">tonyb760</a> · Crafted with curiosity</sub>
</p>
