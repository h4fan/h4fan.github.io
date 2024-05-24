---
layout: post
title:  BitLocker recovery key 是如何工作的
tags: [bitlocker,crypt]
---

磁盘加密是一种常用的保护数据安全的方式，避免电脑丢失或维修时的数据泄露。  
不同的系统支持不同的加密方式。对于Windows，使用的是BitLocker Drive Encryption。  

# BitLocker的加密流程
Encrypt used disk space only 和 Encrypt entire drive 怎么选？  
![image](/static/img/bitlocker_img1.webp)
# BitLocker使用的什么加解密算法？
XTS-AES  
XTS-AES is a tweakable block cipher designed for encryption of sector-based storage. XTS-AES acts on 8 data units of 128 bits or more and uses the AES block cipher as a subroutine. The key material for XTS9 AES consists of a data encryption key (used by the AES block cipher) as well as a “tweak key” that is used 10 to incorporate the logical position of the data block into the encryption.  
![image](/static/img/bitlocker_img2.webp)  
https://en.wikipedia.org/wiki/Disk_encryption_theory  
（macOS的FileVault 2也是使用的该算法）

# BitLocker为什么可以加密操作系统分区？
To function correctly, BitLocker requires a specific disk configuration. BitLocker requires two partitions that meet the following requirements:
The operating system partition contains the operating system and its support files; it must be formatted with the NTFS file system
The system partition (or boot partition) includes the files needed to load Windows after the BIOS or UEFI firmware has prepared the system hardware. BitLocker isn't enabled on this partition. For BitLocker to work, the system partition must not be encrypted, and must be on a different partition than the operating system. On UEFI platforms, the system partition must be formatted with the FAT 32-file system. On BIOS platforms, the system partition must be formatted with the NTFS file system. It should be at least 350 MB in size.

# BitLocker recovery key 是如何工作的？
## What is a BitLocker recovery key?  
Your BitLocker recovery key is a unique 48-digit numerical password that can be used to unlock your system if BitLocker is otherwise unable to confirm for certain that the attempt to access the system drive is authorized.  

你是否也好奇，为什么不输入你的密码，也能解密硬盘？难道算法支持两个独立的密钥，都可以用来解密数据？
是，也不是。  
在我们的印象中，
```
      密钥  
      ｜  
密文 ->算法-> 明文
``` 
即便是非对称加密算法，也是一个密钥用来加密，另外一个密钥用来解密，也没有2个不同的密钥用来解密。  
那么实际是如何做到的呢？  
实际情况是，真正用来加解密的密钥仍然是只有1个，即Full Volume Encryption Key（FVEK），而FVEK是由Volume Master Key(VMK)加密保存的，而VMK会存储在多个地方，如TPM或者硬盘上。VMK存储时，可以使用user PIN加密存储1份，同时也会使用recovery key存储1份。  
这里，我们可以知道，我们的user PIN和recovery key分别用来加密了VMK，因此，当我们无法提供user PIN时，使用recovery key可以用来解密另一份VMK，继而可以解密FVEK，然后可以解密硬盘数据。  
Recovery key的存储需要放在另外的地方，因为当你的硬盘无法解密时，存放在上面的文件你也无法读取。  

https://learn.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-basic-deployment?source=recommendations
https://www.quora.com/How-does-the-recovery-key-work-for-BitLocker
https://crypto.stackexchange.com/questions/34480/how-does-microsofts-bitlocker-recovery-code-work
https://luca-giuzzi.unibs.it/corsi/Support/papers-cryptography/1619-2007-NIST-Submission.pdf
