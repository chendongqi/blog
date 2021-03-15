---
layout:      post
title:      "系统升级系列三"
subtitle:   "签名"
navcolor:   "invert"
date:       2018-12-20
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - System Upgrade
    - OTA
    - 签名
---


这篇承接[系统升级系列二 系统升级包介绍](http://chendongqi.me/2018/12/17/UpdatePackages/)，从升级包的角度来详细了解下涉及到的签名相关知识点。  

### MANIFEST.MF

我们先来看下MANIFEST.MF，它是这个环节中的第一步。它的文件内容部分如下  

```xml
Manifest-Version: 1.0
Created-By: 1.0 (Android SignApk)

Name: system/lib/lib_omx_rtps_pipe_arm11_elinux.so
SHA1-Digest: B0ue12ES7ARHUqKOBXRd/PhJpLU=

Name: system/vendor/lib/libijkplayer.so
SHA1-Digest: xaDhy0KXn7gCusAajweldA0StUA=

Name: system/framework/webview/paks/nl.pak
SHA1-Digest: 9HkaDmNqPTx3g/oUEqLfRBYzjyQ=

Name: system/bin/zteconshell.sh
SHA1-Digest: T7CBWtbYANfc8Io52SMljQK7J4M=

Name: system/lib/libgnsdk_link.3.07.3.so
SHA1-Digest: iUYsCkfe9aBoUR8Z3PbC0XzrJCQ=
```

这是一个manifest文件，首先是版本号，然后是创建者，这里是Android SignApk，后面都是一样，包括了这个升级包中处理META-INF目录下的所有文件，第一行是文件名，第二行是SHA1-Digest，生成的规则是对此文件做sha1计算hash值，然后将hash值再做base64编码。这个过程是在signapk里通过代码执行的，当然这个规则涉及到的hash值和base64编码我们都可以手动计算，那么我们以“system/lib/lib_omx_rtps_pipe_arm11_elinux.so”这一项为例来做一次手动计算，看看结果是否相符。  

首先对升级包中的system/lib/lib_omx_rtps_pipe_arm11_elinux.so文件计算hash值，通过linux的sha1sum命令，计算结果为  

```xml
sha1sum system/lib/lib_omx_rtps_pipe_arm11_elinux.so
074b9ed76112ec044752a28e05745dfcf849a4b5  system/lib/lib_omx_rtps_pipe_arm11_elinux.so
```

然后是对hash值做base64编码，网上有很多在线转的工具，测试了结果都不行，还有一些是exe的转换工具，但是是直接将hash数据做转换，结果跟在线工具一样，都是因为输入的编码不正确。sha1sum生成的hash值是十六进制的，我这边用了window下的WinHex编辑器来创建文件，然后把hash值粘贴进去，注意选择编码格式为“ASCII Hex”  

![WinHex-FormatChoose.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/WinHex-FormatChoose.png)  

![WinHex-ManifestData.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/WinHex-ManifestData.png)  

将此文件保存之后，我们就开始祭出base64编码工具了，这里用到的也是在网上找到的，能够对文件中的十六进制hash值做编码的工具。  

![Base64-ManifestData.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/Base64-ManifestData.png)    

这里的文件“333”就是刚在WinHex里保存的文件名，而“222.txt”可以随便新建一个文本文档，用来工具输出编码结果的，完成后就可以打开来查看结果。  

看到“222.txt”中记录的结果为  

![ManifestDataResult.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/ManifestDataResult.png)    

对比MANIFEST.MF文件中的结果，是一样的。  

我们这里再同步叙述一条支线，签名系统就是为了防止apk或者升级包被篡改，那么我们站到一个攻击者的角度来思考下篡改MANIFEST.MF文件的可能性，通过上面手动操作的这一套，就应该知道了篡改是可行的。只要修改了包里的某个文件，然后将它的sha1值重新做一遍base64编码就可以了。

### CERT.SF

CERT.SF文件是在MANIFEST.MF的基础上生成的，网上有这样一种说法

> 这是对摘要的签名文件。对前一步生成的MANIFEST.MF，使用SHA1-RSA算法，用开发者的私钥进行签名。在安装时只能使用公钥才能解密它。解密之后，将它与未加密的摘要信息（即，MANIFEST.MF文件）进行对比，如果相符，则表明内容没有被异常修改。

初看的时候被这种自圆其说的方式给说服了，直到后来看到了一种质疑，说是用不同的密钥对apk进行签名，查看生成的CERT.SF是一样的。然后我自己测试了一次，确如其说结果是一样的，再后来终于找到了正解，还是感慨国内论坛人云亦云的现象太严重了，绝知此事要躬行。

> https://stackoverflow.com/questions/4245303/android-sf-file
>
> The digests in the .SF file are computed by hashing the 3 lines of the 
> corresponding entry in the .MF file. The .RSA (or .DSA) file contains a 
> signature of the .SF file created from the signing private key, along 
> with the public certificate chain of the signing key. The .RSA (or .DSA)
> file is in a binary (i.e. non-human readable) format that can be 
> programmatically parsed with effort.

上面这段我们也可以用手动的方式来操作下，先来看下CERT.SF的部分内容  

```xml
Signature-Version: 1.0
Created-By: 1.0 (Android SignApk)
SHA1-Digest-Manifest: oCY/jG+8Kr43FXyMzOMLU52VeBM=

Name: system/lib/lib_omx_rtps_pipe_arm11_elinux.so
SHA1-Digest-Manifest: pFDu2GJrNQuqRw2KxCCOTAz0fJw=

Name: system/vendor/lib/libijkplayer.so
SHA1-Digest-Manifest: LxzRx6uB+RlVc+x6dy/KiHNwdLQ=

Name: system/framework/webview/paks/nl.pak
SHA1-Digest-Manifest: 8Qxi3iVzG4DBW3NaoWUEPdeQ3sU=

Name: system/bin/zteconshell.sh
SHA1-Digest-Manifest: 43UDRuC4A/eDXFeetfFS4994A3E=
```

它是一个签名文件，第一项里包含了签名版本、创建者和对MANIFEST.MF文件的加密信息，此信息的来源是通过对前面MANIFEST.MF文件用sha1sum做hash值计算，然后通过上面的WinHex和base64编码工具对此hash值做计算生成base64编码，结果就是这里的“oCY/jG+8Kr43FXyMzOMLU52VeBM=”  

后面的每一项跟MANIFEST.MF里的一一对应，我们挑第一项来模拟下SHA1-Digest-Manifest的生成。  

把MANIFEST.MF中的第一项复制到一个文本文件中

```xml
Name: system/lib/lib_omx_rtps_pipe_arm11_elinux.so
SHA1-Digest: B0ue12ES7ARHUqKOBXRd/PhJpLU=


```

注意后面要加两个“\r\n”，我开始是在linux下测试的，总是发现结果不对，后来查询了资料说是linux下将回车键按"\n"处理，所以文本文件计算出来的hash值始终不对。后来移步到windows下用文本工具就成功了。接下来就跟之前的一样了，计算hash值，然后计算base64编码，生成的结果就是“pFDu2GJrNQuqRw2KxCCOTAz0fJw=”。  

那么我们再来思考下CERT.SF是否能被攻破呢？在修改了MANIFEST.MF之后，手动修改CERT.SF应该也不是难事对吧。  

### CERT.RSA

CERT.RSA文件是生成的包含签名信息的文件，来分析此文件之前先了解下是如何签名跟验证的。  

![ManifestDataResult.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/ManifestDataResult.png)    

签名过程：对所有数据进行摘要计算（这个可能会有多种形式，而这里所谈论的摘要指的也就是CERT.SF），然后用私钥对CERT.SF进行加密，同时将签名证书信息也一起写入，生成一个加密后的数据文件，这里指的就是CERT.RSA。  

验证过程：对CERT.RSA通过公钥解密，解密结果和摘要信息（CERT.SF）对比，如果结果一致则验证通过。  

用到的密钥对可以是自己用keytool生成的，Android签名中证书的格式采用X.509标准的版本三，一对秘钥包括了私钥（.pk8）和公钥（x59.pem），该证书的格式标准如下  

![X509CA.jpeg](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/X509CA.jpeg)  

可以用命令```keytool -printcert -file <*.x59.pem>```来看到公钥的信息，这里包含的是证书的基本信息。  

私钥的内容我们可以通过命令```openssl pkcs8 -in build/target/product/security/releasekey.pk8 -inform DER -nocrypt```来看到  



然后来看一下签名之后的CERT.SF文件的内容，可以用命令```openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs -text```来查看到其中包含的内容，是符合标准格式的  

我们用十六进制编辑器来打开CERT.RSA（我这边用notpad++安装HEX-Editor来打开），可以看到中间包含了公钥和加密的信息。  

![PublicKey.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/PublicKey.png)   

![EncryptedData.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/EncryptedData.png)   

其实也就是说我们用命令查看到证书和加密信息，都是可以通过读取二进制文件的内容通过特定的offset来解析得到的，这个给用程序来获取CERT.SF提供了一个思路。  

那么我们再来谈论一件事情，Android系统在安装apk的时候，是如何进行对签名校验的呢？涉及到了PackageParser.java和JarVerifier.java简单来说分成如下几个步骤（对于系统签名来说）：  

1、读取DSA/RSA/EC后缀的签名证书文件（在/system/etc/下），然后调用verifyCertificate进行难，此处并无限制文件名，因此将CERT.RSA改成CERT123.RSA依然有效，但SF文件得跟RSA文件等签名证书文件同名；  

2、读取SF（与后面证书同名）与RSA/DSA/EC文件的内容，再调用verifySignature进行校验；

3、在verifySignature进行校验时会去读取各项证书信息，包括证书序列号、拥有者、加密算法等等，然后判断CERT.RSA证书对CERT.SF的文件签名是否正确，防止CERT.SF被篡改，若成功则返回证书链，否则抛出异常；

4、在返回证书链时，对于自签名证书，则直接作为合法证书返回；

5、继续回到verifyCertificate函数，它会校验SF文件中的MANIFEST.MF中各项Hash值是否正确，防止MF文件被篡改；

而对于非系统应用，也就说第三方应用来说，去枚举除META-INF目录以外的所有文件，然后进行哈希运算，并将其与MANIFEST.MF中的各文件哈希值进行比对，只有相匹配后才允许安装应用。也就是说只校验文件的完整性，不校验签名也就不校验apk的颁发者。  

那么我们继续站在攻击者的角度来思考下有了CERT.RSA之后，此apk或者升级包是否还能被攻破？按照攻击的思路，我们先修改MANIFEST.MF，然后修改CERT.SF，接下来就要用私钥对CERT.SF加密生成CERT.RSA了，因为我们无法拿到开发者的私钥，假设用自己的密钥对进行了签名。那么在系统端进行校验时，使用开发者的公钥对经过我们自己私钥加密的CERT.RSA进行解密，解出来的信息跟CERT.SF中的肯定不同，所以校验失败，这么来说确实是保护了文件不被篡改。那么在拿不到开发者秘钥对的前提下，我能想到的一个方式就是在校验阶段用自己的公钥去解密CERT.RSA了，再CERT.RSA里本来就包含了公钥信息，那么如何让校验程序能使用到我们的公钥呢？替换系统中的公钥？system分区是只读的，先root？做坏事还真的比较麻烦的一件事，不过挺有意思。  

前面提到了第三方程序的签名校验是不严格的，这里延伸出一个问题，那么这个签名信息还能用来做什么？能想到的一点时，我们如何去判断一个第三方apk是不是官方发布的，或者说在现在有那么多应用市场的情况下，我们如何来判断apk的来源。那么就可以利用到RSA里包含的公钥信息了。我们可以将公钥信息和官方发布的公钥或是应用市场的公钥做比对来确定来源。  

升级包的签名校验过程会在后续文章中专门介绍。  

### otacert

如果-w整包签，则将证书.x509.pem复制到META-INF/com/android/otacert，并在manifest对象中增加META-INF/com/android/otacert的SHA1摘要  

```xml
Name: META-INF/com/android/otacert
SHA1-Digest: N20+op0WrexaAIOsPtpmeD/hWRQ=
```

我们从android源码编译的日志中可以印证  

```xml
running:  java -Xmx2048m -jar out/host/linux-x86/framework/signapk.jar -w build/target/product/security/releasekey.x509.pem build/target/product/security/releasekey.pk8 /tmp/tmpgP48vj out/target/product/xe110bm/xe110bm-update-ota-?.?.?-?-eng.zip
```

确实在make otapackage的时候签名是有-w的参数，那么这个编译命令是在哪儿加的？otacert公钥后续有什么用？我们在制作OTA包的篇章和升级包校验的时候来探究。  

为了验证otacert确实是对应的x59.pem复制过去的，对生成的升级包中的otacert和源码下用到的签名公钥做了对比，确实一致。  

![OtacertWithPem.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/OtacertWithPem.png)   

### Signapk

前面通过手动的方式确认了MANIFEST.MF和CERT.SF是如何生成的，那么我们看到的文件是签名工具（Android这里用的是Signapk），那么我们来从代码看下SignApk的签名过程来印证下。代码位于源码下build/tools/signapk/SignApk.java中。  

```java
	public static void main(String[] args) {
    	...
        //参数带-w，全局签名置为true
        if (args[0].equals("-w")) {
            signWholeFile = true;
            argstart = 1;
        }
    	...
		if (signWholeFile) {
            // 全局签名，后面代码跳转到CMSSigner
            SignApk.signWholeFile(inputJar, firstPublicKeyFile,
                                  publicKey[0], privateKey[0], outputFile);
        } else {
            JarOutputStream outputJar = new JarOutputStream(outputFile);

            // For signing .apks, use the maximum compression to make
            // them as small as possible (since they live forever on
            // the system partition).  For OTA packages, use the
            // default compression level, which is much much faster
            // and produces output that is only a tiny bit larger
            // (~0.1% on full OTA packages I tested).
            outputJar.setLevel(9);//压缩等级，可以自行测试下时间和空间的关系
			// 正常的签名
            // 1.生成MANIFEST.MF文件
            Manifest manifest = addDigestsToManifest(inputJar, hashes);
            // 2.生成新的压缩包，并将文件拷贝进去
            copyFiles(manifest, inputJar, outputJar, timestamp);
            // 3.对新包进行签名
            signFile(manifest, inputJar, publicKey, privateKey, outputJar);
            outputJar.close();
        }
	}
```

```java
	/**
     * Add the hash(es) of every file to the manifest, creating it if
     * necessary.
     */
	// 生成全局文件摘要到MANIFEST.MF文件
    private static Manifest addDigestsToManifest(JarFile jar, int hashes)
        throws IOException, GeneralSecurityException {
        Manifest input = jar.getManifest();
        Manifest output = new Manifest();
        Attributes main = output.getMainAttributes();
        if (input != null) {
            main.putAll(input.getMainAttributes());
        } else {
            main.putValue("Manifest-Version", "1.0");
            main.putValue("Created-By", "1.0 (Android SignApk)");
        }

        MessageDigest md_sha1 = null;
        MessageDigest md_sha256 = null;
        if ((hashes & USE_SHA1) != 0) {
            md_sha1 = MessageDigest.getInstance("SHA1");
        }
        if ((hashes & USE_SHA256) != 0) {
            md_sha256 = MessageDigest.getInstance("SHA256");
        }

        byte[] buffer = new byte[4096];
        int num;

        // We sort the input entries by name, and add them to the
        // output manifest in sorted order.  We expect that the output
        // map will be deterministic.

        TreeMap<String, JarEntry> byName = new TreeMap<String, JarEntry>();

        for (Enumeration<JarEntry> e = jar.entries(); e.hasMoreElements(); ) {
            JarEntry entry = e.nextElement();
            byName.put(entry.getName(), entry);
        }

        for (JarEntry entry: byName.values()) {
            String name = entry.getName();
            if (!entry.isDirectory() &&
                (stripPattern == null || !stripPattern.matcher(name).matches())) {
                InputStream data = jar.getInputStream(entry);
                while ((num = data.read(buffer)) > 0) {
                    if (md_sha1 != null) md_sha1.update(buffer, 0, num);
                    if (md_sha256 != null) md_sha256.update(buffer, 0, num);
                }

                Attributes attr = null;
                if (input != null) attr = input.getAttributes(name);
                attr = attr != null ? new Attributes(attr) : new Attributes();
                if (md_sha1 != null) {
                    attr.putValue("SHA1-Digest",
                                  new String(Base64.encode(md_sha1.digest()), "ASCII"));
                }
                if (md_sha256 != null) {
                    attr.putValue("SHA-256-Digest",
                                  new String(Base64.encode(md_sha256.digest()), "ASCII"));
                }
                output.getEntries().put(name, attr);
            }
        }

        return output;
    }
```

```java
	// 签名文件
	private static void signFile(Manifest manifest, JarFile inputJar,
                                 X509Certificate[] publicKey, PrivateKey[] privateKey,
                                 JarOutputStream outputJar)
        throws Exception {
        // Assume the certificate is valid for at least an hour.
        long timestamp = publicKey[0].getNotBefore().getTime() + 3600L * 1000;

        // MANIFEST.MF
        JarEntry je = new JarEntry(JarFile.MANIFEST_NAME);
        je.setTime(timestamp);
        outputJar.putNextEntry(je);
        manifest.write(outputJar);

        int numKeys = publicKey.length;
        for (int k = 0; k < numKeys; ++k) {
            // CERT.SF / CERT#.SF
            je = new JarEntry(numKeys == 1 ? CERT_SF_NAME :
                              (String.format(CERT_SF_MULTI_NAME, k)));
            je.setTime(timestamp);
            outputJar.putNextEntry(je);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            // 生成CERT.SF
            writeSignatureFile(manifest, baos, getAlgorithm(publicKey[k]));
            byte[] signedData = baos.toByteArray();
            outputJar.write(signedData);

            // CERT.RSA / CERT#.RSA
            je = new JarEntry(numKeys == 1 ? CERT_RSA_NAME :
                              (String.format(CERT_RSA_MULTI_NAME, k)));
            je.setTime(timestamp);
            outputJar.putNextEntry(je);
            // 生成CERT.RSA
            writeSignatureBlock(new CMSProcessableByteArray(signedData),
                                publicKey[k], privateKey[k], outputJar);
        }
    }
```

```java
	/** Write a .SF file with a digest of the specified manifest. */
	// 生成CERT.SF文件
    private static void writeSignatureFile(Manifest manifest, OutputStream out,
                                           int hash)
        throws IOException, GeneralSecurityException {
        Manifest sf = new Manifest();
        Attributes main = sf.getMainAttributes();
        main.putValue("Signature-Version", "1.0");
        main.putValue("Created-By", "1.0 (Android SignApk)");

        MessageDigest md = MessageDigest.getInstance(
            hash == USE_SHA256 ? "SHA256" : "SHA1");
        PrintStream print = new PrintStream(
            new DigestOutputStream(new ByteArrayOutputStream(), md),
            true, "UTF-8");

        // Digest of the entire manifest
        manifest.write(print);
        print.flush();
        main.putValue(hash == USE_SHA256 ? "SHA-256-Digest-Manifest" : "SHA1-Digest-Manifest",
                      new String(Base64.encode(md.digest()), "ASCII"));

        Map<String, Attributes> entries = manifest.getEntries();
        for (Map.Entry<String, Attributes> entry : entries.entrySet()) {
            // Digest of the manifest stanza for this entry.
            print.print("Name: " + entry.getKey() + "\r\n");
            for (Map.Entry<Object, Object> att : entry.getValue().entrySet()) {
                print.print(att.getKey() + ": " + att.getValue() + "\r\n");
            }
            print.print("\r\n");
            print.flush();

            Attributes sfAttr = new Attributes();
            sfAttr.putValue(hash == USE_SHA256 ? "SHA-256-Digest" : "SHA1-Digest-Manifest",
                            new String(Base64.encode(md.digest()), "ASCII"));
            sf.getEntries().put(entry.getKey(), sfAttr);
        }

        CountOutputStream cout = new CountOutputStream(out);
        sf.write(cout);

        // A bug in the java.util.jar implementation of Android platforms
        // up to version 1.6 will cause a spurious IOException to be thrown
        // if the length of the signature file is a multiple of 1024 bytes.
        // As a workaround, add an extra CRLF in this case.
        if ((cout.size() % 1024) == 0) {
            cout.write('\r');
            cout.write('\n');
        }
    }
```

```java
	private static class CMSSigner implements CMSTypedData {
        private JarFile inputJar;
        private File publicKeyFile;
        private X509Certificate publicKey;
        private PrivateKey privateKey;
        private String outputFile;
        private OutputStream outputStream;
        private final ASN1ObjectIdentifier type;
        private WholeFileSignerOutputStream signer;

        public CMSSigner(JarFile inputJar, File publicKeyFile,
                         X509Certificate publicKey, PrivateKey privateKey,
                         OutputStream outputStream) {
            this.inputJar = inputJar;
            this.publicKeyFile = publicKeyFile;
            this.publicKey = publicKey;
            this.privateKey = privateKey;
            this.outputStream = outputStream;
            this.type = new ASN1ObjectIdentifier(CMSObjectIdentifiers.data.getId());
        }

        public Object getContent() {
            throw new UnsupportedOperationException();
        }

        public ASN1ObjectIdentifier getContentType() {
            return type;
        }

       	// 写新包
        public void write(OutputStream out) throws IOException {
            try {
                signer = new WholeFileSignerOutputStream(out, outputStream);
                JarOutputStream outputJar = new JarOutputStream(signer);

                int hash = getAlgorithm(publicKey);

                // Assume the certificate is valid for at least an hour.
                long timestamp = publicKey.getNotBefore().getTime() + 3600L * 1000;

                Manifest manifest = addDigestsToManifest(inputJar, hash);
                copyFiles(manifest, inputJar, outputJar, timestamp);
                // -w的方式会添加otacert
                addOtacert(outputJar, publicKeyFile, timestamp, manifest, hash);

                signFile(manifest, inputJar,
                         new X509Certificate[]{ publicKey },
                         new PrivateKey[]{ privateKey },
                         outputJar);

                signer.notifyClosing();
                outputJar.close();
                signer.finish();
            }
            catch (Exception e) {
                throw new IOException(e);
            }
        }

        public void writeSignatureBlock(ByteArrayOutputStream temp)
            throws IOException,
                   CertificateEncodingException,
                   OperatorCreationException,
                   CMSException {
            SignApk.writeSignatureBlock(this, publicKey, privateKey, temp);
        }

        public WholeFileSignerOutputStream getSigner() {
            return signer;
        }
    }
```

从上面的代码中可以看出Signapk签名的步骤就是从新的压缩包里生成摘要文件MANIFEST.MF，然后根据生成CERT.SF，再根据秘钥对生成CERT.RSA，将这些写入到新的压缩包里，如果带-w参数，则拷贝otacert公钥。  

另外，全局签名在文件标记上还有点区别，这里暂时不做展开，可以参考[Android签名与校验过程详解](https://blog.csdn.net/gulinxieying/article/details/78677487)

后续再做研究。  