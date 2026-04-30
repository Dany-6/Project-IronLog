# Phalanx Check — Social Media & Launch Materials

## 1. LinkedIn Post

**Objective:** Professional announcement highlighting the problem solved, the technical architecture, and encouraging engagement.

---

🔒 **Excited to announce the release of Phalanx Check!** 🛡️

As cybersecurity professionals, we know that phishing remains the #1 initial access vector for breaches. But conducting internal phishing simulations often requires expensive enterprise platforms or risky, weaponized red-team tools that lack safety guardrails. 

That's why I built **Phalanx Check** — an open-source, Python-based phishing simulation framework designed specifically with safety and authorization in mind.

I wanted a tool that allows security teams to test user awareness effectively without the risk of accidental malicious use. 

🛠️ **Key Features:**
- **Strict Domain Whitelisting:** The tool absolutely refuses to send emails to domains not explicitly pre-approved by the administrator.
- **Embedded Watermarking:** Every simulation email contains cryptographic watermarks, making it instantly verifiable by incident response teams.
- **Rate Limiting:** Built-in throttles prevent mail server blacklisting and accidental spam storms.
- **CSV Target Management:** Easily manage authorized target lists for structured campaigns.

If you are a security educator, a blue teamer, or a sysadmin looking to baseline your organization's phishing resilience safely, check it out!

💻 **GitHub Repo:** [Insert GitHub Link Here]

I'd love to hear your feedback, feature requests, or contributions! Let's make security awareness testing safer and more accessible.

#CyberSecurity #Python #OpenSource #PhishingSimulation #BlueTeam #InfoSec #SecurityAwareness

---

## 2. Reddit Posts

**Objective:** Technical, community-focused posts tailored for different subreddits.

### Option A (For r/Python and r/opensource)
**Title:** I built an open-source Python tool for safe phishing simulations (with strict safety guardrails)

**Body:**
Hey r/Python,

I recently wanted to run some phishing awareness tests, but I found that most open-source tools out there are either too complex (geared towards advanced red teaming) or lack built-in safety features, making it easy to accidentally blast the wrong targets.

So, I built **Phalanx Check** entirely in Python. 

It’s a modular phishing simulation framework that enforces safety first. The core feature is a strict domain whitelisting engine — if the target's email domain isn't in your `authorized_domains.json`, the Python script hard-halts. It also injects verifiable watermarks into the email headers so the Blue Team can always distinguish a simulation from a real attack.

I used standard libraries where possible and focused on keeping the architecture modular so people can add their own email templates or sending backends. 

Repo here: [Insert GitHub Link]

Would love for some Python devs to roast my code or suggest architectural improvements!

### Option B (For r/cybersecurity, r/netsec, and r/HowToHack)
**Title:** Release: Phalanx Check - An authorized, safety-first phishing simulation tool

**Body:**
Hey everyone, 

I'm releasing a project I've been working on: **Phalanx Check**. 

A major issue with running internal phishing campaigns is the risk of the tool being misused or simulations accidentally leaking outside the scope of engagement. Phalanx Check is designed to solve this by enforcing guardrails at the code level.

**Features:**
*   **Zero-Trust Targeting:** Enforces strict domain whitelisting. It drops any target not matching the authorized domain list.
*   **IR Watermarking:** Injects `X-Phalanx-Simulation: True` and cryptographic hashes into the email headers to prevent false-positive incident response panics.
*   **Rate Limiting & Jitter:** Prevents overwhelming target mail servers.
*   **Modular architecture:** Easy to plug in custom HTML templates and SMTP relays.

It’s open source and meant to be a lightweight alternative to heavy commercial platforms for organizations wanting to run safe, verifiable awareness tests.

Check out the source code and documentation here: [Insert GitHub Link]

Feedback and PRs are highly welcome.

---

## 3. Dev.to / Medium Article

**Title:** Building Phalanx Check: Why We Need Safer Phishing Simulation Tools

**Subtitle:** How I used Python to build an open-source, guardrailed framework for security awareness testing.

**Article Body:**

Phishing is responsible for the vast majority of data breaches today. To combat this, organizations run phishing simulations to train their employees. But there is a glaring gap in the tooling ecosystem: most open-source tools are built for offensive red-team operations. They are powerful, but they lack the intrinsic safety guardrails needed for routine, administrative awareness testing. 

A simple configuration mistake in a powerful red-team tool can result in phishing emails being sent to unauthorized external domains, potentially causing legal issues or damaging reputation.

I realized the community needed a tool built from the ground up for *safety*. That’s why I created **Phalanx Check**.

### The Core Philosophy: Guardrails by Default
When building Phalanx Check in Python, the primary design constraint was ensuring the tool could not be accidentally weaponized against unauthorized targets. 

Here is how I implemented the safety features:

**1. Strict Domain Whitelisting**
Before the SMTP engine even initializes, Phalanx Check parses the target CSV list and cross-references every single email address against an `authorized_domains.txt` file. If a target domain is not explicitly whitelisted, the script halts and throws an exception. This makes it impossible to accidentally email a vendor or a personal Gmail account.

**2. Incident Response (IR) Watermarking**
There is nothing worse for a Blue Team than wasting hours triaging a reported phishing email, only to realize it was an internal simulation. Phalanx Check automatically injects specific, configurable SMTP headers (like `X-Phalanx-Simulation`) and subtle body watermarks. This allows SOC analysts to quickly verify the origin of the email.

**3. Controlled Rate Limiting**
Sending thousands of emails at once will rapidly get your simulation infrastructure blacklisted by secure email gateways (SEGs). I built a rate-limiting and jitter engine that randomizes the send intervals, mimicking human behavior and ensuring the mail server isn't overwhelmed.

### The Technical Stack
The tool is built in pure Python to ensure it is lightweight and easily deployable across Windows, macOS, and Linux. It uses a modular architecture:
*   **Core Engine:** Handles the logic, safety checks, and CSV parsing.
*   **Template Engine:** Allows users to load custom HTML phishing templates.
*   **SMTP Dispatcher:** Securely connects to mail relays via TLS.

### Open Source and Community
Security is a collaborative effort. I have made Phalanx Check completely open-source. Whether you are a system administrator looking to test your company's resilience, or a developer wanting to contribute to open-source security tools, I invite you to check it out.

You can view the source code, read the documentation, and start running your own safe simulations here:

🔗 **[GitHub Repository Link]**

Let’s make security awareness testing safer, more verifiable, and accessible to everyone.
