# Berkas Perkara

**Event:** POLRI CTF 2026  
**Stage:** Quals — Jeopardy  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

A multi-artifact evidence bundle required one file to explain how another should be interpreted. Structural inspection ruled out simple appended payloads, after which a WAV spectrogram and PNG pixel data became the key evidence sources.

## Investigation / exploitation path

1. Record file types and SHA-256 values for the supplied evidence.
2. Inspect PNG/WAV structure and metadata; rule out a simple appended-file path.
3. Generate a spectrogram from the WAV and recover the clue `SAKSI-99`.
4. Treat that clue as the password/keying input for the image stage.
5. Extract the red-channel LSB bitstream from the PNG.
6. Parse the embedded payload as `IV || ciphertext`.
7. Derive an AES key from the recovered password and decrypt the ciphertext with AES-CBC.
8. Validate the recovered plaintext against the expected CTF evidence format.

## Root cause / key observation

The challenge was an evidence chain: **WAV → spectrogram clue → PNG LSB payload → cryptographic recovery**. Skipping the ordering would leave the image payload without its keying material.

## What this demonstrates

- evidence hashing and file triage
- spectrogram analysis
- LSB steganography
- payload parsing
- SHA-256 key derivation
- AES-CBC recovery
