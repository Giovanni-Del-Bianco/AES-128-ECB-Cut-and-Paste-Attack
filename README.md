
# Cryptographic Cut-and-Paste: Bypassing AES-128 ECB Integrity

**University Project | Fondations Of Cybersecurity Laboratory | MSc Cybersecurity**

A technical analysis and exploitation of AES-128 in Electronic Codebook (ECB) mode. This project demonstrates how the lack of integrity checks in block ciphers allows for surgical ciphertext manipulation (Cut-and-Paste attack) to hijack financial transactions.

## 📌 Info Details

| Info | Details |
| :--- | :--- |
| 👤 **Author** | Giovanni Del Bianco |
| 💻 **Language** | C++ (OpenSSL API) |
| 🛠️ **Tools** | G++, OpenSSL, DD, Cat, Linux Terminal |
| 🧠 **Key Concepts** | Block Cipher Cryptanalysis, ECB Mode Vulnerabilities, Ciphertext Manipulation, PKCS#7 Padding Integrity. |
| 🎯 **Goal** | Manipulate a captured encrypted transaction to forge a new, valid ciphertext that modifies the payment amount and transaction ID. |

---

## 🚨 The Scenario: Mission Briefing

### 1. The Target Environment
The University of Pisa is in debt, and a user named "Alice" is attempting to settle her balance. However, the captured transaction shows she is paying far less than she owes. Our objective is to intercept and modify her encrypted payment to the University's benefit.

The system provides three main components:
*   **`bt_enc`**: The executable used by users (like Alice) to encrypt transactions.
*   **`bt_dec`**: The tool used by the bank to decrypt and read the transaction details.
*   **`verifier`**: The automated bank system that validates the final forged transaction.

### 2. Technical Specifications
*   **Encryption Algorithm:** AES-128.
*   **Cipher Mode:** ECB (Electronic Codebook).
*   **Message Structure:** The transaction string is constructed as follows:
    > `[Payer] transfers [Amount]$ to: [Beneficiary]. Transfer# [Transaction_ID].`


## 🛠️ Laboratory Setup & Compilation

To analyze and exploit the system, we must first compile the source code provided in the `src/` directory. The programs require the `libssl-dev` package to interface with OpenSSL APIs.

### Compilation Commands
Based on the repository structure, execute the following commands to generate the binaries:

```bash
# Compile the Encryption tool
g++ -o bt_enc src/bt_enc.cpp -lcrypto

# Compile the Decryption tool
g++ -o bt_dec src/bt_dec.cpp -lcrypto

# Compile the Bank Verifier
g++ -o verifier src/verifier.cpp -lcrypto
```

*Note: The `-lcrypto` flag is mandatory to link the OpenSSL cryptographic libraries.*


## 🧠 The Vulnerability: Why ECB is Broken by Design

The fundamental flaw in this system is the choice of AES-128 in Electronic Codebook (ECB) mode. In ECB mode, the plaintext is divided into independent 16-byte blocks, and each block is encrypted separately using the same key.

**Key Weaknesses:**
*   **Block Independence:** There is no "chaining" or dependency between blocks. Encrypting Block A does not affect how Block B is encrypted.
*   **Determinism:** Identical plaintext blocks will always result in identical ciphertext blocks as long as the key remains the same.
*   **Lack of Integrity:** While the data is confidential (hidden), its integrity is not protected. An attacker can rearrange, delete, or replace blocks like LEGO bricks without the decryption process detecting the interference.



## 🕵️‍♂️ Black-Box Analysis: Mapping the Ciphertext

To perform a successful "Cut-and-Paste" attack, we first needed to understand the internal structure of the encrypted message. By using `bt_enc` as an "oracle"—performing several encryption tests with known inputs—the following memory map was deduced.

### 1. The Probing Phase (Differential Analysis)
By observing how the ciphertext changed based on input variations, the following patterns emerged:
*   **Payer Identification:** Changing the Payer name from "A" to "Alice" completely altered the first 16 bytes, but left subsequent blocks unchanged if other inputs remained constant.
*   **Fixed Field Lengths:** The program forces the amount and `t_num` fields to exactly 16 characters by padding them with leading zeros.
*   **Block Alignment:** Because the string `" transfers "` is exactly 11 characters, a 5-character nickname like `"Alice"` perfectly fills the first 16-byte block (5 + 11 = 16).

### 2. The Logical Block Map
Based on the analysis of the captured `transaction.enc` file (96 bytes total, representing 6 blocks), we mapped the data as follows:

| Block | Byte Range | Deduced Content (Plaintext) | Role in Attack |
| :---: | :---: | :--- | :--- |
| **B1** | `0 - 15` | `Alice transfers ` | Fixed Payer Identity |
| **B2** | `16 - 31` | `0000000000100000` | Original Amount ($100k) |
| **B3** | `32 - 47` | `$ to: PisaUniver` | Beneficiary (Part 1) |
| **B4** | `48 - 63` | `sity. Transfer# ` | Beneficiary (Part 2) |
| **B5** | `64 - 79` | `0000001352782121` | Original Transaction ID |
| **B6** | `80 - 95` | `[PKCS#7 Padding]` | Cryptographic Integrity |



## 🏗️ The Engineering Approach: Surgical Block Swap

The objective set by the `verifier` is to make Alice pay a much higher amount. 

The **"Aha!" moment** came when realizing that the Transaction ID (B5) was a large random number, while the Amount (B2) was the small $100k payment.

> **The Strategy:**  
> By swapping **Block 2** with **Block 5**, we trick the bank's software into interpreting the large random ID as the payment amount, and the original $100k as the new transaction ID.



## 🚀 Execution: Step-by-Step Forgery

To execute the "Cut-and-Paste" attack, I used the `dd` utility to surgically extract the 16-byte blocks from the original `captured/transaction.enc` file. The process required extreme precision to ensure every byte was perfectly aligned with the AES block boundaries.

### Step 1: Block Extraction (The "Cut")
Each command extracts exactly one 16-byte block from the source file by skipping the preceding data.

```bash
# Extract Block 1: "Alice transfers "
dd if=captured/transaction.enc of=b1 bs=16 count=1 skip=0

# Extract Block 2: Original $100k Amount
dd if=captured/transaction.enc of=b2 bs=16 count=1 skip=1

# Extract Block 3: "$ to: PisaUniver"
dd if=captured/transaction.enc of=b3 bs=16 count=1 skip=2

# Extract Block 4: "sity. Transfer# "
dd if=captured/transaction.enc of=b4 bs=16 count=1 skip=3

# Extract Block 5: The Large Random ID (Our new Amount)
dd if=captured/transaction.enc of=b5 bs=16 count=1 skip=4

# Extract Block 6: The Original Cryptographic Padding
dd if=captured/transaction.enc of=b6 bs=16 count=1 skip=5
```

### Step 2: The Padding Challenge (Troubleshooting)
During the initial attempt, simply swapping the data blocks caused a `bad decrypt` error in the verifier.

*   **⚠️ The Problem:** OpenSSL's `EVP_DecryptFinal` function checks the final block for valid PKCS#7 padding. If the file ends abruptly or with a data block (like B2), the padding check fails, causing a core dump.
*   **✅ The Solution:** I realized that the padding is mathematically tied to the secret key. To bypass this, I extracted the original padding block (**B6**) from Alice's transaction and appended it to the end of my forged file, ensuring a "clean" decryption.

### Step 3: Reassembly (The "Paste")
The blocks were combined in the specific order required to trick the bank's logic: *Payer + New Amount + Beneficiary + Old Amount (as ID) + Padding*.

```bash
cat b1 b5 b3 b4 b2 b6 > forged.enc
```



## 🏆 Results & Proof of Concept

Feeding the `forged.enc` file into the `verifier` confirmed the successful hijack of the execution flow and the modification of the financial record.

```text
$ ./verifier
Please, type the file to decrypt: forged.enc

Transaction Accepted.
{ECB_is_n0t_r3si1i3nt_t0_r30rd3r}
Decrypted text is:
Alice transfers 0000001352782121$ to: PisaUniversity. Transfer# 0000000000100000
```

The system successfully decrypted the file because all blocks were originally encrypted with Alice's key. However, it interpreted the data in our new, malicious order.



## 🎓 Key Takeaways & Security Hardening

This laboratory provided a deep understanding of why **Confidentiality does not equal Integrity**.

*   **Cryptographic Lesson:** AES-ECB is a "stateless" mode. It hides the content but fails to protect the structure of the message.
*   **Engineering Insight:** Debugging the `bad decrypt` error reinforced the importance of understanding the underlying mechanics of cryptographic padding (PKCS#7).
*   **Defense Strategy:** To prevent this attack, the system should implement:
    *   **Authenticated Encryption (AEAD):** Using modes like AES-GCM which include a "Tag" to verify that the ciphertext hasn't been modified.
    *   **HMAC (Hash-based Message Authentication Code):** Appending a signature to the file to ensure the sender and the message order are authentic.

## 📥 Clone & Reproduce

Want to try this attack yourself? You can easily clone the repository and recreate the exploit in your local environment.

```bash
# Clone the repository
git clone https://github.com/Giovanni-Del-Bianco/AES-128-ECB-Cut-and-Paste-Attack.git

# Navigate into the project directory
cd AES-128-ECB-Cut-and-Paste-Attack
```

Once cloned, simply follow the compilation commands and the step-by-step execution guide detailed above in this README to manually perform the Cut-and-Paste attack.



## 📜 License & Disclaimer

This project is released under the MIT License.

> **Disclaimer:** This repository is for educational purposes only. These techniques were demonstrated within a controlled university laboratory environment to understand and improve cryptographic security.