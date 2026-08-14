# Working Methodology

## Web exploitation

My typical workflow is:

```text
surface mapping
→ baseline request/response
→ hypothesis
→ controlled manual validation
→ source/config review when available
→ minimal exploit or proof script
→ impact confirmation
→ root-cause explanation
→ remediation / regression notes (A&D)
```

Tools commonly appearing in the retained material include Burp Suite/Repeater, curl, Python, JavaScript, source review, and small purpose-built automation.

## Digital forensics

My forensic workflow is evidence-first:

```text
preserve / hash
→ identify container & filesystem
→ enumerate artifacts
→ reduce the search space
→ correlate timeline / network / host evidence
→ extract or reconstruct payload
→ validate recovered data
→ document chain of reasoning
```

The retained evidence spans PCAP analysis, steganography, spectrogram recovery, VM/disk-image extraction, Windows host artifacts, IIS artifacts, Linux persistence/service artifacts, and data recovery.

## Attack/Defense

For A/D finals I treat availability as part of security. A useful patch must:

1. close the actual root cause;
2. preserve the service contract/checker behavior;
3. survive syntax and functional checks;
4. be small enough to rollback quickly;
5. be followed by a security regression test.
