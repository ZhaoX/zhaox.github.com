---
layout: post
title: "ECC公钥格式详解"
description: ""
category: Block Chain 
tags: [ECC, Cryptography, Java]
---
{% include JB/setup %}

本文首先介绍公钥格式相关的若干概念/技术，随后以示例的方式剖析DER格式的ECC公钥，最后介绍如何使用Java生成、解析和使用ECC公钥。

### ASN.1

[Abstract Syntax Notation One (ASN.1)](https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One)是一种接口描述语言，提供了一种平台无关的描述数据结构的方式。ASN.1是ITU-T、ISO、以及IEC的标准，广泛应用于电信和计算机网络领域，尤其是密码学领域。

ASN.1与大家熟悉的[Protocol Buffers](https://en.wikipedia.org/wiki/Protocol_Buffers)和[Apache Thrift](https://en.wikipedia.org/wiki/Apache_Thrift)非常相似，都可以通过schema来定义数据结构，提供跨平台的数据序列化和反序列化能力。不同的是，ASN.1早在1984年就被定为标准，比这两者要早很多年，并得到了广泛的应用，被用来定义了很多世界范围内广泛使用的数据结构，有大量的RFC文档使用ASN.1定义协议、数据格式等。比如https所使用的X.509证书结构，就是使用ASN.1定义的。

ASN.1定义了若干基础的数据类型和结构类型：

Topic             | Description
----------------- | -------------
Basic Types       | BIT STRING <br>BOOLEAN <br>INTEGER <br>NULL <br>OBJECT IDENTIFIER <br>OCTET STRING
String Types      | BMPString <br>IA5String <br>PrintableString <br>TeletexString <br>UTF8String
Constructed Types | SEQUENCE <br>SET <br>CHOICE

上述的基础类型可以在[这里](https://msdn.microsoft.com/en-us/library/windows/desktop/bb540789(v=vs.85).aspx)找到详尽的解释。我们可以使用这些来描述我们自己的数据结构：

```
    FooQuestion ::= SEQUENCE {
        trackingNumber INTEGER,
        question       IA5String
    }
```

如上定义了一个名为FooQuestion的数据结构。它是一个SEQUENCE结构，包含了一个INTEGER一个IA5String
一个具体的FooQuestion可以描述为：

```
    myQuestion FooQuestion ::= {
        trackingNumber     5,
        question           "Anybody there?"
    }
```

用ASN.1定义的数据结构实例，可以序列化为二进制的BER、文本类型的JSON、XML等。

### Object Identifier

[Object Identifier (OID)](https://en.wikipedia.org/wiki/Object_identifier)是一项由ITU和ISO/IEC制定的标准，用来唯一标识对象、概念，或者其它任何具有全球唯一特性的东西。

一个OID表现为用.分隔的一串数字，比如椭圆曲线secp256r1的OID是这样：

```
1.2.840.10045.3.1.7
```

其每个数字的含义如下：
```
iso(1) member-body(2) us(840) ansi-X9-62(10045) curves(3) prime(1) 7
```

OID是全局统一分配的，全部的OID可以看做一棵多叉树，每一个有效的OID表现为树上的一个节点。当前所有的OID可以在[这里](http://www.oid-info.com/)找到。

OID是ASN.1的基本类型。

### BER & DER

[Basic Encoding Rules (BER)](https://en.wikipedia.org/wiki/X.690#BER_encoding)是一种自描述的ASN.1数据结构的二进制编码格式。每一个编码后的BER数据依次由数据类型标识（Type identifier），长度描述（Length description）, 实际数据（actual Value）排列而成，即BER是一种二进制TLV编码。TLV编码的一个好处，是数据的解析者不需要读取完整的数据，仅从一个不完整的数据流就可以开始解析。

[Distinguished Encoding Rules (DER)](https://en.wikipedia.org/wiki/X.690#DER_encoding)是BER的子集，主要是消除了BER的一些不确定性的编码规则，比如在BER中Boolean类型true的value字节，可以为任何小于255大于0的整数，而在DER中，value字节只能为255。DER的这种确定性，保证了一个ASN.1数据结构，在编码为为DER后，只会有一种正确的结果。这使得DER更适合用在数字签名领域，比如X.509中广泛使用了DER。

关于各种ASN.1数据类型是如何被编码为DER，可以在[这里](https://msdn.microsoft.com/en-us/library/windows/desktop/bb540809(v=vs.85).aspx)找到详尽的解释。

如果有DER数据需要解析查看内容，这里有一个很方便的[在线工具](http://lapo.it/asn1js/)。

用DER来编码ASN.1小节中自定义的myQuestion如下：

```
0x30 0x13 0x02 0x01 0x05 0x16 0x0e 0x41 0x6e 0x79 0x62 0x6f 064 0x79 0x20 0x74 0x68 0x65 0x72 0x65 0x3f
---  ---  ---  ---  ---  ---  ---  --------------------------------------------------------------------
 ^    ^    ^    ^    ^    ^    ^                                   ^
 |    |    |    |    |    |    |                                   |
 |    |    | INTEGER | IA5STRING                                   |
 |    |    | LEN=1   | TAG     |                                   |
 |    |    |         |         |                                   |
 |    | INTEGER   INTEGER   IA5STRING                          IA5STRING
 |    | TAG       VALUE(5)  LEN=14                             VALUE("Anybody there?")
 |    |
 |    |  ----------------------------------------------------------------------------------------------
 |    |                                              ^
 |  SEQUENCE LEN=19                                  |
 |                                                   |
SEQUENCE TAG                                  SEQUENCE VALUE

```

### PEM

DER格式是ASN.1数据的二进制编码，计算机处理方便，但不利于人类处理，比如不方便直接在邮件正文中粘贴发送。PEM是DER格式的[BASE64编码](https://en.wikipedia.org/wiki/Base64)。除此之外，PEM在DER的BASE64前后各增加了一行，用来标识数据内容。示例如下：

```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDMYfnvWtC8Id5bPKae5yXSxQTt
+Zpul6AnnZWfI2TtIarvjHBFUtXRo96y7hoL4VWOPKGCsRqMFDkrbeUjRrx8iL91
4/srnyf6sh9c8Zk04xEOpK1ypvBz+Ks4uZObtjnnitf0NBGdjMKxveTq+VE7BWUI
yQjtQ8mbDOsiLLvh7wIDAQAB
-----END PUBLIC KEY-----
```

### X.509

[X.509](https://en.wikipedia.org/wiki/X.509)是一项描述[公钥证书](https://en.wikipedia.org/wiki/Public_key_certificate)结构的标准，广泛使用在HTTPS协议中，定义在[RFC 3280](https://tools.ietf.org/html/rfc3280)

X.509使用ASN.1来描述公钥证书的结构，通常编码为DER格式，也可以进一步BASE64编码为可打印的PEM格式。V3版本的X.509结构如下：

```
    Certificate  ::=  SEQUENCE  {
        tbsCertificate       TBSCertificate,
        signatureAlgorithm   AlgorithmIdentifier,
        signatureValue       BIT STRING  }

    TBSCertificate  ::=  SEQUENCE  {
        version         [0]  EXPLICIT Version DEFAULT v1,
        serialNumber         CertificateSerialNumber,
        signature            AlgorithmIdentifier,
        issuer               Name,
        validity             Validity,
        subject              Name,
        subjectPublicKeyInfo SubjectPublicKeyInfo,
        issuerUniqueID  [1]  IMPLICIT UniqueIdentifier OPTIONAL,
                             -- If present, version MUST be v2 or v3
        subjectUniqueID [2]  IMPLICIT UniqueIdentifier OPTIONAL,
                             -- If present, version MUST be v2 or v3
        extensions      [3]  EXPLICIT Extensions OPTIONAL
                             -- If present, version MUST be v3
        }

   Version  ::=  INTEGER  {  v1(0), v2(1), v3(2)  }

   CertificateSerialNumber  ::=  INTEGER

   Validity ::= SEQUENCE {
        notBefore      Time,
        notAfter       Time }

   Time ::= CHOICE {
        utcTime        UTCTime,
        generalTime    GeneralizedTime }

   UniqueIdentifier  ::=  BIT STRING

   SubjectPublicKeyInfo  ::=  SEQUENCE  {
        algorithm            AlgorithmIdentifier,
        subjectPublicKey     BIT STRING  }

   Extensions  ::=  SEQUENCE SIZE (1..MAX) OF Extension

   Extension  ::=  SEQUENCE  {
        extnID      OBJECT IDENTIFIER,
        critical    BOOLEAN DEFAULT FALSE,
        extnValue   OCTET STRING  }
```

### SubjectPublicKeyInfo

如上一节所示，SubjectPublicKeyInfo是公钥证书格式X.509的组成部分。SubjectPublicKeyInfo结构使用ASN.1描述，其中使用了椭圆曲线公私钥加密算法的SubjectPublicKeyInfo结构定义在[RFC 5480](https://tools.ietf.org/html/rfc5480)

其结构如下：

```
   SubjectPublicKeyInfo  ::=  SEQUENCE  {
        algorithm            AlgorithmIdentifier,
        subjectPublicKey     BIT STRING
   }

   AlgorithmIdentifier  ::=  SEQUENCE  {
        algorithm   OBJECT IDENTIFIER,
        parameters  ANY DEFINED BY algorithm OPTIONAL
   }
```

可以看到AlgorithmIdentifier也是一个SEQUENCE，其parameters部分取决于algorithm的具体取值。

对不限制的ECC公钥使用算法的场景，algorithm取值：

```
1.2.840.10045.2.1

即： iso(1) member-body(2) us(840) ansi-X9-62(10045) keyType(2) 1
```

在该种类场景下，parameters的定义如下：

```
    ECParameters ::= CHOICE {
        namedCurve         OBJECT IDENTIFIER
    }
```

即parameters指定了ECC公钥所使用的椭圆曲线。其可选的值有：

```
    secp192r1 OBJECT IDENTIFIER ::= {
        iso(1) member-body(2) us(840) ansi-X9-62(10045) curves(3) prime(1) 1 }

    sect163k1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 1 }

    sect163r2 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 15 }

    secp224r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 33 }

    sect233k1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 26 }

    sect233r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 27 }

    secp256r1 OBJECT IDENTIFIER ::= {
        iso(1) member-body(2) us(840) ansi-X9-62(10045) curves(3) prime(1) 7 }

    sect283k1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 16 }

    sect283r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 17 }

    secp384r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 34 }

    sect409k1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 36 }

    sect409r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 37 }

    secp521r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 35 }

    sect571k1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 38 }

    sect571r1 OBJECT IDENTIFIER ::= {
        iso(1) identified-organization(3) certicom(132) curve(0) 39 }
```

algorithm确定后，再来看下subjectPublicKey，对ECC公钥来讲，subjectPublicKey就是ECPoint：

```
    ECPoint ::= OCTET STRING
```

是长度为65字节的OCTET STRING，其中第一个字节代表ECPoint是否经过压缩，如果为0x04，代表没有压缩。剩下的64个字节，前32个字节，表示ECPoint的X坐标，后32个字节表示ECPoint的Y坐标。

OCTET STRING类型的ECPoint在转换为BIT STRING类型的subjectPublicKey时，按照大端字节序转换。

### ECC Public Key Example

我们以一个DER编码的ECC公钥为例，详细剖析一下X.509 ECC公钥的格式。公钥内容如下：

```
0x30 0x59 0x30 0x13 0x06 0x07 
0x2a 0x86 0x48 0xce 0x3d 0x02 
0x01 0x06 0x08 0x2a 0x86 0x48 
0xce 0x3d 0x03 0x01 0x07 0x03 
0x42 0x00 0x04 0x13 0x32 0x8e 
0x0c 0x11 0x8a 0x70 0x1a 0x9e 
0x18 0xa3 0xa9 0xa5 0x65 0xd8 
0x41 0x68 0xce 0x2f 0x5b 0x11 
0x94 0x57 0xec 0xe3 0x67 0x76 
0x4a 0x3f 0xb9 0xec 0xd1 0x15 
0xd0 0xf9 0x56 0x8b 0x15 0xe6 
0x06 0x2d 0x72 0xa9 0x45 0x56 
0x99 0xb0 0x9b 0xb5 0x30 0x90 
0x8d 0x2e 0x31 0x0e 0x95 0x68 
0xcc 0xcc 0x19 0x5c 0x65 0x53 
0xba
```

通过前面的介绍，我们已经知道这是一个ASN.1格式的SubjectPublicKeyInfo的DER编码，是一个TLV类型的二进制数据。现在我们逐层解析下：

```
0x30 (SEQUENCE TAG: SubjectPublicKeyInfo) 0x59 (SEQUENCE LEN=89)
        0x30 (SEQUENCE TAG: AlgorithmIdentifier) 0x13 (SEQUENCE LEN=19)
                0x06 (OID TAG: Algorithm) 0x07 (OID LEN=7)
                        0x2a 0x86 0x48 0xce 0x3d 0x02 0x01 (OID VALUE="1.2.840.10045.2.1": ecPublicKey/Unrestricted Algorithm Identifier)
                0x06 (OID TAG: ECParameters:NamedCurve) 0x08 (OID LEN=8)
                        0x2a 0x86 0x48 0xce 0x3d 0x03 0x01 0x07 (OID VALUE="1.2.840.10045.3.1.7": Secp256r1/prime256v1)
        0x03 (BIT STRING TAG: SubjectPublicKey:ECPoint) 0x42 (BIT STRING LEN=66) 0x00 (填充bit数量为0)
                0x04 (未压缩的ECPoint)
                0x13 0x32 0x8e 0x0c 0x11 0x8a 0x70 0x1a 0x9e 0x18 0xa3 0xa9 0xa5 0x65 0xd8 0x41 0x68 0xce 0x2f 0x5b 0x11 0x94 0x57 0xec 0xe3 0x67 0x76 0x4a 0x3f 0xb9 0xec 0xd1 (ECPoint:X)
                0x15 0xd0 0xf9 0x56 0x8b 0x15 0xe6 0x06 0x2d 0x72 0xa9 0x45 0x56 0x99 0xb0 0x9b 0xb5 0x30 0x90 0x8d 0x2e 0x31 0x0e 0x95 0x68 0xcc 0xcc 0x19 0x5c 0x65 0x53 0xba (ECPoint:Y)
``` 

### Java Code

本节给出使用使用Java来生成ECC公私钥、编码解码ECC公私钥、使用ECC进行签名验签、加密解密相关的示例代码供参考。在Java中使用ECC算法有以下几点需要注意：

* JDK1.7开始内置了ECC公私钥生成、签名验签，但没有实现加密解密，因此需要使用BouncyCastle来做Security Provider；
* 在Java中使用高级别的加解密算法，比如AES使用256bit密钥、ECC使用Secp256r1等需要更新JRE的security policy文件，否则会报类似“Illegal key size or default parameters”这样的错误。具体怎样更换policy文件，可以参考[这里](https://stackoverflow.com/questions/6481627/java-security-illegal-key-size-or-default-parameters)
* 实际项目开发过程中，可能发现有传递给Java的公钥不是完整的X.509 SubjectPublicKeyInfo，比如只传递了一个65字节的ECPoint过来，这种情况可以跟对方沟通清楚所使用的Algorithm以及NamedCurve，补全DER数据后，再使用Java Security库解析。

```
JDK 1.7
依赖：org.bouncycastle:bcprov-jdk15on:1.59

import com.sun.jersey.core.util.Base64;
import java.security.InvalidKeyException;
import java.security.KeyFactory;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.NoSuchProviderException;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.SecureRandom;
import java.security.Security;
import java.security.Signature;
import java.security.SignatureException;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;
import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.codec.binary.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ECCUtils {

  private static final Logger LOGGER = LoggerFactory.getLogger(ECCUtils.class);
  private static final String PROVIDER = "BC";

  private static final byte[] PUB_KEY_TL= new byte[26];

  static {
    Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());

    try {
      KeyPair keyPair = genKeyPair();
      PublicKey publicKeyExample = keyPair.getPublic();
      System.arraycopy(publicKeyExample.getEncoded(), 0, PUB_KEY_TL, 0, 26);
    } catch (Exception e) {
      LOGGER.error("无法初始化算法", e);
    }
  }

  public static KeyPair genKeyPair() throws NoSuchAlgorithmException, NoSuchProviderException {
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("EC", PROVIDER);
    keyPairGenerator.initialize(256, new SecureRandom());
    return keyPairGenerator.generateKeyPair();
  }

  public static String encodePublicKey(PublicKey publicKey) {
    return StringUtils.newStringUtf8(Base64.encode(publicKey.getEncoded()));
  }

  public static PublicKey decodePublicKey(String keyStr)
      throws NoSuchProviderException, NoSuchAlgorithmException {
    byte[] keyBytes = getPubKeyTLV(keyStr);

    X509EncodedKeySpec keySpec = new X509EncodedKeySpec(keyBytes);
    KeyFactory keyFactory = KeyFactory.getInstance("EC", PROVIDER);
    try {
      return keyFactory.generatePublic(keySpec);
    } catch (InvalidKeySpecException e) {
      LOGGER.error("无效的ECC公钥", e);
      return null;
    }
  }

  private static byte[] getPubKeyTLV(String keyStr) {
    byte[] keyBytes = Base64.decode(StringUtils.getBytesUtf8(keyStr));

    if(keyBytes.length == 65) {
      byte[] tlv = new byte[91];
      System.arraycopy(PUB_KEY_TL, 0, tlv, 0, 26);
      System.arraycopy(keyBytes, 0, tlv, 26, 65);
      return tlv;
    }

    return keyBytes;
  }

  public static String encodePrivateKey(PrivateKey privateKey) {
    return StringUtils.newStringUtf8(Base64.encode(privateKey.getEncoded()));
  }

  public static PrivateKey decodePrivateKey(String keyStr)
      throws NoSuchProviderException, NoSuchAlgorithmException {
    byte[] keyBytes = Base64.decode(StringUtils.getBytesUtf8(keyStr));
    PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
    KeyFactory keyFactory = KeyFactory.getInstance("EC", PROVIDER);
    try {
      return keyFactory.generatePrivate(keySpec);
    } catch (InvalidKeySpecException e) {
      LOGGER.error("无效的ECC私钥", e);
      return null;
    }
  }

  public static byte[] encrypt(byte[] content, PublicKey publicKey)
      throws NoSuchPaddingException, NoSuchAlgorithmException, InvalidKeyException, BadPaddingException, IllegalBlockSizeException, NoSuchProviderException {
    Cipher cipher = Cipher.getInstance("ECIES", PROVIDER);
    cipher.init(Cipher.ENCRYPT_MODE, publicKey);
    return cipher.doFinal(content);
  }

  public static byte[] decrypt(byte[] content, PrivateKey privateKey)
      throws NoSuchPaddingException, NoSuchAlgorithmException, InvalidKeyException, BadPaddingException, IllegalBlockSizeException, NoSuchProviderException {
    Cipher cipher = Cipher.getInstance("ECIES", PROVIDER);
    cipher.init(Cipher.DECRYPT_MODE, privateKey);
    return cipher.doFinal(content);
  }

  public static byte[] signature(byte[] content, PrivateKey privateKey)
      throws NoSuchAlgorithmException, InvalidKeyException, SignatureException {
    Signature signature = Signature.getInstance("SHA256withECDSA");
    signature.initSign(privateKey);
    signature.update(content);
    return signature.sign();
  }

  public static boolean verify(byte[] content, byte[] sign, PublicKey publicKey)
      throws NoSuchAlgorithmException, InvalidKeyException {
    Signature signature = Signature.getInstance("SHA256withECDSA");
    signature.initVerify(publicKey);
    try {
      signature.update(content);
      return signature.verify(sign);
    } catch (SignatureException e) {
      LOGGER.warn("无效的签名", e);
      return false;
    }

  }
}
```

总结：密码学相关的标准、协议很多，原理往往需要一些数学基础。想要程序马上work起来可能容易，想要搞清楚原理，需要花些时间才行。