---
title: "macOS NT QQ 聊天记录解密"
dates: 2026-03-31
category: "技术分享"
tags:
  - 逆向工程
  - SQLite
  - SQLCipher
  - 
description: "揭秘 macOS NT QQ 聊天记录的存储与解密过程"
keywords: QT, SQLite, SQLCipher, 聊天记录，逆向工程
---

## 背景

最近出于兴趣研究一下 macOS 上新版 NT QQ 的聊天记录存储机制。发现其使用了加密的 SQLite 数据库来存储消息数据。

## 环境信息

- **系统**: macOS
- **软件**: NT QQ (新版腾讯 QQ)
- **数据库类型**: SQLCipher (加密 SQLite)
- **加密算法**: AES-256 + HMAC-SHA1

## 数据定位

NT QQ 的用户数据主要存储位置：

```bash
~/Library/Application\ Support/QQ/
```

在这个目录下可以找到两个关键的数据库文件：

1. `nt_msg.clean.db` - 主消息数据库（约 2.8GB）
2. `profile_info.db` - 用户配置信息（约 9.8MB）

## 密钥获取

NT QQ 将解密密钥存储在一个文本文件中，通常命名为类似 `@qqkey` 的文件。通过查看该文件可以找到数据库的解密密码。

> **注意**: 不同版本的 QQ 可能使用不同的密钥存储方式和位置。

## 解密过程

### 1. 导出 SQL Dump

使用 `sqlite3` 命令行工具配合正确的加密参数导出数据：

```bash
sqlcipher nt_msg.clean.db <<EOF
PRAGMA key = '<实际密码已隐去>';  # 从密钥文件获取的密码
PRAGMA kdf_iter = 4096;           # QQ 使用的 KDF 迭代次数
PRAGMA cipher_hmac_algorithm = HMAC_SHA1;
PRAGMA cipher_page_size = 4096;   # 默认页面大小
.output raw_dump.sql
.dump
.exit
EOF
```

### 2. 修复 SQL Dump

由于数据库可能处于事务状态，导出的 SQL 文件会以 `ROLLBACK` 结尾，需要替换为 `COMMIT`：

```bash
cat raw_dump.sql | sed -e 's|^ROLLBACK;\( -- due to errors\)*$|COMMIT;|g' | sqlite3 nt_msg_fixed.db
```

### 3. 读取解密后的数据

```bash
sqlite3 nt_msg_fixed.db ".tables"
```

可以看到多个数据表，主要包括：

```
c2c_msg_table              # 私聊消息表
group_msg_table            # 群聊消息表
dataline_msg_table         # 长消息
recent_contact_v3_table    # 最近联系人
c2c_temp_msg_table         # 临时会话消息
discuss_msg_table          # 讨论组消息
```

## 注意事项

1. **PRAGMA kdf_iter**: NT QQ 使用的是 `4096` 次迭代，而非 SQLCipher 的默认值 `4000`。这点很重要，如果参数不对会导致解密失败。
2. **事务处理**: 导出的 SQL dump 可能包含未提交事务，需要手动修复。
3. **数据完整性**: 建议先将解密后的数据库备份再进行查询操作。

## 结语

通过以上步骤，就可以成功解密 NT QQ 的聊天记录并导出数据。整个过程中最重要的是获取正确的加密密钥和配置参数。这个研究过程让我对 Qt 应用的数据存储机制有了更深的理解。

---

> **免责声明**: 本文仅用于技术学习和个人数据备份，请勿用于非法用途或侵犯他人隐私。
