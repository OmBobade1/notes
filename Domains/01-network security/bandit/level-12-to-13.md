## Level
Bandit12 → Bandit13

## Challenge
The password file is actually a hexdump of a file that's been compressed multiple times in a row (gzip, bzip2, tar, etc. nested).

## Technique Used
Convert the hexdump back to binary with `xxd -r`, then repeatedly inspect with `file` and decompress with the matching tool (gzip/bzip2/tar/xz) until a plain text file remains.

## Commands
```bash
mkdir /tmp/work12 && cp data.txt /tmp/work12 && cd /tmp/work12
xxd -r data.txt > data
file data   # inspect type, then repeat as needed:
mv data data.gz && gzip -d data.gz
file data   # repeat file/rename/decompress cycle (bzip2 -d, tar -xf, etc.) until you get ASCII text
```
