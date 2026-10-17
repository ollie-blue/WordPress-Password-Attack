# Project: WordPress Password Attack & Rate-Limit Bypass

## 📋 Executive Summary
Executed a targeted brute-force password attack against a local WordPress environment (`http://localhost/cybersec/wp-login.php`) to compromise the `admin` account. By analyzing WordPress's default security lockout mechanisms (3 failed attempts triggering a 10-minute ban), a **Pitchfork attack with interleaved request pairs** was implemented using Burp Suite Intruder. Interleaving legitimate authentication requests (`think`/`123`) between target brute-force attempts successfully bypassed rate-limiting, discovering the administrator password (`#1mama`).

---

## 🛠️ Architecture & Tech Stack
* **Target Application:** WordPress (Hosted locally via XAMPP)
* **Local Web Server:** Apache & MySQL (XAMPP Control Panel)
* **Traffic Interception & Proxy:** FoxyProxy, Burp Suite Community Edition (Proxy, Intruder)
* **Scripting & Text Processing:** VS Code, Python / REGEX parameter expansion
* **Target Endpoints:** `/wp-login.php`, `/wp-admin/profile.php`

---

## ⚙️ Step-by-Step Implementation

### Step 1: Environment Setup & Reconnaissance
* Started **Apache** and **MySQL** services via the XAMPP Control Panel and enabled **FoxyProxy** in the browser.
* Identified two target usernames on the web application:
  1. `think` (Known valid credentials: `think` / `123`)
  2. `admin` (Target account with unknown password)
* Established operational constraints: WordPress blocks login attempts for 10 minutes after 3 consecutive failed attempts.

---

### Step 2: Wordlist & Payload Generation
To bypass the 3-attempt lockout threshold, a wordlist was structured to alternate between a known valid login (`think`/`123`) and an experimental attempt (`admin`/`candidate_password`).

1. **User Payload (`user.txt`):**
   * Generated an alternating list containing 110 pairs of `think` and `admin` (220 total lines).
2. **Password Payload (`pass.txt`):**
   * Extracted 110 candidate passwords from `Hint_passwords.txt`.
   * Formatted the list in VS Code using search-and-replace (`\n` $\rightarrow$ `\n123\n`) and manually prepended `123` to align with the first `think` username payload.

```bash
# Example file structure alignment:
# user.txt           pass.txt
think        -->     123       (Valid login reset)
admin        -->     111       (Attempt 1)
think        -->     123       (Valid login reset)
admin        -->     444       (Attempt 2)
```

---

### Step 3: Traffic Interception & Intruder Configuration
1. Captured a baseline authentication POST request to `/wp-login.php` using Burp Suite Proxy.
2. Forwarded the request to **Burp Suite Intruder** and set up the attack parameters:
   * **Attack Type:** `Pitchfork`
   * **Payload Position 1:** Username parameter (`log`) $\rightarrow$ mapped to `user.txt` (220 items)
   * **Payload Position 2:** Password parameter (`pwd`) $\rightarrow$ mapped to `pass.txt` (220 items)

---

### Step 4: Bypassing Lockout & Response Analysis
1. **Grep Match Configuration:**
   * Configured response flag keywords in Intruder Settings (`Error`, `incorrect`, `password`, `Lost`, `authentication`, `limit`) to isolate failed login attempts.
2. **Resource Pool Optimization:**
   * Created a custom Resource Pool (`attack1`) and set **Maximum Concurrent Requests** to `1`.
   * Single-threaded execution ensured strict sequential order, keeping the failed attempt counter reset by the alternating `think`/`123` requests.
3. **Execution & Discovery:**
   * Launched the attack. Filtered results by `Status Code (302 Redirect)` and payload response signatures.
   * **Result:** Request #142 yielded an `HTTP 302` response without any error flag signatures for username `admin` with password `#1mama`.

---

## 🧪 Testing & Validation
* Navigated to `http://localhost/cybersec/wp-login.php` and manually submitted the discovered credentials (`admin` / `#1mama`).
* Successfully bypassed the login screen, passed the Administration Email Verification step, and gained administrative control over the WordPress dashboard (`/wp-admin/profile.php`).

---

## 💡 Key Learnings & Challenges
* **Challenge:** Overcoming WordPress default brute-force defenses without modifying system databases or server configuration files.
* **Solution:** Interleaving known valid credentials into a single-threaded Pitchfork attack stream to continuously clear the consecutive failure counter.
* **Key Takeaway:** Application-level rate limits that count *consecutive* failures can be effectively neutralized if an attacker possesses one set of low-privilege valid credentials.

---

## 🚀 Future Improvements
* Implement IP-based and account-based lockout policies that track global failure counts rather than purely consecutive failures.
* Enforce Multi-Factor Authentication (MFA) and CAPTCHA validation on all administrative login endpoints.