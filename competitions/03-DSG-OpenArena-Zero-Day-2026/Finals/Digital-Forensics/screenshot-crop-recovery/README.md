# Screenshot Crop Recovery

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Finals  
**Category:** Digital Forensics  
**Evidence status:** Confirmed — retained final write-up

## Summary

An image appeared safely cropped in a viewer, but the underlying file still contained data beyond the logical PNG end. The challenge required distinguishing rendered appearance from stored bytes and recovering the leftover image information.

## Investigation / exploitation path

1. Validate the PNG structure and locate the logical end (`IEND`).
2. Check whether bytes remain after the normal end of the displayed image.
3. Carve/reconstruct the leftover image data from the trailing region.
4. Validate that the recovered content corresponds to the pre-crop information expected by the challenge.

## Root cause / key observation

Visual redaction/cropping is not equivalent to data removal. A file can render only the new view while still retaining recoverable bytes from an earlier image state.

## What this demonstrates

- PNG structure analysis
- trailing-data detection
- file carving/reconstruction
- forensic validation of visual redaction failures
