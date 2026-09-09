# Cipher Field Notes

**TryHackMe — encoding, ciphers & steganography · 11/11 solved · 4 September 2026**

Eleven challenges, one room. Every layer we peeled off, how we spotted it, and the six mistakes that cost the most time.

| | |
|---|---|
| Challenges | 11 |
| Deepest stack | 5 layers |
| Encodings met | 8 |
| Carrier files | WAV · JPG · RAR |

---

## 00 · The brief

The room is a ladder. It opens with a single encoding you can decode by hand, and closes with a JPEG that has a RAR archive glued onto its tail and a PNG inside that with a string buried in its bytes. Nothing here is cryptography in the hard sense — no keys, no brute force. It is all *representation*: the same bytes wearing different clothes.

Which makes the real skill **identification**, not decoding. Once you name the encoding, every tool decodes it in one command. The whole room is a training set for one question: *what am I looking at?*

---

## 01 · Recognition

The single most useful page here. Each encoding leaves a fingerprint in its **character set** — what symbols appear, and which never do.

| Looks like | It's | The tell |
|---|---|---|
| `01101100 01100101` | **Binary** | Only `0` and `1`, in groups of eight. Bytes starting `01` mean printable ASCII. |
| `68 65 78 61 64` | **Hex** | Pairs from `0-9 a-f`. A `20` in the stream is a space. |
| `85 110 112 97 99` | **Decimal ASCII** | Plain numbers, all 32–126. A recurring `32` marks the word breaks. |
| `MJQXGZJTGIQ…===` | **Base32** | Uppercase `A–Z` plus digits `2–7` only. Long `=` padding runs. |
| `LS0tLS0gLi0t…=` | **Base64** | Mixed case, `+` `/`, length always a multiple of 4. |
| `Ebgngr zr 13` | **ROT13** | Word shapes and spaces survive; only letters are scrambled. |
| `*@F DA:? >6 C:89E` | **ROT47** | Punctuation appears *inside* words. Spaces still survive. |
| `- . .-.. . -.-.` | **Morse** | Two symbols only, in short space-separated groups. |

### The two-question triage

1. **Do spaces survive?** If yes it is a rotation cipher (ROT-something) — encodings destroy word boundaries.
2. **What is the alphabet?** That narrows it to one row above.

Those two questions resolve nearly every text challenge in this room.

---

## 02 · Walkthrough

In room order. The numbering matters — the difficulty is cumulative, and challenge 08 only makes sense once 01, 05, 06 and 07 are muscle memory.

### Phase one — single-layer encodings

#### 01 · Binary

```
01101100 01100101 01110100 01110011 00100000 …
```

Each 8-bit group is one character code. `01101100` = 108 = `l`. Every group began `01`, which is the printable-ASCII band — the giveaway that it decodes to text rather than raw data.

> **Answer:** `lets try some binary out!`

#### 02 · Base32

```
MJQXGZJTGIQGS4ZAON2XAZLSEBRW63LNN5XCA2LOEBBVIRRHOM======
```

Uppercase letters and the digits 2–7 with a six-character `=` tail. No lowercase anywhere, which rules out Base64 immediately.

> **Answer:** `base32 is super common in CTF's`

#### 03 · Hexadecimal

```
68 65 78 61 64 65 63 69 6d 61 6c 20 6f 72 20 …
```

Byte pairs in base 16. This is where `xxd -r -p` earns its place — one pipe and it's done, no dictionary needed.

> **Answer:** `hexadecimal or base16?`

#### 04 · ROT13

```
Ebgngr zr 13 cynprf!
```

Spaces and the digits survived untouched — so a rotation, not an encoding. And the plaintext number `13` sitting in the ciphertext is the puzzle telling you its own key.

> **Answer:** `Rotate me 13 places!`

#### 05 · ROT47

```
*@F DA:? >6 C:89E C@F?5 323J C:89E C@F?5 Wcf E:>6DX
```

ROT13 only touches letters, so it can never put a `:` in the middle of a word. ROT47 rotates the whole printable-ASCII range 33–126 — punctuation, digits and all — while leaving space (32) alone. That mix of symbols inside word-shaped chunks is the signature.

> **Answer:** `You spin me right round baby right round (47 times)`

#### 06 · Morse code

```
- . .-.. . -.-. --- -- -- ..- -. .. -.-. .- - .. --- -.
```

Two symbols, grouped. Letters are separated by one space, words by a larger gap. Worth memorising the short ones: `.` E, `-` T, `..` I, `--` M.

> **Answer:** `telecommunication encoding`

#### 07 · Decimal ASCII

```
85 110 112 97 99 107 32 116 104 105 115 32 66 67 68
```

Bare numbers, every one inside 32–126. The repeated `32` is the space character, and seeing it land between word-length runs confirms the read before you decode anything.

> **Answer:** `Unpack this BCD`

### Phase two — the stacked one

#### 08 · Five layers deep

```
LS0tLS0gLi0tLS0gLi0tLS0gLS0tLS0g… (7,296 characters)
```

```
Base64 → Morse → Binary → ROT47 → Decimal ASCII
```

The hard part was not any single layer — it was **trusting the intermediate output**. After the binary step the text read `` fe `_` ``e bh… ``, which looks exactly like a failed decode. It wasn't. Every character sat in the narrow band `_` to `h` with the spaces still intact: that is what ROT47 does to a string of digits.

**The rule this taught:** random-looking bytes mean you got it wrong; *structured*-looking garbage means there's another layer. Narrow character range plus surviving spaces is never noise.

> **Answer:** `Let's make this a bit trickier..`

### Phase three — files, not text

#### 09 · Spectrogram

A `.wav` with text painted into its frequency content — inaudible, invisible in the waveform, obvious the moment you plot frequency against time.

```bash
sox secretaudio.wav -n spectrogram -o spec.png
xdg-open ./spec.png
```

Audacity does the same thing: track dropdown → Spectrogram, then raise the max frequency in the settings.

> **Answer:** `Super Secret Message`

#### 10 · Steghide

A JPEG carrying an embedded payload. The passphrase was blank — just press Enter. If it isn't, `stegseek` tears through rockyou in seconds.

```bash
steghide extract -sf stegosteg.jpg      # Enter for blank passphrase
cat steganopayload2248.txt

# if the passphrase isn't blank:
stegseek stegosteg.jpg /usr/share/wordlists/rockyou.txt
```

**Format matters:** steghide handles JPEG, BMP, WAV and AU. For PNG you need `zsteg` instead — different container, different technique.

> **Answer:** `SpaghettiSteg`

#### 11 · Security through obscurity

A JPEG with a whole RAR archive concatenated onto the end. Image viewers stop reading at the JPEG end-marker, so the extra megabyte is simply never noticed.

```bash
binwalk meme.jpg                                  # RAR found at offset 74407
dd if=meme.jpg of=secret.rar bs=1 skip=74407
unrar l secret.rar                                # -> hackerchat.png
unrar x secret.rar
strings hackerchat.png | grep _
```

`unzip` failed here and that was the useful signal — `binwalk` named it RAR, not ZIP. Never assume the archive type; let binwalk tell you. And the final string wasn't in the picture at all, it was in the file's raw bytes.

> **Answer 1:** `hackerchat.png`
> **Answer 2:** `AHH_YOU_FOUND_ME!`

---

## 03 · Toolbox

Everything the room needed, in the order you'd reach for it.

### CyberChef — browser

Drag operations into a recipe and they chain. *Magic* auto-detects; *ROT13 Brute Force* shows all 25 shifts at once. The right first stop for anything text-shaped. <https://gchq.github.io/CyberChef>

### xxd — hex

```bash
echo '68 65 78' | xxd -r -p
```

`-r` reverses, `-p` means plain. Shortest hex decode there is.

### base64 / base32 — coreutils

```bash
base64 -d cipher.txt
base32 -d cipher.txt
```

Feed them a *file*, not an `echo` — long strings get mangled by shell quoting.

### binwalk — carrier files

```bash
binwalk suspect.jpg
binwalk -e suspect.jpg
```

Scans for embedded file signatures at any offset. Run it on *every* file challenge before anything else.

### steghide / stegseek — jpg, bmp, wav

```bash
steghide extract -sf image.jpg
stegseek image.jpg rockyou.txt
```

Blank passphrase first, always. stegseek only if that fails.

### zsteg — png, bmp

```bash
zsteg -a image.png
```

The PNG counterpart to steghide — walks the LSB planes and hidden chunks.

### sox — audio

```bash
sox in.wav -n spectrogram -o spec.png
```

Renders the frequency plot. Any audio challenge gets this treatment first.

### strings + exiftool — always

```bash
strings file | grep -i flag
exiftool file
```

Ten seconds, and it solved challenge 11 outright. Never skip these two.

---

## 04 · What bit us

Six real errors from this session. None were about cryptography; every one cost more time than the puzzle it was blocking.

| Symptom | Fix |
|---|---|
| `base64: invalid input` | The pasted string was truncated. Base64 length is always a multiple of 4 — if it isn't, you lost characters on the way. Paste into a file with `nano` rather than into a shell command. |
| Text won't select on the page | Open DevTools and run `copy(document.body.innerText)` — the whole page text goes to your clipboard, no dragging. Firefox blocks console pasting until you type the words `allow pasting` first. |
| `Unable to detect the URI-scheme` | `xdg-open` needs a path it recognises. Use `xdg-open ./spec.png`, or just open it in a browser. |
| unrar prints its help page | You typed `1` where the command is a lowercase `l` for *list*. Easy to miss in a terminal font — worth a second look whenever a tool answers with usage text. |
| `End-of-central-directory not found` | `unzip` was the wrong tool — the payload was RAR. Run `binwalk` first and let it name the format instead of guessing from the extension. |
| A decode that "looks broken" | Structured garbage is another layer, not a failure. Check the character range and whether spaces survived before you throw the output away. |

---

## 05 · Recall test

Identification only — no decoding. That is the skill worth drilling. Answers below.

1. `NBUWIZDFNYQGS3RAOBWGC2LOEBZWSZ3IOQ======` — Base64, Base32, Hex or ROT47?
2. `Gur synt vf uvqqra urer` — ROT47, Base64, ROT13 or Morse?
3. `%96 7=28 :D 96C6` — ROT13, ROT47, Hex or Binary?
4. A decode returns `` fe `_` ``e bh `` — narrow character range, spaces intact. What now?
5. A JPEG hides something. Which command runs *first*?
6. You need to find text hidden in a PNG. steghide or zsteg?
7. A `.wav` with no audible message. Next move?

<details>
<summary><strong>Answers</strong></summary>

1. **Base32.** Uppercase A–Z and digits 2–7 only — no lowercase, no `+` or `/`. That alphabet is Base32.
2. **ROT13.** Spaces and word shapes survive and only letters are altered — the ROT13 signature.
3. **ROT47.** Punctuation sits inside the words. ROT13 can never do that; ROT47 rotates the whole printable range.
4. **Try the next layer.** Structured garbage means another layer. Random bytes mean a mistake. This one was ROT47 over digits.
5. **`binwalk`.** It names what is actually embedded and at what offset, so you don't guess the archive format.
6. **zsteg.** steghide supports JPEG, BMP, WAV and AU — not PNG.
7. **Render a spectrogram.** Text painted into the frequency domain is silent and invisible until you plot frequency against time.

</details>

---

*TryHackMe · encoding, ciphers and steganography · solved 4 September 2026 · 11 of 11 challenges*
