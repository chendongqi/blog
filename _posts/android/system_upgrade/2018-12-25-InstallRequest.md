---
layout:      post
title:      "系统升级系列五"
subtitle:   "请求升级"
navcolor:   "invert"
date:       2018-12-25
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - System Upgrade
    - OTA
    - 升级请求
---

### 1 前言

这一片是接系列二，我们准备好了升级包之后，先跳过部署环节，假设升级包已经下载到了本地，或者是将升级包拷贝到U盘，拷贝到设备本地存储等方式，也就是系列一里讲到的卡刷模式。 

### 2 升级app

用户UE是从升级app发起的，一般在android里执行升级OTA升级的入口是在设置里，当然我们也可以写自己的升级程序。一个完整的OTA升级的apk，框架就是检查更新->下载升级包->校验->安装，我们跳过在线检查和下载的部分，来看下本地升级会用到校验和安装，就是为了引出后续framework层的流程。直接贴一段app的核心代码  

MainActivity.java  

```java
mBtnUpgrade.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        try {
            RecoverySystem.verifyPackage(new File("/mnt/udisk/update/update.zip"), null, null);
        } catch (IOException e) {
            e.printStackTrace();
            return;
        } catch (GeneralSecurityException e) {
            e.printStackTrace();
            return;
        }

        Log.d(TAG, "install go go go!!!");

        try {
            RecoverySystem.installPackage(mContext, new File("/mnt/udisk/update/update.zip"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
});
```

 AndroidManifest.xml  

```xml
<uses-permission android:name="android.permission.RECOVERY" />
<uses-permission android:name="android.permission.ACCESS_CACHE_FILESYSTEM" />
```

引出了两个关键的方法RecoverySystem.verifyPackage和RecoverySystem.installPackage  

### 3 RecoverySystem

framework里写了RecoverySystem这个类来提供跟升级、恢复出厂设置相关的功能，第一个重要的方法就是verifyPackage，它是用来校验文件的签名有效性的，在进入verifyPackage之前，我们必须要了解一个被签名后的zip包是什么样的格式，什么位置存储了什么信息，这个就必须要通过signapk的代码去看签名时都写入了什么内容。而要看懂签名时是如何写文件的，我们又得了解一个没有签名注释的zip包是什么样的格式。所以在正式介绍verifyPackage之前我写了两节关于zip文件格式和signapk.java里的signWholeFile方法。  

#### 3.1 zip文件格式

这部分参考了[zip文件格式](https://blog.csdn.net/xiaobing1994/article/details/78367035)  
zip文件格式由文件数据区、中央目录结构,中央目录结束标志组成。其中中央目录结束节又有一个字段保存了中央目录结构的偏移。整体结构如下图   

 ![zip文件格式](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/Zip-File-Format.png)   

中央目录结束标示，在signapk和verifyPackage里都会出现end-of-central-directory（EOCD），指的就是这段结构体数据，而且会提到是“For a zip with no archive comment”除去char * elComment共22个字节      

```cpp
struct EndLocator { 
    ui32 signature; //目录结束标记,(固定值0x504b0506)   
    
    ui16 elDiskNumber; //当前磁盘编号   
    
    ui16 elStartDiskNumber; //中央目录开始位置的磁盘编号   
    
    ui16 elEntriesOnDisk; //该磁盘上所记录的核心目录数量   
    
    ui16 elEntriesInDirectory; //中央目录结构总数   
    
    ui32 elDirectorySize; //中央目录的大小   
    
    ui32 elDirectoryOffset; //中央目录开始位置相对于文件头的偏移   
    
    ui16 elCommentLen; // 注释长度   
    
    char *elComment; // 注释内容   
    
};
```

我们通过二进制文件查看工具（我这里在windows下用到了notepad++配合Hex-Editor插件，在系列二中剖析升级包结构时也用到了）来打开未签名的压缩包。可以看到文件尾部是   

 ![未签名的EOCD](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/EOCD-unsigned.png) 

可以看到确实是文件尾部22个字节，我们来跟结构体对应下  

```cpp
struct EndLocator { 
    ui32 signature; //目录结束标记,(固定值0x504b0506)   
    
    ui16 elDiskNumber; //当前磁盘编号（0x0000）   
    
    ui16 elStartDiskNumber; //中央目录开始位置的磁盘编号（0x0000）   
    
    ui16 elEntriesOnDisk; //该磁盘上所记录的核心目录数量（0x1000）   
    
    ui16 elEntriesInDirectory; //中央目录结构总数（0x1000）   
    
    ui32 elDirectorySize; //中央目录的大小 （0xf2050000）  
    
    ui32 elDirectoryOffset; //中央目录开始位置相对于文件头的偏移 （0xadff0200）  
    
    ui16 elCommentLen; // 注释长度 （0x0000）  
    
    // char *elComment; // 注释内容 （没有这个数据）  
    
};
```

中央目录结构位于文件数据区后，它用来保存所有文件的路径信息，和对应文件数据结构区在文件中的偏移。该结构具体如下：  

```cpp
struct DirEntry { 
    ui32 signature; // 中央目录文件header标识（0x504b0102）   
    
    ui16 deVersionMadeBy; // 压缩所用的pkware版本   
    
    ui16 deVersionToExtract; // 解压所需pkware的最低版本   
    
    ui16 deFlags; // 通用位标记   
    
    ui16 deCompression; // 压缩方法   
    
    ui16 deFileTime; // 文件最后修改时间   
    
    ui16 deFileDate; // 文件最后修改日期   
    
    ui32 deCrc; // CRC-32校验码   
    
    ui32 deCompressedSize; // 压缩后的大小   
    
    ui32 deUncompressedSize; // 未压缩的大小   
    
    ui16 deFileNameLength; // 文件名长度   
    
    ui16 deExtraFieldLength; // 扩展域长度   
    
    ui16 deFileCommentLength; // 文件注释长度   
    
    ui16 deDiskNumberStart; // 文件开始位置的磁盘编号   
    
    ui16 deInternalAttributes; // 内部文件属性   
    
    ui32 deExternalAttributes; // 外部文件属性   
    
    ui32 deHeaderOffset; // 本地文件头的相对位移   
    
    char *deFileName; // 目录文件名   
    
    char *deExtraField; // 扩展域   
    
    char *deFileComment; // 文件注释内容   
    
};
```

这部分跟签名不相关，我们这里也不做关注。

文件数据区是保存所有压缩文件数据的区，它位于文件头，并由压缩数据结构的数组组成。其结构如下：  

```cpp  
struct Record { 
    ui32 signature; // 文件头标识，值固定(0x504b0304)  
    
    ui16 frVersion; // 解压文件所需 pkware最低版本   
    
    ui16 frFlags; // 通用比特标志位(置比特0位=加密)   
    
    ui16 frCompression; // 压缩方式   
    
    ui16 frFileTime; // 文件最后修改时间   
    
    ui16 frFileDate; //文件最后修改日期   
    
    ui32 frCrc; // CRC-32校验码   
    
    ui32 frCompressedSize; //  压缩后的大小   
    
    ui32 frUncompressedSize; // 未压缩的大小   
    
    ui16 frFileNameLength; //  文件名长度   
    
    ui16 frExtraFieldLength; // 扩展区长度   
    
    char* frFileName; // 文件名   
    
    char* frExtraField; // 扩展区   
    
    char* frData; // 压缩数据   
    
};
```

这个对签名也不相关，我们这里也不做关注。  

#### 3.2 signWholeFile  

```java
private static void signWholeFile(JarFile inputJar, File publicKeyFile,
                                      X509Certificate publicKey, PrivateKey privateKey,
                                      OutputStream outputStream) throws Exception {
        CMSSigner cmsOut = new CMSSigner(inputJar, publicKeyFile,
                                         publicKey, privateKey, outputStream);

        ByteArrayOutputStream temp = new ByteArrayOutputStream();

        // put a readable message and a null char at the start of the
        // archive comment, so that tools that display the comment
        // (hopefully) show something sensible.
        // TODO: anything more useful we can put in this message?
        // 直接添加评论信息，参考后面签名后的压缩包二进制信息，可以看到在原先的EOCD后面多了签名信息
      	// 并且ECOD里的elCommentLen也发生了改变  
        byte[] message = "signed by SignApk".getBytes("UTF-8");
    	// 添加日志打印： message length = 17。没有算字符串的结束符\0
        System.out.printf("signWholeFile message.length = %d\n", message.length);
        temp.write(message);
        temp.write(0);
    	// 添加日志打印： temp size = 18。多了一个0，ASCII=48，可以从二进制文件里看到在字符串结束符00后面多了30(十六进制的48)
    	System.out.printf("signWholeFile temp size = %d\n", temp.size());

    	// 将签名写到comment后面
        cmsOut.writeSignatureBlock(temp);

        byte[] zipData = cmsOut.getSigner().getTail();

        // For a zip with no archive comment, the
        // end-of-central-directory record will be 22 bytes long, so
        // we expect to find the EOCD marker 22 bytes from the end.
        // 从未签名的原始压缩文件中读到EOCD的内容，去判断下signature是否是固定的50400506，如果不是的话则抛出异常，说明这个文件已经被签过名了
    	// 实操一下，对一个已经签过名的zip包再去签名就会报错，重签的话需要把META-INF先删除
        if (zipData[zipData.length-22] != 0x50 ||
            zipData[zipData.length-21] != 0x4b ||
            zipData[zipData.length-20] != 0x05 ||
            zipData[zipData.length-19] != 0x06) {
            throw new IllegalArgumentException("zip data already has an archive comment");
        }

    	// temp = message + 0x00，实际是19个byte
    	// 这里为什么要+6，因为后面还需要6个字节来存储签名开始(两个字节)，comment总字节数(两个字节)，
        int total_size = temp.size() + 6;
    	// 添加日志打印： total size = 1537
    	System.out.printf("signWholeFile total_size = %d\n", total_size);
        if (total_size > 0xffff) {
            throw new IllegalArgumentException("signature is too big for ZIP file comment");
        }
        // signature starts this many bytes from the end of the file
    	// 这里signature_start = 1519，也就是说从文件尾倒数1519个字节就是签名开始的地方
        int signature_start = total_size - message.length - 1;
    	// int型的数据这里占两个字节，temp是byte数组，那么怎么存呢？
    	// 将int型的通过和0xff求与计算，拿到低八位，右移8位之后再跟0xff求与计算拿到高八位
    	// 从日志可以看到这里低八位是0xef，高八位是0x5（换算回十进制就是5*16*16+14*16+15=1519，bingo！），那这个方法记录的数据后面就可以为签名校验时拿到签名的内容了
        temp.write(signature_start & 0xff);
        temp.write((signature_start >> 8) & 0xff);
        // Why the 0xff bytes?  In a zip file with no archive comment,
        // bytes [-6:-2] of the file are the little-endian offset from
        // the start of the file to the central directory.  So for the
        // two high bytes to be 0xff 0xff, the archive would have to
        // be nearly 4GB in size.  So it's unlikely that a real
        // commentless archive would have 0xffs here, and lets us tell
        // an old signed archive from a new one.
    	// 这里为什么写入两个0xff，功能应该是为了分割签名开始位置数据和comment总字节数，为什么可以用两个0xff来做呢？上面注释里说了，因为zip包大小不太可能接近4GB，所以这个位置不太可能出现两个0xff。
    	// 具体怎么理解呢？因为在一个未签名的zip里，最末22个字节是EOCD，而[-6:-2]的位置elDirectoryOffset标识了中央目录距离文件头的偏移，如果这里的[-3:-2]也就是elDirectoryOffset高位两个字节出现0xff，那么这个zip包大小接近4G，这里说这种情况不太可能出现。所以用这个来作为签名后，最末6个字节中的特殊标示，以区分这6个字节的独特性。  
        temp.write(0xff);
        temp.write(0xff);
    	// 有了上面存储signature_start的部分，这里就很好理解了，存储了整个签名加comment加额外的6个字节的总字节数
        temp.write(total_size & 0xff);//这里低八位0x1
        temp.write((total_size >> 8) & 0xff);//这里高八位0x6，换算回十进制刚好就是1537
        temp.flush();

        // Signature verification checks that the EOCD header is the
        // last such sequence in the file (to avoid minzip finding a
        // fake EOCD appended after the signature in its scan).  The
        // odds of producing this sequence by chance are very low, but
        // let's catch it here if it does.
    	// 来检查下temp整段数据里是不是包含了一个假冒的EOCD标示504b0505
        byte[] b = temp.toByteArray();
        for (int i = 0; i < b.length-3; ++i) {
            if (b[i] == 0x50 && b[i+1] == 0x4b && b[i+2] == 0x05 && b[i+3] == 0x06) {
                throw new IllegalArgumentException("found spurious EOCD header at " + i);
            }
        }

    	// 将注释内容长度输出升级包的ECOD的elCommentLen字段
        outputStream.write(total_size & 0xff);
        outputStream.write((total_size >> 8) & 0xff);
    	// 将完整的注释内容写到输出包中
        temp.writeTo(outputStream);
    }
```

所以签名后的zip文件格式变成了  

![ZipFileWithComment](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/ZipFileWithComment.png)   

打开签名后的升级包的二进制文件，可以看到对比前面未签名的EOCD数据，最后的elCommentLen变成了0x0106，也就是signWholeFile最后写入的  

```java  
outputStream.write(total_size & 0xff);
outputStream.write((total_size >> 8) & 0xff);
```

![EOCD-signed](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/EOCD-signed.png)  

而签名后的文件最末可以看到是  

![EndOfSignedZip](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/EndOfSignedZip.png)   

也就是附加的6个字节的内容，记录了签名开始的位置，两个0xff标示和额外添加的注释部分的size  

zip文件格式是了解签名过程的基础，而这部分签名文件的过程和格式又是了解校验签名的基础，一环扣一环。这部分熟悉之后就可以顺利的来看verifyPackage是如何工作的了。  

#### 3.3 verifyPackage

```java
	/**  
     * Verify the cryptographic signature of a system update package  
     * before installing it.  Note that the package is also verified  
     * separately by the installer once the device is rebooted into  
     * the recovery system.  This function will return only if the  
     * package was successfully verified; otherwise it will throw an  
     * exception.  
     *  
     * Verification of a package can take significant time, so this  
     * function should not be called from a UI thread.  Interrupting  
     * the thread while this function is in progress will result in a  
     * SecurityException being thrown (and the thread's interrupt flag  
     * will be cleared).  
     *  
     * @param packageFile  the package to be verified  
     * @param listener     an object to receive periodic progress  
     * updates as verification proceeds.  May be null.  
     * @param deviceCertsZipFile  the zip file of certificates whose  
     * public keys we will accept.  Verification succeeds if the  
     * package is signed by the private key corresponding to any  
     * public key in this file.  May be null to use the system default  
     * file (currently "/system/etc/security/otacerts.zip").  
     *  
     * @throws IOException if there were any errors reading the  
     * package or certs files.  
     * @throws GeneralSecurityException if verification failed  
     */  
    public static void verifyPackage(File packageFile,
                                     ProgressListener listener,
                                     File deviceCertsZipFile)
        throws IOException, GeneralSecurityException {// 3.1.1 参数解释
        // 取到文件长度，为后面读取文件特定位置
        long fileLen = packageFile.length();
		// 只读打开升级包
        RandomAccessFile raf = new RandomAccessFile(packageFile, "r");
        try {
            int lastPercent = 0;// 当前进度
            long lastPublishTime = System.currentTimeMillis();
            if (listener != null) {
                listener.onProgress(lastPercent);
            }
            // 阶段1：校验文件本身格式是否正确，是否包含签名信息
			// 3.1.2 校验文件尾
            // 将文件尾6个字节读到footer中
            raf.seek(fileLen - 6);
            byte[] footer = new byte[6];
            raf.readFully(footer);

            if (footer[2] != (byte)0xff || footer[3] != (byte)0xff) {
                throw new SignatureException("no signature in file (no footer)");
            }

            // 计算整个注释的大小，footer[4]是低八位，footer[5]是高八位
            int commentSize = (footer[4] & 0xff) | ((footer[5] & 0xff) << 8);
            // 计算签名开始的位置，footer[0]是低八位，footer[1]是高八位
            int signatureStart = (footer[0] & 0xff) | ((footer[1] & 0xff) << 8);

            // eocd本来应该是22个byte，这里我的理解应该是为了读文件方便，直接是从eocd开始的位置读到了文件尾，反正eocd是在读出来的前22个byte
            byte[] eocd = new byte[commentSize + 22];
            // 定位到eocd开始位置
            raf.seek(fileLen - (commentSize + 22));
            raf.readFully(eocd);

            // Check that we have found the start of the
            // end-of-central-directory record.
            // 检查eocd的标示
            if (eocd[0] != (byte)0x50 || eocd[1] != (byte)0x4b ||
                eocd[2] != (byte)0x05 || eocd[3] != (byte)0x06) {
                // 没检查到eocd为什么要抛异常说没有签名？我们反着理解，如果正常签名了，这里应该是正确的eocd没错，那么如果签名异常或是没签，这种方式拿到的肯定也就不是eocd的标示了
                throw new SignatureException("no signature in file (bad footer)");
            }
			// 检查下是不是有不该出现的eocd标示
            for (int i = 4; i < eocd.length-3; ++i) {
                if (eocd[i  ] == (byte)0x50 && eocd[i+1] == (byte)0x4b &&
                    eocd[i+2] == (byte)0x05 && eocd[i+3] == (byte)0x06) {
                    throw new SignatureException("EOCD marker found after start of EOCD");
                }
            }
            
            // 步骤2：检验签名数据本身是否有效，是否包含证书信息，比较证书是否和可信证书相同

            // The following code is largely copied from
            // JarUtils.verifySignature().  We could just *call* that
            // method here if that function didn't read the entire
            // input (ie, the whole OTA package) into memory just to
            // compute its message digest.
            // 这部分校验签名的方法大多数是从JarUtils.verifySignature()，适用于校验升级包、apk等这种只需要校验摘要的部分，而不需要将整个包的数据加载到内存中。
            // 对这里的摘要不理解的，可以去参考我签名的系列三 签名里关于升级包签名系统的设计

            // 这里的eocd包含了eocd+comment(signed by Signapk)+signedData+6个字节
            BerInputStream bis = new BerInputStream(
                // commentSize+22就是整个eocd的size，减去signatureStart之后，就形成offset
                // 意思就是去读取签名数据部分
                new ByteArrayInputStream(eocd, commentSize+22-signatureStart, signatureStart));
            // 拿到签名后的数据块
            ContentInfo info = (ContentInfo)ContentInfo.ASN1.decode(bis);
            SignedData signedData = info.getSignedData();
            if (signedData == null) {
                throw new IOException("signedData is null");
            }
            // 从中拿到公钥
            Collection encCerts = signedData.getCertificates();
            if (encCerts.isEmpty()) {
                throw new IOException("encCerts is empty");
            }
            // Take the first certificate from the signature (packages
            // should contain only one).
            Iterator it = encCerts.iterator();
            X509Certificate cert = null;
            if (it.hasNext()) {
                // 公钥，x509.pem里的证书
                cert = new X509CertImpl((org.apache.harmony.security.x509.Certificate)it.next());
            } else {
                throw new SignatureException("signature contains no certificates");
            }
			// 从签名数据中拿到签名者信息
            List sigInfos = signedData.getSignerInfos();
            SignerInfo sigInfo;
            if (!sigInfos.isEmpty()) {
                sigInfo = (SignerInfo)sigInfos.get(0);
            } else {
                throw new IOException("no signer infos!");
            }

            // Check that the public key of the certificate contained
            // in the package equals one of our trusted public keys.
			// 从deviceCertsZipFile里拿到传进来的公钥信息（传进来的被认为是可信公钥，去跟zip包里读出来的做对比）
            // deviceCertsZipFile一般为null，我们就用设备下的/system/etc/security/otacerts.zip
            HashSet<Certificate> trusted = getTrustedCerts(
                deviceCertsZipFile == null ? DEFAULT_KEYSTORE : deviceCertsZipFile);

            PublicKey signatureKey = cert.getPublicKey();
            boolean verified = false;
            // 对比公钥是否相同
            for (Certificate c : trusted) {
                if (c.getPublicKey().equals(signatureKey)) {
                    verified = true;
                    break;
                }
            }
            if (!verified) {// 不相同直接就报错了
                throw new SignatureException("signature doesn't match any trusted key");
            }
            
            // 步骤3： 检查加密数据块里摘要信息跟实际文件里的是否一致

            // The signature cert matches a trusted key.  Now verify that
            // the digest in the cert matches the actual file data.

            // The verifier in recovery only handles SHA1withRSA and
            // SHA256withRSA signatures.  SignApk chooses which to use
            // based on the signature algorithm of the cert:
            //
            //    "SHA256withRSA" cert -> "SHA256withRSA" signature
            //    "SHA1withRSA" cert   -> "SHA1withRSA" signature
            //    "MD5withRSA" cert    -> "SHA1withRSA" signature (for backwards compatibility)
            //    any other cert       -> SignApk fails
            //
            // Here we ignore whatever the cert says, and instead use
            // whatever algorithm is used by the signature.

            String da = sigInfo.getDigestAlgorithm();// 获取摘要算法
            String dea = sigInfo.getDigestEncryptionAlgorithm();// 获取摘要加密算法
            String alg = null;
            if (da == null || dea == null) {
                // fall back to the cert algorithm if the sig one
                // doesn't look right.
                alg = cert.getSigAlgName();//用证书里的签名算法，这里是SHA1withRSA
            } else {
                alg = da + "with" + dea;
            }
            Signature sig = Signature.getInstance(alg);
            sig.initVerify(cert);

            // The signature covers all of the OTA package except the
            // archive comment and its 2-byte length.
            // 这一部分关系到签名算法等，我还不是太清楚，只能大概解释这里在什么，后续有缘再深入思考吧 
            // 前面的sig初始化了cert公钥信息
            long toRead = fileLen - commentSize - 2;
            long soFar = 0;
            raf.seek(0);
            byte[] buffer = new byte[4096];
            boolean interrupted = false;
            while (soFar < toRead) {
                interrupted = Thread.interrupted();
                if (interrupted) break;
                int size = buffer.length;
                if (soFar + size > toRead) {// 如果没有超过文件尾，就用4k的步长来读
                    size = (int)(toRead - soFar);
                }
                // 
                int read = raf.read(buffer, 0, size);
                sig.update(buffer, 0, read);// 不停的从文件中读出内容，然后更新待验证的sig
                soFar += read;

                if (listener != null) {
                    long now = System.currentTimeMillis();
                    int p = (int)(soFar * 100 / toRead);
                    if (p > lastPercent &&
                        now - lastPublishTime > PUBLISH_PROGRESS_INTERVAL_MS) {
                        lastPercent = p;
                        lastPublishTime = now;
                        listener.onProgress(lastPercent);
                    }
                }
            }
            if (listener != null) {
                listener.onProgress(100);
            }

            if (interrupted) {
                throw new SignatureException("verification was interrupted");
            }

            if (!sig.verify(sigInfo.getEncryptedDigest())) {// 读完之后和加密数据里的信息校验一把
                throw new SignatureException("signature digest verification failed");
            }
        } finally {
            raf.close();
        }
    }
```

##### 3.3.1 参数解释

传入三个参数：  

packageFile--安装包路径。在verifyPackage方法里没有判断该文件是否存在的代码，直接去读取该文件里的内容，有问题则跑IOException，我们在设计升级app的时候，可以把判断升级包文件是否存在加在校验之前，减少不必要的校验。  

ProgressListener--进度监听器，传入此参数可以通过listener.onProgress来通知调用者当前进度  

deviceCertsZipFile--签名证书压缩包路径，可以传入null，使用系统默认的证书/system/etc/security/otacerts.zip。升级包如果用的是此压缩包里任意公钥对应的私钥签名的，则校验都能通过。其实otacerts.zip里面就是releasekey.x509.pem。    

##### 3.3.2 校验文件格式

从代码能看出来读取了文件尾的6个字节，去校验第三第四个字节，是否是0xff，那么这里为什么这么做呢？认知看过签名的signWholeFile内容的就很清楚了，文件尾最末6个字节，byte[0]和byte[1]存储了签名开始的位置，中间两个0xff只是个标示，用来识别这6个特殊字节，byte[4]和byte[5]存储了整个注释部分的长度。  

##### 3.3.3 校验签名数据本身

检查证书信息是否存在，证书作者的信息是否存在，公钥信息是否存在  

##### 3.3.4 校验摘要信息是否和文件本身中的一致

通过公钥要初始化，然后读文件不停更新，构成一个待校验的签名数据，完成后和签名数据块的加密只要数据对比是否一致。  

#### 3.4 installPackage

升级包的校验成功之后，我们就可以调用installPackage来请求安装升级包，注意这里说的是**请求**安装升级包，而并不是真正的安装，因为安装过程是发生在重启之后进入到recovery下进行的。  

installPackage有两个重载方法，差别在于是否带```String msg```，此参数的含义最后在于传给PowerManagerService的reboot，作为重启原因，而如果msg为null的话，就默认重启原因为recovery。那我们就以不带msg参数的方法来做讲解吧。  

```java
	/**  
     * Reboots the device in order to install the given update  
     * package.  
     * Requires the {@link android.Manifest.permission#REBOOT} permission.  
     *  
     * @param context      the Context to use  
     * @param packageFile  the update package to install.  Must be on  
     * a partition mountable by recovery.  (The set of partitions  
     * known to recovery may vary from device to device.  Generally,  
     * /cache and /data are safe.)  
     *  
     * @throws IOException  if writing the recovery command file  
     * fails, or if the reboot itself fails.  
     */  
    public static void installPackage(Context context, File packageFile)
        throws IOException {
        String filename = packageFile.getCanonicalPath();//获取升级包的标准路径格式
        Log.w(TAG, "!!! REBOOTING TO INSTALL " + filename + " !!!");
        // 构造升级命令，后面要写入到cache/recovery/command里去的
        String arg = "--update_package=" + filename +
            "\n--locale=" + Locale.getDefault().toString();
        // 调用bootCommand来写命令arg
        bootCommand(context, arg);
    }
```

```java
	/**  
     * Reboot into the recovery system with the supplied argument.  
     * @param arg to pass to the recovery utility.  
     * @throws IOException if something goes wrong.  
     */  
    private static void bootCommand(Context context, String arg) throws IOException {
        // cache/recovery/
        RECOVERY_DIR.mkdirs();  // In case we need it
        // cache/recovery/command
        COMMAND_FILE.delete();  // In case it's not writable
        // cache/recovery/log
        LOG_FILE.delete();

        // 写命令到command中
        FileWriter command = new FileWriter(COMMAND_FILE);
        try {
            command.write(arg);
            command.write("\n");
        } finally {
            command.close();
        }

        // 用pm来重启
        // Having written the command file, go ahead and reboot
        PowerManager pm = (PowerManager) context.getSystemService(Context.POWER_SERVICE);
        pm.reboot("recovery");

        throw new IOException("Reboot failed (no permissions?)");
    }
```

请求升级的流程代码就比较简单了，到这里就把升级包请求命令和升级包的位置写入到/cache/recovery/command中，而后面要做的事就是开机时去读取command，来决定启动进入recovery，然后找到升级包安装，敬请关于后续。  