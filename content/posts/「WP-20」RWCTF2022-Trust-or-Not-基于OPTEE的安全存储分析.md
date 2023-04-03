---
title: 「RealWorldCTF2022」Rev | Trust_or_Not 基于 OPTEE 的安全存储分析
date: 2022-03-30 20:18:16
tags:
- Rev
categories:
- TECHNOLOGY
---


<!-- more -->

## TL;DR

这道题是关于 optee 即可信执行环境的开源内核, 我们需要仔细了解 ARM 的 TrustZone 相关技术、optee 的架构, 尤其是文件安全存储的实现细节. 做题过程中需要不停查询文档、翻阅源码、启动 qemu 调试等等, 最好是自己拉源码去 build 一遍, 至少可以先有一个直观的认知.

## Secure Storage

> Reference:
> 
> https://optee.readthedocs.io/en/latest/architecture/secure_storage.html



从中可以看出加密过程如下图:
![](https://img-blog.csdnimg.cn/img_convert/a51a584a02c1ae48692d34b7f286e040.png)

我们的目标就是拿到 FEK 然后去解密 /data/tee/2 这个文件, 这个文件是 REE 的. Secure Enclave 先将它的明文形式写入 TEE_FileSystem , 然后通过 KeyManager 加密后调用 API 写入 REE_FileSystem

## Encrypted TA Structure

生成 FEK 所需的一些常量比较容易拿到, 并且从 rootfs-cpio/lib/optee_armtz/f4e750bb-1437-4fbf-8785-8d3580c34994.ta 这个被加密的 TA 中我们可以拿到一些值, 怎么做到呢?

```bash
git clone https://github.com/OP-TEE/optee_os.git && cd optee_os # 获取项目源码
ctags -R * # 方便阅读
vim core/kernel/huk_subkey.c # 拿 HUK 和 static string
vim core/include/signed_hdr.h # 描述 TA 文件结构
```

先看看文件头:

```c
struct shdr {
    uint32_t magic;
    uint32_t img_type;
    uint32_t img_size;
    uint32_t algo;
    uint16_t hash_size;
    uint16_t sig_size;
    /*
     * Commented out element used to visualize the layout dynamic part
     * of the struct.
     *
     * hash is accessed through the macro SHDR_GET_HASH and
     * signature is accessed through the macro SHDR_GET_SIG
     *
     * uint8_t hash[hash_size];
     * uint8_t sig[sig_size];
     */
};
struct shdr_bootstrap_ta {
    uint8_t uuid[sizeof(TEE_UUID)];
    uint32_t ta_version;
};
struct shdr_encrypted_ta {
    uint32_t enc_algo;
    uint32_t flags;
    uint16_t iv_size;
    uint16_t tag_size;
    /*
     * Commented out element used to visualize the layout dynamic part
     * of the struct.
     *
     * iv is accessed through the macro SHDR_ENC_GET_IV and
     * tag is accessed through the macro SHDR_ENC_GET_TAG
     *
     * uint8_t iv[iv_size];
     * uint8_t tag[tag_size];
     */
};
```

然后对照着看 16 进制编辑器看就能找到 uuid 的位置是 0x134, 同时注意小端序的问题. 或者猜测可以整一个 010 模版然后导入进去.

## Hash Tree

哈希树在安全存储文件的加密和解密阶段发挥作用. 它主要是用一个二叉树的想法来实现的, 每个结点维护两个子结点和一个数据块. Meta Data 加密过程如下图. **注意**这个加密的数据是描述文件性质和属性的, 而不是真正的文件内容:

![](https://optee.readthedocs.io/en/latest/_images/meta_data_encryption.png)

看文档可以知道相关结构体定义在 core/tee/fs_htree.h 中:

```c
struct tee_fs_htree_node_image { // 一般节点
    uint8_t hash[TEE_FS_HTREE_HASH_SIZE];
    uint8_t iv[TEE_FS_HTREE_IV_SIZE];
    uint8_t tag[TEE_FS_HTREE_TAG_SIZE];
    uint16_t flags;
};

struct tee_fs_htree_meta {
    uint64_t length;
};

struct tee_fs_htree_imeta {
    struct tee_fs_htree_meta meta;
    uint32_t max_node_id;
};

struct tee_fs_htree_image { // 顶部节点
    uint8_t iv[TEE_FS_HTREE_IV_SIZE];
    uint8_t tag[TEE_FS_HTREE_TAG_SIZE];
    uint8_t enc_fek[TEE_FS_HTREE_FEK_SIZE];
    uint8_t imeta[sizeof(struct tee_fs_htree_imeta)];
    uint32_t counter;
};
```

## Decryption

首先密文结构被记录在 core/tee/tee_ree_fs.c 中:

```
File layout
[demo with input:
BLOCK_SIZE = 4096,
node_size = 66,
block_nodes = 4096/(66*2) = 31 ]

phys block 0:
tee_fs_htree_image vers 0 @ offs = 0
tee_fs_htree_image vers 1 @ offs = sizeof(tee_fs_htree_image)

phys block 1:
tee_fs_htree_node_image 0  vers 0 @ offs = 0
tee_fs_htree_node_image 0  vers 1 @ offs = node_size
tee_fs_htree_node_image 1  vers 0 @ offs = node_size * 2
tee_fs_htree_node_image 1  vers 1 @ offs = node_size * 3
...
tee_fs_htree_node_image 30 vers 0 @ offs = node_size * 60
tee_fs_htree_node_image 30 vers 1 @ offs = node_size * 61

phys block 2:
data block 0 vers 0

phys block 3:
data block 0 vers 1

...
phys block 62:
data block 30 vers 0

phys block 63:
data block 30 vers 1

phys block 64:
tee_fs_htree_node_image 31  vers 0 @ offs = 0
tee_fs_htree_node_image 31  vers 1 @ offs = node_size
tee_fs_htree_node_image 32  vers 0 @ offs = node_size * 2
tee_fs_htree_node_image 32  vers 1 @ offs = node_size * 3
...
tee_fs_htree_node_image 61 vers 0 @ offs = node_size * 60
tee_fs_htree_node_image 61 vers 1 @ offs = node_size * 61

phys block 65:
data block 31 vers 0

phys block 66:
data block 31 vers 1
...
```

结合上面的结构体发现确实是一对二的关系, 那么我们只需要去拿对应的数据, 解密就好🌶️:

```python
import binascii
import struct

with open('2', 'rb') as fin:
    data = fin.read()

def get_htree_image_data(offset:int):
    htree_img_iv = data[offset + 0x00: offset + 0x10]
    htree_img_tag = data[offset + 0x10 : offset + 0x20]
    htree_img_enc_fek = data[offset + 0x20 : offset + 0x30]
    htree_img_imeta = data[offset + 0x30 : offset + 0x40]
    htree_img_counter = struct.unpack("I", data[offset + 0x40 : offset + 0x44])[0]
    return htree_img_iv, htree_img_tag, htree_img_enc_fek, htree_img_imeta, htree_img_counter

def get_htree_node_image_data(offset:int):
    htree_node_img_hash = data[offset + 0x00: offset + 0x20]
    htree_node_img_iv = data[offset + 0x20 : offset + 0x30]
    htree_node_img_tag = data[offset + 0x30 : offset + 0x40]
    htree_node_img_flags = struct.unpack("H", data[offset + 0x40 : offset + 0x42])[0]
    return htree_node_img_hash, htree_node_img_iv, htree_node_img_tag, htree_node_img_flags

# fs_htree_image_ver0
htree_img_iv0, htree_img_tag0, htree_img_enc_fek0, htree_img_imeta0, htree_img_counter0 = get_htree_image_data(0)
print(list(map(binascii.hexlify, [htree_img_iv0, htree_img_tag0, htree_img_enc_fek0, htree_img_imeta0])))
print(htree_img_counter0)

# fs_htree_image_ver1
htree_img_iv1, htree_img_tag1, htree_img_enc_fek1, htree_img_imeta1, htree_img_counter1 = get_htree_image_data(0x44)
print(list(map(binascii.hexlify, [htree_img_iv1, htree_img_tag1, htree_img_enc_fek1, htree_img_imeta1])))
print(htree_img_counter1)

# fs_htree_node_image_ver0
htree_node_img_hash0, htree_node_img_iv0, htree_node_img_tag0, htree_node_img_flags0 = get_htree_node_image_data(0x1000)
print(list(map(binascii.hexlify, [htree_node_img_hash0, htree_node_img_iv0, htree_node_img_tag0])))
print(htree_node_img_flags0)

# fs_htree_node_image_ver1
htree_node_img_hash1, htree_node_img_iv1, htree_node_img_tag1, htree_node_img_flags1 = get_htree_node_image_data(0x1042)
print(list(map(binascii.hexlify, [htree_node_img_hash1, htree_node_img_iv1, htree_node_img_tag1])))
print(htree_node_img_flags1)
```

解密涉及到 AES-GCM 和 AAD:

```python
from hashlib import sha256
from hmac import HMAC
from Crypto.Cipher import AES

HUK = b'\x00' * 0x10
chip_id = b'BEEF' * 8
static_string = b'ONLY_FOR_tee_fs_ssk'
message = chip_id + static_string + b'\x00'
ta_uuid =  b'\xbb\x50\xe7\xf4\x37\x14\xbf\x4f\x87\x85\x8d\x35\x80\xc3\x49\x94'

SSK = HMAC(HUK, message, digestmod = sha256).digest()
TSK = HMAC(SSK, ta_uuid, digestmod = sha256).digest()
FEK = AES_Decrypt_ECB(TSK, htree_img_enc_fek1)

block_data = data[0x2000 : 0x3000]
cipher = AES.new(FEK, AES.MODE_GCM, nonce = htree_node_img_iv1)
cipher.update(htree_img_enc_fek1)
cipher.update(htree_node_img_iv1)

plaintext = cipher.decrypt_and_verify(block_data, htree_node_img_tag1)
print (plaintext)
```
