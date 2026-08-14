# Insiden Lampiran

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Finals  
**Category:** Digital Forensics  
**Evidence status:** Confirmed — detailed retained final write-up

## Summary

The case combined a malicious-looking DOCX, PCAP evidence, and incident notes. The investigation treated the Word document as an OOXML container, correlated its external relationship behavior with network traffic, then decoded the staged script/exfiltration chain.

## Investigation / exploitation path

1. Unzip the DOCX as OOXML instead of opening it interactively.
2. Inspect relationships and identify a linked OLE/`htmlfile` object that references external HTML content.
3. Correlate that external reference with DNS/HTTP activity in the PCAP.
4. Decode the staged `ms-msdt:` / PowerShell behavior from the document/network evidence.
5. Recover key material derived from document properties and DNS beacons.
6. Decode/decompress the outbound data and reconstruct the exfiltrated content.
7. Build an incident narrative and IOC set from multiple evidence sources.

## Root cause / key observation

The conclusion depends on **document structure + network evidence + script decoding**. No single artifact is sufficient by itself.

## What this demonstrates

- OOXML/document forensics
- PCAP correlation
- encoded PowerShell analysis
- DNS/HTTP evidence linking
- exfiltration reconstruction
- IOC/timeline reporting
