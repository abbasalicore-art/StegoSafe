# StegoSafe

**Secure image steganography and message recovery.** Made by Abbas Ali.

StegoSafe hides a text message inside an image using least significant bit (LSB) steganography and recovers it later. An optional password encrypts the message with AES-256 before it is hidden. It is a static web app built with only HTML5, CSS3 and vanilla JavaScript. There is no framework, build step or backend, and nothing leaves your browser.

## Run it

1. Download or clone the `StegoSafe` folder.
2. Open `index.html` in a current version of Chrome, Edge, Firefox or Safari.

No installation is needed. If you serve it instead, use `localhost` or HTTPS. Browsers disable the Web Crypto API on plain `http://` pages that are not local, and encryption needs it.

```
StegoSafe/
├── index.html   Structure, forms, icon sprite, in-page guide
├── style.css    Dark navy theme, layout, components, responsive rules
├── script.js    LSB engine, payload format, encryption, UI logic
└── README.md
```

## Features

**Hide Message**
- Choose a PNG, JPEG, WebP, BMP or GIF cover image (up to 25 MB) by browsing or dragging it in. File type is checked from the file's first bytes, not its extension.
- Preview with file name, dimensions, size and how much text the image can hold.
- A live meter shows how much of the image's capacity the message uses, including the 44 bytes added by encryption.
- Optional password with confirmation, a show/hide toggle and a rough strength indicator.
- Five visible steps: check capacity, prepare message, embed bits, write PNG, verify output.
- The finished PNG is decoded again and compared with what was embedded before success is reported.
- Download as PNG, plus a **pixel inspector** that shows the first four pixels before and after, with the changed last bits highlighted, and the stored payload fields.

**Extract Message**
- Choose an image and StegoSafe immediately reports whether it contains a hidden message and whether it is encrypted.
- If the message is encrypted, it asks for the password. A wrong password is refused.
- Recovered text appears in a read-only box with a Copy button.
- **Clear** removes the recovered text, password, image preview and file selection.

**Everywhere**
- Clear errors for unsupported or damaged files, images that are too small, missing or damaged payloads and wrong passwords.
- Keyboard support: arrow keys, Home and End move between tabs, Ctrl+Enter hides from the message box, Enter submits password fields. Visible focus, labelled controls, screen-reader status messages and reduced-motion support are included.
- Layouts for desktop, tablet and mobile.

## How the LSB algorithm works

An image pixel stores red, green and blue as numbers from 0 to 255. Changing the last (least significant) bit of a number changes it by at most 1, which is invisible to the eye. StegoSafe stores one message bit in the last bit of every red, green and blue value.

1. The canvas gives the pixels as a flat list of bytes: `R, G, B, A, R, G, B, A, ...`, left to right, top to bottom.
2. The payload (header plus body, below) is written bit by bit, most significant bit of each byte first. Bit 1 goes in the red value of the first pixel, bit 2 in its green value, bit 3 in its blue value, bit 4 in the red value of the next pixel, and so on.
3. Alpha is never written. Only fully opaque pixels (alpha = 255) are used. Browsers store canvas colors premultiplied by alpha, so the color values of translucent pixels cannot be trusted to survive saving and reloading. Skipping them keeps alpha untouched and extraction exact. The extractor applies the same rule.
4. The result is exported as PNG, which is lossless, so every changed bit survives.

Extraction reads the bits back in the same order: first the 13-byte header, then exactly as many body bytes as the header says.

**Capacity** is `floor(opaque pixels × 3 ÷ 8) − 13` bytes. A 600 × 400 opaque image holds 89,987 bytes of body. With a password, 44 of those bytes go to encryption data.

### Payload format

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 4 | Signature | `STG1` (`53 54 47 31`) |
| 4 | 1 | Flags | Bit 0 = body is encrypted. All other bits must be 0 |
| 5 | 4 | Body length | Bytes, big-endian |
| 9 | 4 | CRC-32 | Of the body, big-endian |
| 13 | n | Body | See below |

- **Plain body:** the message as UTF-8 bytes.
- **Encrypted body:** `salt (16) | IV (12) | ciphertext + authentication tag (message length + 16)`.

When reading, StegoSafe rejects the image if the signature is missing (no message), unknown flag bits are set (newer format), the length is larger than the image can hold (damaged header), the data ends early (cropped) or the CRC-32 does not match (edited or recompressed). The CRC-32 detects accidental damage. Anyone who can edit the image can recompute it, so it is not a security feature.

## Password-based encryption

When a password is entered, the message is encrypted before it is hidden:

1. A random 16-byte salt and a random 12-byte IV are generated with `crypto.getRandomValues`. New values are used for every message.
2. The password is normalized (Unicode NFKC) and turned into a 256-bit key with **PBKDF2, SHA-256, 600,000 iterations** using the Web Crypto API.
3. The message is encrypted with **AES-256-GCM** (128-bit tag). The signature and flags bytes are passed as additional authenticated data.
4. `salt | IV | ciphertext + tag` becomes the body and the encrypted flag is set.

To decrypt, StegoSafe reads the salt and IV from the body, derives the same key and decrypts. If the password is wrong, or the ciphertext or flags were changed, GCM verification fails and StegoSafe reports that the message could not be decrypted without returning any data. It does not say which of the two happened.

**Why GCM and not CBC.** The brief mentions that AES-CBC does not authenticate its ciphertext. StegoSafe uses AES-GCM instead, which does. Wrong passwords are therefore rejected reliably instead of sometimes producing garbage, and tampering with the ciphertext is detected. This is covered by tests: a forged payload with a recomputed CRC is rejected, and flipping the encrypted flag is rejected.

**Limits of the encryption**
- Authentication proves the ciphertext was not changed. It does not stop someone from replacing the whole image with another one.
- The signature, flags, length and sizes are visible to anyone who reads the pixels. They reveal that a StegoSafe payload exists, whether it is encrypted and roughly how long the message is.
- Security depends on the password. PBKDF2 slows guessing but cannot save a weak password. The strength meter is a rough estimate, not a guarantee.
- A forgotten password cannot be recovered.
- JavaScript cannot reliably erase strings from memory, so passwords and messages stay in memory until the page is closed or garbage collected.

## Privacy and security

- Images, messages and passwords are processed locally. The page makes no network requests. A Content-Security-Policy meta tag blocks connections, remote images, frames and form posts.
- Nothing is written to `localStorage`, `sessionStorage`, cookies or the console. The only thing in the address bar is the tab name (`#hide` or `#extract`).
- User text is only ever inserted with `textContent` or `.value`, never as HTML.
- Error messages explain what to do without technical details.
- Steganography hides that a message exists. It does not make an image impossible to analyze. Statistical tests can reveal LSB changes, and the `STG1` signature is recognizable to anyone who knows the format. Use a strong password for anything sensitive.

## Image handling

- Supported inputs: PNG, JPEG, WebP, BMP and GIF (first frame). Output is always PNG.
- Images above **16 megapixels** are scaled down when hiding, to keep memory use reasonable. The notice shows the old and new size. A smaller image holds less text. Extraction never resizes, because that would destroy the message, so it refuses images above 16 megapixels.
- Extraction reads the image at its own pixel dimensions, with the same pixel order used for hiding. No other normalization is applied.
- Hiding works on a copy of the pixels, so you can generate again from the same cover image with a different message.

## Testing

### Manual test matrix

| # | Test | Expected result |
|---|---|---|
| 1 | Hide a short message in a PNG, download, reopen, extract | Same text recovered |
| 2 | Use a JPEG cover | Works. Notice explains output is PNG |
| 3 | Fill an image exactly to capacity | Succeeds. One more byte shows a capacity error |
| 4 | Use a tiny image (4 × 4) | Error: no usable space |
| 5 | Hide without a password | Detected as plain text, no password asked |
| 6 | Hide with a password, extract with the right one | Text recovered |
| 7 | Extract with a wrong password | "Could not decrypt the message", nothing shown |
| 8 | Extract an encrypted image without a password | Asks for the password |
| 9 | Extract from an image with no hidden data | "No hidden message found" |
| 10 | Choose a text file renamed `.png` | "Not a supported image" |
| 11 | Download, **refresh the page**, extract | Works, since nothing depends on page state |
| 12 | Press Clear after extracting | Text, password, preview and file selection are gone |
| 13 | Resize or save the PNG as JPEG or WebP, then extract | Message not found or reported damaged. This is expected |
| 14 | Hide in a PNG with transparency | Alpha unchanged, message recovered |

### What was run

Both suites were run against this code in Node 22 and headless Chromium 141 (Linux).

- **Core logic (Node, 27 tests):** CRC-32 known value, signature and header layout, bit order, that only the last bit of red, green and blue ever changes, exact-capacity and over-capacity messages, encrypted round trips, 25 consecutive wrong passwords, random salt and IV, tampered ciphertext with a recomputed CRC, flipped flags, 200 random images with no payload, bit flips, forged lengths, cropped data, simulated recompression and transparency.
- **Browser (Playwright, 89 checks):** every row of the matrix above, plus real PNG download and reload in a fresh page, pixel comparison of cover and output (no value differs by more than 1, alpha identical), a 20-megapixel image, a lossless re-save, keyboard use, responsive layouts at 390, 820 and 1440 px with no horizontal scroll, and a clean console with no secrets logged.

To try the core without a browser, load `script.js` from Node. It exports the pure functions (`createPayload`, `embedPayload`, `readPayload`, `openPayload`, and others) when loaded as a module. In a browser they are also available as `window.StegoSafeCore`.

## Known limitations

- **Anything that changes pixels destroys the message:** resizing, cropping, filters, screenshots, JPEG or WebP conversion and services that recompress images on upload. Share the downloaded PNG as a file.
- **Browsers that randomize canvas output** (some privacy modes and extensions) alter pixel values. StegoSafe's read-back check catches this and reports it instead of giving you a broken file.
- **Transparent areas hold no data.** Mostly transparent images have little or no capacity.
- **Animated GIFs** use the first frame only. Output is a still PNG.
- **Text only.** Files are not supported.
- Images above 16 megapixels are reduced when hiding. Very large images can still be slow on low-memory phones.
- The format is fixed at version `STG1` with 600,000 PBKDF2 iterations. Images made by other tools, or by a future version with different settings, are not readable.
- Testing was done in Chromium only. The code uses standard APIs (Canvas, Web Crypto, File API) that Firefox and Safari also support, but they were not run here.
