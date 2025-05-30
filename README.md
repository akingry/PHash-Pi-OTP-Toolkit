# PHash-Pi-OTP-Toolkit

**Short Description:** A client-side, single-HTML file tool for one-time pad (OTP) inspired encryption/decryption, deriving its cryptographic pad from a perceptual image hash (pHash), a user-defined passphrase, and an embedded segment of the digits of Pi.

---

## Overview

`PHash-Pi-OTP-Toolkit` is a self-contained HTML/JavaScript application that implements a symmetric encryption scheme reminiscent of a one-time pad. It uniquely generates a keystream by combining three user-influenced factors:

1.  A **perceptual hash (pHash)** derived from a user-uploaded image.
2.  A **secret passphrase** provided by the user.
3.  A deterministic selection of digits from an **embedded sequence of the mathematical constant Pi**.

The tool allows for both encryption of plaintext (UTF-8) to Base64 ciphertext and decryption of Base64 ciphertext back to plaintext. It operates entirely on the client-side, requiring no backend or external libraries beyond what's embedded in the single HTML file.

This project is primarily an exploration of cryptographic principles and unconventional key derivation methods. While it aims for a high degree of pseudo-randomness in its pad generation, it should be understood in the context of experimental cryptography rather than a replacement for standardized, audited encryption algorithms for highly sensitive data.

## Core Concepts

* **One-Time Pad (OTP) Principle:** The core encryption mechanism is a byte-wise XOR operation between the plaintext/ciphertext and a generated pad. If the pad were truly random and used only once, this would offer perfect secrecy. This tool *simulates* this by generating a pad that is highly dependent on unique inputs.

* **Perceptual Hashing (pHash):** Instead of a cryptographic hash of the image file itself (which would change with metadata modifications), a perceptual hash (specifically, an "average hash") is computed. This hash represents the visual features of the image, meaning visually similar images will produce similar (but not necessarily identical) hashes. The pHash provides a source of initial entropy derived from the image content.

* **Pi as a Pseudo-Random Source:** The digits of Pi are often considered a good source of pseudo-random digits. This tool embeds the first 30,000 digits of Pi (after the decimal) and uses a derived offset to select a segment of these digits for pad generation.

* **Key Derivation Function (KDF) - Custom:** The process of combining the pHash and passphrase to determine the starting offset in the Pi digits acts as a custom Key Derivation Function.

## Key Derivation & Pad Generation Process

The generation of the OTP pad is the crux of this tool's security model:

1.  **Image pHash Computation:**
    * The user uploads an image.
    * The image is drawn onto a 128x128 canvas.
    * An **Average Hash** algorithm is applied:
        * The 128x128 image is scaled down to an 8x8 pixel representation.
        * These 64 pixels are converted to grayscale.
        * The average grayscale value of these 64 pixels is calculated.
        * Each pixel's grayscale value is compared to the average: 1 if brighter or equal, 0 if darker.
        * This results in a 64-bit binary string, which is then converted to a 16-character hexadecimal string (e.g., `c3c3c3c3c3c3c3c3`). This is the `currentPHash`.

2.  **Passphrase Processing & Combination:**
    * The user provides a secret passphrase.
    * The passphrase is UTF-8 encoded into bytes (`passphraseBytes`).
    * The `pHashBytes` (8 bytes derived from `currentPHash`) are XORed with a "folded" version of the `passphraseBytes`. The folding process ensures that the entire passphrase contributes to an 8-byte intermediate key, which is then XORed with `pHashBytes`.
        * `tempPassphraseKey = new Uint8Array(pHashBytes.length)` (initialized to zeros)
        * For each byte in `passphraseBytes`: `tempPassphraseKey[i % pHashBytes.length] ^= passphraseBytes[i]`
        * `xorResultBytes[i] = pHashBytes[i] ^ tempPassphraseKey[i]`

3.  **Pi Offset Derivation:**
    * The `xorResultBytes` are converted to a hexadecimal string (`xorResultHex`).
    * The first 10 hexadecimal characters of `xorResultHex` are taken.
    * These 10 hex characters are parsed as a decimal integer (`decimalValue`).
    * The final `offset` into the Pi digits string is calculated as:
        `offset = decimalValue % (PI_DIGITS_30K.length - (messageByteLength * 3) + 1)`
        This ensures the offset is always valid for the given message length and the available Pi digits, preventing out-of-bounds reads.

4.  **OTP Pad Generation from Pi Digits:**
    * The tool uses an embedded string `PI_DIGITS_30K` containing the first 30,000 digits of Pi (after the decimal).
    * Starting from the calculated `offset`, a segment of Pi digits is extracted. The length of this segment is `messageByteLength * 3` (since 3 Pi digits are used to form each byte of the pad).
    * This segment of Pi digits is processed in 3-digit chunks.
    * Each 3-digit chunk is converted to an integer, and then `modulo 256` is applied to produce a byte value (0-255).
    * These bytes form the OTP pad.

## Encryption/Decryption

* **Encryption:**
    1.  Plaintext from the input textarea is UTF-8 encoded into bytes.
    2.  The OTP pad is generated as described above, matching the length of the plaintext byte array.
    3.  Each byte of the plaintext is XORed with the corresponding byte of the OTP pad.
    4.  The resulting byte array (ciphertext) is Base64 encoded and displayed.

* **Decryption:**
    1.  Base64 ciphertext is decoded into a byte array.
    2.  The OTP pad is regenerated using the *exact same image (pHash) and passphrase*. This is critical.
    3.  Each byte of the decoded ciphertext is XORed with the corresponding byte of the OTP pad.
    4.  The resulting byte array is UTF-8 decoded into plaintext and displayed.

## Security Considerations & Limitations

* **Client-Side Operation:** All cryptographic operations occur in the user's browser. No data (image, passphrase, plaintext, ciphertext) is transmitted to any server. This enhances privacy but also means the security relies on the integrity of the client's environment.
* **Importance of Full Pi String:** The tool is designed to use 30,000 embedded Pi digits. If this string is truncated, the effective keyspace for the Pi offset is drastically reduced, severely weakening the security and leading to potential offset collisions.
* **pHash Properties:** Perceptual hashes are designed for similarity detection, not cryptographic collision resistance. While distinct images will generally produce distinct pHashes, it's theoretically possible for two different images to yield the same pHash or pHashes that, when combined with a passphrase, lead to the same Pi offset for short messages.
* **Passphrase Strength:** The security heavily relies on the secrecy and complexity of the passphrase. The "folding" mechanism ensures the entire passphrase contributes, but a weak passphrase remains a weak link.
* **Key Uniqueness (Image + Passphrase):** The combination of the image's pHash and the full passphrase is what makes the derived Pi offset (and thus the OTP pad) unique.
* **Not a True OTP:** While inspired by OTPs, the pad is deterministically generated. If the same image and passphrase are used to encrypt different messages of the same length that result in the same initial `decimalValue` for the offset calculation (before the message-length dependent modulo), they will use the same keystream segment, violating OTP principles. The message-length dependent modulo in `deriveSeedOffset` mitigates this for many cases but doesn't eliminate all cryptographic weaknesses inherent in a deterministic pad.
* **Experimental Nature:** This tool is an educational and experimental implementation. For production-level security, use well-vetted, standardized cryptographic libraries and protocols.

## Technical Details

* **Language:** HTML, CSS, JavaScript (ES6+).
* **Single File:** Designed to run as a single `otp_tool.html` file.
* **pHash Implementation:** Embedded "Average Hash" algorithm.
* **Pi Source:** Embedded string of the first 30,000 digits of Pi.
* **Encoding:** UTF-8 for text-to-byte conversion, Base64 for ciphertext representation.
* **Dependencies:** None (fully self-contained).

## How to Use

1.  Open `otp_tool.html` in a modern web browser.
2.  **Upload Image:** Select an image file. Its pHash will be computed automatically. A preview is shown.
3.  **Enter Passphrase:** Type a secret passphrase. Use the "Show" toggle to verify.
4.  **Encryption:**
    * Enter plaintext in the "Plaintext" textarea.
    * Click "Encrypt".
    * The Base64 encoded ciphertext will appear in the "Ciphertext" textarea.
    * Log messages will detail the process.
5.  **Decryption:**
    * Ensure the **same image** is loaded and the **exact same passphrase** is entered.
    * Paste Base64 ciphertext into the "Ciphertext" textarea.
    * Click "Decrypt".
    * The recovered plaintext will appear in the "Plaintext" textarea.
6.  **Copy Buttons:** Use the "Copy" buttons under each textarea to copy their contents to the clipboard.
7.  **Log:** The log area on the side provides status updates and error messages.

## Potential Future Enhancements

* Option for different pHash algorithms.
* Integration of a stronger KDF for combining pHash and passphrase.
* Allowing a user-specified salt.

---
