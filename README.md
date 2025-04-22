

# 🕵️‍♂️ Wireshark Packet Analysis Case Study  
**Simulating a Real-World SOC Investigation**

In this case study, I took on the role of a Tier 1 SOC Analyst investigating suspicious activity across multiple PCAPs using **Wireshark**. Each section mirrors real SOC tasks — identifying scanning behavior, credential abuse, unauthorized downloads, and file exfiltration. Below is a scenario-driven breakdown of each finding.

![image](https://github.com/user-attachments/assets/c3d254d9-8f1b-426c-b270-ee343ff43754)

Fig: PCAP file location 
---

## 🔍 PCAP 1 – Reconnaissance and FTP Enumeration

### 🔹 1. Host Discovery Scanning Detected  
**Scenario:**  
An internal IDS alert flagged unusual broadcast traffic. Upon opening the PCAP, I observed `192.168.56.1` performing an ARP sweep — sending requests across the subnet to identify live hosts. This pre-TCP technique is commonly used by attackers in the recon phase.  
**Protocol:** ARP  
**Malicious IP:** 192.168.56.1  

![image](https://github.com/user-attachments/assets/0f4c4f8e-4dc0-45d4-b810-920d9a6a7d63)

We can see in the info column that 192.168.56.1 is sending multiple ARP requests. The attacker is trying to find which IPs are live in the local network.  
> “Are any of these IPs active on the network?”  
That’s host discovery scanning — done before any TCP connection is attempted.

---

### 🔹 2. Port Scanning Identified  
**Scenario:**  
After host discovery, the attacker proceeds to scan `192.168.56.111` for open ports. Using `Statistics > Conversations`, I identified short two-packet TCP attempts — a classic sign of a stealthy port scan.

Opening the first PCAP, we immediately see ARP traffic. Based on the ‘Info’ column, 192.168.56.1 is broadcasting to every IP in the subnet (192.168.56.2 to 192.168.56.254), asking for their MAC addresses.

![image](https://github.com/user-attachments/assets/a2d58a6a-a98b-4b81-9a44-f082c8171bab)

![image](https://github.com/user-attachments/assets/ad66dae4-155c-4fcb-bd0e-60b77b653fe5)

We see the source IP `192.168.56.1` is connecting to random destination ports on destination IP `192.168.56.111`, where each of these conversations is two packets long — clear evidence of port scanning.

---

### 🔹 3. FTP Server Inspection – User Limit  
**Scenario:**  
While reviewing FTP packets, I filtered on the `ftp` protocol and found server responses showing user session limits. This could indicate whether brute-force or concurrent login attempts are possible.

![image](https://github.com/user-attachments/assets/dff8798b-d801-488f-8062-30d861903625)

Filtering on FTP traffic using the filter `ftp`, I found response packets mentioning how many users can have active sessions at once.

---

### 🔹 4. Failed Anonymous Login  
**Scenario:**  
The attacker attempts to log in using the username `anonymous`. Following the TCP stream reveals the incorrect password used was `IEUser@`. This points to enumeration or credential guessing activity.

![image](https://github.com/user-attachments/assets/d0f5d1c4-2281-4ba2-b5f2-f3cf21bf2927)

Right-click the highlighted request and follow > TCP stream.

![image](https://github.com/user-attachments/assets/86228757-890d-4df2-81aa-2ff6d44a60c9)

The stream shows that the client used username `anonymous` and password `IEUser@`, which resulted in a failed authentication.

---

### 🔹 5. Web Server Fingerprinting via robots.txt  
**Scenario:**  
In packet 4612, a 404 error page was triggered. I exported the `robots.txt` file via HTTP objects and identified the web server as **Apache/2.4.38**, revealing valuable service version info.

![image](https://github.com/user-attachments/assets/250965ee-9843-47ab-abc4-12b0d73fde61)

![image](https://github.com/user-attachments/assets/f0179de7-e53a-4b89-abeb-d9f76cb3f1e9)

![image](https://github.com/user-attachments/assets/f61f8f86-7b3e-4c59-93c0-d0eca07709d0)

The system `192.168.56.111` sends a 404 "page not found" to `192.168.56.1`.  
Using **File > Export Objects > HTTP**, I extracted the HTML 404 page.

![image](https://github.com/user-attachments/assets/bfe09e6e-99f2-4551-8cce-6573a49333b7)

Opening it reveals the web server is running **Apache/2.4.38**.

---

## 📦 PCAP 2 – Malicious File Transfer

### 🔹 6. ZIP File Download Detected  
**Scenario:**  
An alert flagged outbound traffic to a ZIP file. Using **File > Export Objects > HTTP**, I confirmed that `192.168.56.1` downloaded a ZIP file served by `192.168.56.111`.  
TCP stream confirmed the full `GET` / `200 OK` transaction.

Method 1: 
![image](https://github.com/user-attachments/assets/7414dd21-2b66-4ea0-aad2-0b2d25e37280)

...

Method 2:  
HTTP is likely, so go to **File > Export Objects > HTTP**. The ZIP appears in the list as `application/zip`.

![image](https://github.com/user-attachments/assets/8319b7e3-c19d-4a7c-8223-f0e274a94f19)

![image](https://github.com/user-attachments/assets/ca6e6427-1070-4ef6-a4f1-dc2d7ce208c2)

Clicking the row brings us to the correct packet. TCP stream confirms the red `GET` from `192.168.56.1` and blue `200 OK` from `192.168.56.111`.

![image](https://github.com/user-attachments/assets/97bb6bdb-e663-4c9a-9218-58ae0472e989)

![image](https://github.com/user-attachments/assets/91e6ddc1-704d-409f-991f-9a0183d726af)

---

### 🔹 7. Port Analysis of File Transfer  
**Scenario:**  
To complete the investigation, I identified source and destination ports during the ZIP file transfer.

Continuing from the last case, I selected the `200 OK` response packet or re-exported the HTTP ZIP entry to identify the relevant ports.

![image](https://github.com/user-attachments/assets/47cd3d1c-ed08-4a3a-b620-36339dcd4913)


---

## 🔐 PCAP 3 – FTP Brute Force and Exfiltration

### 🔹 8. FTP Server Identified  
**Scenario:**  
Suspicious login attempts triggered a brute-force alert. Filtering by `ftp`, I traced `220 FTP` welcome responses and confirmed the server IP as `192.168.56.118`.


![image](https://github.com/user-attachments/assets/a72e7314-56e2-4b45-99b1-e08b894fde6b)


![image](https://github.com/user-attachments/assets/4dcdfb80-a744-401b-97a4-809cc0319864)

Packets with `Response: 220` indicate the FTP server. The source IP is `192.168.56.118`.

---

### 🔹 9. First Brute-Force Password Attempt  
**Scenario:**  
Using the `Time` column and `PASS` command, I noted the timestamp of the first password attempt.  
I changed time format via **View > Time Display Format > Date and Time of Day**.


![image](https://github.com/user-attachments/assets/01dc3c6c-9798-41f1-8bcd-632d3f3c0e72)

---

### 🔹 10. Successful FTP Login  
**Scenario:**  
A `230 Login Successful` packet confirmed access. This marked a compromise, and I immediately traced back to the credentials used.

Time format set to `YYYY-MM-DD HH:MM:SS`.


![image](https://github.com/user-attachments/assets/45fb767c-8721-467f-a61d-0b9c7e3011f4)

---

### 🔹 11. Credentials Used in Breach  
**Scenario:**  
Following the TCP stream before the `230` response revealed the attacker used:  
- **Username:** Chris  
- **Password:** naruto


![image](https://github.com/user-attachments/assets/d219af9c-7006-43c2-a559-179fa0e37b39)

---

### 🔹 12. File Name Exfiltrated via FTP  
**Scenario:**  
The attacker issued a `RETR` command after login.  
The file `password.backup` was downloaded — likely a sensitive dump.


![image](https://github.com/user-attachments/assets/5ce98f13-e0e1-4e51-b53d-0b9a44ecc3fd)

---

### 🔹 13. File Download Timestamp  
**Scenario:**  
I correlated the `RETR` command packet to identify the exact timestamp when the attacker started exfiltrating the file.

Use TCP stream > click `RETR` > jump to packet > note timestamp.

![image](https://github.com/user-attachments/assets/7cfcde2d-d2ce-4b0d-86c4-4f5822e7aaa0)

---

## 🌱 Key Takeaways

- ✅ Developed deep familiarity with Wireshark for real-case SOC scenarios  
- ✅ Strengthened ability to reconstruct attacker timelines from raw traffic  
- ✅ Practiced evidence extraction, credential identification, and protocol analysis  
- ✅ Reinforced understanding of attacker behavior — from scanning to exploitation  

---

> **Security is about protecting people, not just systems.**  
> Every packet tells a story — this was my journey putting those clues together to uncover a breach.
