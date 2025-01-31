---
title: "Nugget: Generating Usable Self-Signed Certificates In & For The Modern World"
description: "This is a short piece on how to generate self-signed certificates in 2025 & beyond"
image: "https://raw.githubusercontent.com/TaskMasterErnest/smol/master/images/tskmstr.jpg"
date: 2025-01-31T12:00:00+00:00
draft: false
tags: ["kubernetes", "ssl/tls", "tips"]
categories: ["devops", "linux"]
---

### **Generating Self-Signed Certificates (Without Getting Fired)**  
Look, I get it. Certificates are a necessary evil, and now they’ve gone and changed the rules on us. Gone are the days of just slapping a **Common Name (CN)** on your certificate and calling it a day. Now you need **SANs (Subject Alternative Names)** because apparently, the internet decided CNs are as outdated as dial-up. *Sigh.*  

Let’s cut through the noise. Here’s how to generate a modern, SANs-compliant certificate without losing your sanity.  

---

### **What Even Are These Terms?**  
- **CN (Common Name)**:  
  The old way to specify a server’s hostname (e.g., `webhook-server.webhook.svc`). Now deprecated for hostname validation but still required in certificates because… *tradition?*  

- **SANs (Subject Alternative Names)**:  
  The new way to list valid DNS names or IPs for your certificate. If your certificate doesn’t include these, Kubernetes (or your browser) will throw a tantrum.  

- **Why Bother?**:  
  Modern systems like Kubernetes enforce SANs for TLS validation. No SANs = cryptic `x509` errors. Yes, it’s annoying. No, you can’t ignore it.  

---

### **The "New Way" to Generate Certificates**  
#### **1. Create a Config File (`csr.conf`)**  
This file defines the certificate’s rules. Think of it as a bureaucratic form to appease the TLS gods:  
```conf
[ req ]
default_bits = 2048       # Key size. 2048 is standard. Don’t argue.
prompt = no               # "Don’t ask me questions. Use the values below."
default_md = sha256       # Hashing algorithm. SHA-1 is dead. Move on.
distinguished_name = dn   # "Hey, look at the [dn] section for details."
req_extensions = req_ext  # "Also, check [req_ext] for SANs."

[ dn ]
CN = webhook-server.webhook.svc  # Still required, but ignored for hostname checks.
O = Kubernetes                    # Organization. Put your company here (or fake it).

[ req_ext ]
subjectAltName = @alt_names  # "SANs go here. This is the part that matters."

[ alt_names ]
DNS.1 = webhook-server.webhook.svc              # Your service’s DNS name.
DNS.2 = webhook-server.webhook.svc.cluster.local # Kubernetes’ internal FQDN. Add this.
```  

#### **2. Run This Command**  
```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -days 365 \
  -nodes \
  -config csr.conf \
  -extensions req_ext
```  

**Decoding the Flags**:  
- `-x509`: Output a self-signed certificate instead of a CSR.  
- `-nodes`: “No DES” – don’t encrypt the private key. You’ll need this for Kubernetes to read it.  
- `-extensions req_ext`: Use the SANs defined in the `[req_ext]` section.  

---

### **Why This Works**  
- **SANs Over CN**: Kubernetes (and modern TLS) checks the `alt_names` list, not the CN.  
- **Future-Proofing**: This certificate will actually work in 2025.  
