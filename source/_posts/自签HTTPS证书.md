---
title: 自签HTTPS证书
date: 2022-06-16 10:36:00
tags:
  - linux
search: true
---

## 创建CA根证书

第一步是创建一个秘钥，这个便是CA证书的根本，之后所有的东西都来自这个秘钥：

```sh
# 通过rsa算法生成2048位长度的秘钥
openssl genrsa -out rootCA.key 2048
```

第二步是通过秘钥加密机构信息形成公钥：

```sh
# 公钥包含了机构信息，在输入下面的指令之后会有一系列的信息输入，这些信息便是机构信息，公司名称地址什么的
# 这里还有一个过期信息，CA证书也会过期，openssl默认是一个月，我们直接搞到100年
openssl req -new -x509 -key rootCA.key -out rootCA.cer -days 36500
```

## 颁发服务器证书

在得到CA证书之后，需要通过`openssl`工具对证书进行转换得到公钥（`.crt文件`）和密钥（`.key文件`）。

1. 第一步通过`openssl`工具创建服务器证书的秘钥

   ```sh
   # 通过RSA算法生成长度2048位的秘钥
   openssl genrsa -out server.key 2048
   
   ```

2. 第二步这里是创建一个签名请求

   将下面的配置内容保存为`openssl.cnf`放到生成的服务器证书文件的目录下（**注意**：修改alt_names里面的域名或者IP为最终部署需要的地址，支持通配符），然后执行创建签名申请文件即可

   ```ini
   tsa_policy2 = 1.2.3.4.5.6
   tsa_policy3 = 1.2.3.4.5.7
   [ ca ]
   default_ca = CA_default  # The default ca section
   [ CA_default ]
   dir  = ./demoCA  # Where everything is kept
   certs  = $dir/certs  # Where the issued certs are kept
   crl_dir  = $dir/crl  # Where the issued crl are kept
   database = $dir/index.txt # database index file.
   new_certs_dir = $dir/newcerts  # default place for new certs.
   certificate = $dir/cacert.pem  # The CA certificate
   serial  = $dir/serial   # The current serial number
   crlnumber = $dir/crlnumber # the current crl number
   crl  = $dir/crl.pem   # The current CRL
   private_key = $dir/private/cakey.pem# The private key
   RANDFILE = $dir/private/.rand # private random number file
   x509_extensions = usr_cert  # The extentions to add to the cert
   name_opt  = ca_default  # Subject Name options
   cert_opt  = ca_default  # Certificate field options
   default_days = 365   # how long to certify for
   default_crl_days= 30   # how long before next CRL
   default_md = default  # use public key default MD
   preserve = no   # keep passed DN ordering
   policy  = policy_match
   [ policy_match ]
   countryName  = match
   stateOrProvinceName = match
   organizationName = match
   organizationalUnitName = optional
   commonName  = supplied
   emailAddress  = optional
   [ policy_anything ]
   countryName  = optional
   stateOrProvinceName = optional
   localityName  = optional
   organizationName = optional
   organizationalUnitName = optional
   commonName  = supplied
   emailAddress  = optional
   [ req ]
   default_bits  = 1024
   default_keyfile  = privkey.pem
   distinguished_name = req_distinguished_name
   attributes  = req_attributes
   x509_extensions = v3_ca # The extentions to add to the self signed cert
   string_mask = utf8only
   req_extensions = v3_req # The extensions to add to a certificate request
   [ req_distinguished_name ]
   countryName   = Country Name (2 letter code)
   countryName_default  = CN
   countryName_min   = 2
   countryName_max   = 2
   stateOrProvinceName  = State or Province Name (full name)
   stateOrProvinceName_default = ZheJiang
   localityName   = Locality Name (eg, city)
   0.organizationName  = Organization Name (eg, company)
   0.organizationName_default = HZSYRIS
   organizationalUnitName  = Organizational Unit Name (eg, section)
   commonName   = Common Name (e.g. server FQDN or YOUR name)
   commonName_max   = 64
   emailAddress   = Email Address
   emailAddress_max  = 64
   [ req_attributes ]
   challengePassword  = A challenge password
   challengePassword_min  = 4
   challengePassword_max  = 20
   unstructuredName  = An optional company name
   [ usr_cert ]
   basicConstraints=CA:FALSE
   nsCertType = client, email, objsign
   keyUsage = nonRepudiation, digitalSignature, keyEncipherment
   nsComment   = "OpenSSL Generated Certificate"
   subjectKeyIdentifier=hash
   authorityKeyIdentifier=keyid,issuer
   [ svr_cert ]
   basicConstraints=CA:FALSE
   nsCertType   = server
   keyUsage = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment, keyAgreement
   subjectKeyIdentifier=hash
   authorityKeyIdentifier=keyid,issuer
   extendedKeyUsage = serverAuth,clientAuth
   [ v3_req ]
   subjectAltName = @alt_names
   # 这里是重点，需要将里面配置为最终服务端需要的域名或者IP
   # 这里可以写多个，能够自行添加DNS.X = XXXXXX
   [ alt_names ]
   DNS.1 = syris.devp
   DNS.2 = *.syris.devp
   [ v3_ca ]
   subjectKeyIdentifier=hash
   authorityKeyIdentifier=keyid:always,issuer
   basicConstraints = CA:true
   [ crl_ext ]
   authorityKeyIdentifier=keyid:always
   [ proxy_cert_ext ]
   basicConstraints=CA:FALSE
   nsComment   = "OpenSSL Generated Certificate"
   subjectKeyIdentifier=hash
   authorityKeyIdentifier=keyid,issuer
   proxyCertInfo=critical,language:id-ppl-anyLanguage,pathlen:3,policy:foo
   [ tsa ]
   default_tsa = tsa_config1 # the default TSA section
   [ tsa_config1 ]
   dir  = ./demoCA  # TSA root directory
   serial  = $dir/tsaserial # The current serial number (mandatory)
   crypto_device = builtin  # OpenSSL engine to use for signing
   signer_cert = $dir/tsacert.pem  # The TSA signing certificate
        # (optional)
   certs  = $dir/cacert.pem # Certificate chain to include in reply
        # (optional)
   signer_key = $dir/private/tsakey.pem # The TSA private key (optional)
   default_policy = tsa_policy1  # Policy if request did not specify it
        # (optional)
   other_policies = tsa_policy2, tsa_policy3 # acceptable policies (optional)
   digests  = md5, sha1  # Acceptable message digests (mandatory)
   accuracy = secs:1, millisecs:500, microsecs:100 # (optional)
   clock_precision_digits  = 0 # number of digits after dot. (optional)
   ordering  = yes # Is ordering defined for timestamps?
       # (optional, default: no)
   tsa_name  = yes # Must the TSA name be included in the reply?
       # (optional, default: no)
   ess_cert_id_chain = no # Must the ESS cert id chain be included?
       # (optional, default: no)
   ```

   执行开始创建请求

   ```sh
   # 和创建CA时一样这里需要输入一堆服务器信息，输入项也是相同的。
   # 不过在输入Common Name（CN）最好直接输入服务器的IP地址或者域名。
   openssl req -config openssl.cnf -new -out server.req -key server.key 
   ```

3. 第三步通过CA机构证书对服务器证书进行签名认证

   ```sh
   # 这里没有什么需要说的，本质上就是将签名请求文件进行签名最终得到服务器的公钥
   openssl x509 -req  -extfile openssl.cnf -extensions v3_req -in server.req -out server.cer -CAkey rootCA.key -CA rootCA.cer -days 3650 -CAcreateserial -CAserial serial
   ```

## 信任CA机构证书

如果通过Windows域控创建的CA证书，其证书本身通过组策略便可以给每一个域下计算机添加机构信任。如果你没有域控只是通过`openssl`创建的CA证书也没有关系，只需要将CA证书的公钥（`rootCA.cer文件`）导入到系统信任的根证书颁发机构里面就行了：

