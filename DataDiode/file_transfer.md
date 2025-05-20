In this data diode setup, the communication is strictly unidirectional (sender → receiver), and here's exactly how data flows through the system:

### **1. Input to Sender Container**
The sender container receives input through its proxy service (port 8080). There are multiple ways to feed data:

#### **Method A: Direct HTTP POST (Recommended for Testing)**
```bash
# From your HOST machine or any external system that can reach the sender's IP:
curl -X POST -d "Your message here" http://10.0.1.2:8080
```
- This sends raw data to the sender's proxy, which forwards it to the receiver.

#### **Method B: File Upload via HTTP**
```bash
# Send a file's content to the sender:
curl -X POST --data-binary "@/path/to/your/file.txt" http://10.0.1.2:8080
```
- The sender's `socat` proxy forwards the file content to the receiver.

#### **Method C: Automated Input (For Continuous Data)**
You could configure a service inside the sender container to:
- Watch a directory (`inotifywait`)
- Read from a serial device (if simulating hardware)
- Accept data from a network socket (if allowed by firewall)

---

### **2. How the Receiver Processes & Stores Data**
The receiver's proxy (`socat` on port 8081) is configured to:
1. **Accept incoming data** from the sender.
2. **Insert it into PostgreSQL** automatically:
   ```sql
   INSERT INTO data (content) VALUES ('[received data]');
   ```

#### **Viewing Received Data**
To check stored data:
```bash
# Enter the receiver container:
sudo nixos-container root-login receiver

# Query the database:
psql -U diode diode_data -c "SELECT * FROM data;"
```
- This lists all received entries with timestamps.

---

### **3. Extracting Data from the Receiver (Securely)**
Since the diode is **one-way**, you **cannot** pull data back through the diode. Instead, you must:

#### **Option A: Direct Database Access (If Receiver is Exposed)**
```bash
# From a trusted machine that can reach the receiver:
psql -h 10.0.2.2 -U diode diode_data -c "SELECT * FROM data;"
```
*(Ensure firewall rules allow this only from trusted hosts.)*

#### **Option B: Manual Export (Air-Gapped Style)**
1. **Dump data to a file** inside the receiver:
   ```bash
   psql -U diode diode_data -c "COPY (SELECT * FROM data) TO '/tmp/export.csv' WITH CSV HEADER;"
   ```
2. **Transfer manually** (since no reverse connection is allowed):
   - Use `scp` (if SSH is enabled on a different port)
   - Physically copy (if simulating high-security air-gap)

---

### **4. Full Example: Sending a File & Retrieving It**
#### **Step 1: Send a File via Sender**
```bash
curl -X POST --data-binary "@/home/user/testfile.txt" http://10.0.1.2:8080
```

#### **Step 2: Verify in Receiver**
```bash
sudo nixos-container root-login receiver
psql -U diode diode_data -c "SELECT * FROM data;"
```
*(You should see the file content stored in the database.)*

#### **Step 3: Export Data (If Needed)**
```bash
psql -U diode diode_data -c "COPY (SELECT * FROM data) TO '/tmp/diode_export.csv' WITH CSV;"
```

---

### **Key Security Notes**
✅ **Strictly One-Way**:  
- No TCP/UDP packets can go **receiver → sender** (enforced by `nftables`).  
- Even if an attacker compromises the receiver, they **cannot** send data back.

⚠️ **Secure Extraction Required**:  
- Use **separate credentials** for database access.  
- If high security is needed, manually transfer data (USB/memory card).  

🔧 **For Production**:  
- Replace `socat` with a hardened proxy (like `HAProxy`).  
- Encrypt data before sending (TLS between containers).  
- Add checksum verification.  

Would you like modifications for a specific use case (e.g., large files, real-time streams)?