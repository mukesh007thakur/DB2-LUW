DEV
Inadwa10: Access Server 11.4.0.4.11072
Windows Server: Management Console 11.4.0.3.11037

PROD
Access Server 11.4.0.4.11072
Windows Server: Management Console 11.4.0.4.11072



==========

Generate a Private Key for all, self-sign or CA Sign or Public Authority.

$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -genkeypair -noprompt -alias self -keyalg EC -groupname secp256r1 -sigalg SHA256withECDSA -dname "CN=mukesh007thakur.com" -validity 365 -keystore privatekey.p12 -storepass password -storetype PKCS12 -ext BasicConstraints:critical=ca:true -ext KeyUsage:critical=keyCertSign,cRLSign


$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -list -keystore privatekey.p12 -storepass password
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 1 entry

self, Sep 25, 2026, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 11:06:3C:9B:E1:25:B4:58:91:74:8E:13:BD:5B:D8:2F:ED:ED:3E:3E:54:2B:64:A0:15:53:17:BB:73:9D:17:B5

-----

Export the Certificate from Private Key:

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -exportcert -noprompt -rfc -alias self -file pk1.crt -keystore privatekey.p12 -storepass password -storetype PKCS12

$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -exportcert -noprompt -rfc -alias self -file pk1.crt -keystore privatekey.p12 -storepass password -storetype PKCS12
Certificate stored in file <pk1.crt>
$ ls -ltr
total 24
-rw-rw-r-- 1 cdcadm cdcadm    0 Oct 17  2024 secrets.lock
-rw-rw-r-- 1 cdcadm cdcadm   32 Oct 17  2024 secrets.b64
-rw-rw-r-- 1 cdcadm cdcadm   45 Oct 17  2024 encryptionprofile.properties
-rw-rw-r-- 1 cdcadm cdcadm 2333 Oct 17  2024 bak.secrets
-rw-rw-r-- 1 cdcadm cdcadm 3093 Sep 14 14:28 secrets.p12
-rw-r--r-- 1 cdcadm cdcadm 1171 Sep 25 06:35 privatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  615 Sep 25 06:39 pk1.crt

-----

Create a CSR request:

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -certreq -noprompt -alias self -sigalg SHA256withECDSA -file pk1.csr -keystore privatekey.p12 -dname "CN=mukesh007thakur.com" -storepass password -storetype PKCS12 -ext KeyUsage:critical=digitalSignature -ext ExtendedKeyUsage:critical=serverAuth,clientAuth



$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -certreq -noprompt -alias self -sigalg SHA256withECDSA -file pk1.csr -keystore privatekey.p12 -dname "CN=mukesh007thakur.com" -storepass password -storetype PKCS12 -ext KeyUsage:critical=digitalSignature -ext ExtendedKeyUsage:critical=serverAuth,clientAuth

$ ls -ltr
total 28
-rw-rw-r-- 1 cdcadm cdcadm    0 Oct 17  2024 secrets.lock
-rw-rw-r-- 1 cdcadm cdcadm   32 Oct 17  2024 secrets.b64
-rw-rw-r-- 1 cdcadm cdcadm   45 Oct 17  2024 encryptionprofile.properties
-rw-rw-r-- 1 cdcadm cdcadm 2333 Oct 17  2024 bak.secrets
-rw-rw-r-- 1 cdcadm cdcadm 3093 Sep 14 14:28 secrets.p12
-rw-r--r-- 1 cdcadm cdcadm 1171 Sep 25 06:35 privatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  615 Sep 25 06:39 pk1.crt
-rw-r--r-- 1 cdcadm cdcadm  535 Sep 25 06:41 pk1.csr

-----

Import the certificates:

In Private Key Store:

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -importcert -noprompt -alias self -file pk1.pem -keystore privatekey.p12 -storepass password -storetype PKCS12

In Trust Store:

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -importcert -noprompt -alias pk1 -file pk1.crt -keystore trust.p12 -storepass password -storetype PKCS12

-----

Import Keystore:
/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -importkeystore -noprompt -srckeystore /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/lib/security/cacerts -destkeystore trust.p12 -deststoretype PKCS12 -srcstorepass changeit -deststorepass password


==============

Generate a Private Key for Cert Authority when i am the authority:

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -genkeypair -noprompt -alias self -keyalg EC -groupname secp256r1 -sigalg SHA256withECDSA -dname "O=mkt.com"           -validity 365 -keystore rootcaprivatekey.p12 -storepass password -storetype PKCS12 -ext BasicConstraints:critical=ca:true -ext KeyUsage:critical=keyCertSign,cRLSign


$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -genkeypair -noprompt -alias self -keyalg EC -groupname secp256r1 -sigalg SHA256withECDSA -dname "O=mkt.com"           -validity 365 -keystore rootcaprivatekey.p12 -storepass password -storetype PKCS12 -ext BasicConstraints:critical=ca:true -ext KeyUsage:critical=keyCertSign,cRLSign
Generating 256 bit EC (secp256r1) key pair and self-signed certificate (SHA256withECDSA) with a validity of 365 days
	for: O=ibm.com


$ ls -ltr
total 32
-rw-rw-r-- 1 cdcadm cdcadm    0 Oct 17  2024 secrets.lock
-rw-rw-r-- 1 cdcadm cdcadm   32 Oct 17  2024 secrets.b64
-rw-rw-r-- 1 cdcadm cdcadm   45 Oct 17  2024 encryptionprofile.properties
-rw-rw-r-- 1 cdcadm cdcadm 2333 Oct 17  2024 bak.secrets
-rw-rw-r-- 1 cdcadm cdcadm 3093 Sep 14 14:28 secrets.p12
-rw-r--r-- 1 cdcadm cdcadm 1171 Sep 25 06:35 privatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  615 Sep 25 06:39 pk1.crt
-rw-r--r-- 1 cdcadm cdcadm  535 Sep 25 06:41 pk1.csr
-rw-r--r-- 1 cdcadm cdcadm 1123 Sep 25 06:43 rootcaprivatekey.p12

$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -list -keystore rootcaprivatekey.p12 -storepass password
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 1 entry

self, Sep 25, 2026, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 38:F7:FC:70:23:A5:0D:AF:E3:67:87:3A:0F:42:4B:86:32:9B:F5:29:F2:42:60:70:EA:59:EB:99:5F:07:AF:3C


$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -list -keystore privatekey.p12 -storepass password
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 1 entry

self, Sep 25, 2026, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 11:06:3C:9B:E1:25:B4:58:91:74:8E:13:BD:5B:D8:2F:ED:ED:3E:3E:54:2B:64:A0:15:53:17:BB:73:9D:17:B5

-----

Sign a Certificate 

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -gencert -noprompt -infile pk1.csr -outfile pk1signedbyrootca.crt -alias self -sigalg SHA256withECDSA -validity 365 -keystore rootcaprivatekey.p12 -storepass password -storetype PKCS12 -rfc -ext KeyUsage:critical=digitalSignature -ext ExtendedKeyUsage:critical=serverAuth,clientAuth

$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -gencert -noprompt -infile pk1.csr -outfile pk1signedbyrootca.crt -alias self -sigalg SHA256withECDSA -validity 365 -keystore rootcaprivatekey.p12 -storepass password -storetype PKCS12 -rfc -ext KeyUsage:critical=digitalSignature -ext ExtendedKeyUsage:critical=serverAuth,clientAuth

$ ls -ltr
total 36
-rw-rw-r-- 1 cdcadm cdcadm    0 Oct 17  2024 secrets.lock
-rw-rw-r-- 1 cdcadm cdcadm   32 Oct 17  2024 secrets.b64
-rw-rw-r-- 1 cdcadm cdcadm   45 Oct 17  2024 encryptionprofile.properties
-rw-rw-r-- 1 cdcadm cdcadm 2333 Oct 17  2024 bak.secrets
-rw-rw-r-- 1 cdcadm cdcadm 3093 Sep 14 14:28 secrets.p12
-rw-r--r-- 1 cdcadm cdcadm 1171 Sep 25 06:35 privatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  615 Sep 25 06:39 pk1.crt
-rw-r--r-- 1 cdcadm cdcadm  535 Sep 25 06:41 pk1.csr
-rw-r--r-- 1 cdcadm cdcadm 1123 Sep 25 06:43 rootcaprivatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  647 Sep 25 06:45 pk1signedbyrootca.crt

$ cat pk1signedbyrootca.crt
-----BEGIN CERTIFICATE-----
MIIBqjCCAVCgAwIBAgIJAJHyszMkf7vdMAoGCCqGSM49BAMCMBIxEDAOBgNVBAoT
B2libS5jb20wHhcNMjYwOTI1MTM0NTM2WhcNMjcwOTI1MTM0NTM2WjAtMSswKQYD
VQQDEyJ0ZWwtZ2RjLWNkYy1wcmFrZWRpYTEuZnlyZS5pYm0uY29tMFkwEwYHKoZI
zj0CAQYIKoZIzj0DAQcDQgAE8BcPNtGroekqqp8eGNHw+woenitjtuBOQhCcNbsz
mP2csgtnuNpt0qtLL3Sy3ePlK9VUD9On+3mGjqXY6hQezKN0MHIwHQYDVR0OBBYE
FF2V7pm0qsi1ze3brsurKYDZLBxfMA4GA1UdDwEB/wQEAwIHgDAfBgNVHSMEGDAW
gBQM3xm54KNpTQ5xp5DbA+JUnk0DXzAgBgNVHSUBAf8EFjAUBggrBgEFBQcDAQYI
KwYBBQUHAwIwCgYIKoZIzj0EAwIDSAAwRQIgLxsVf+yzIq5Kr0RZMO1lfs+KMFsx
DEL6zrp6z576WT4CIQC3Yu7RGhjGT2awoAUY0dL4SbMSxoxyUQSZRhhLpybxdQ==
-----END CERTIFICATE-----

$ cat pk1.crt
-----BEGIN CERTIFICATE-----
MIIBkzCCATmgAwIBAgIJAIW3jwHVntxbMAoGCCqGSM49BAMCMC0xKzApBgNVBAMT
InRlbC1nZGMtY2RjLXByYWtlZGlhMS5meXJlLmlibS5jb20wHhcNMjYwOTI1MTMz
NTQ2WhcNMjcwOTI1MTMzNTQ2WjAtMSswKQYDVQQDEyJ0ZWwtZ2RjLWNkYy1wcmFr
ZWRpYTEuZnlyZS5pYm0uY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE8BcP
NtGroekqqp8eGNHw+woenitjtuBOQhCcNbszmP2csgtnuNpt0qtLL3Sy3ePlK9VU
D9On+3mGjqXY6hQezKNCMEAwHQYDVR0OBBYEFF2V7pm0qsi1ze3brsurKYDZLBxf
MA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MAoGCCqGSM49BAMCA0gA
MEUCIDbOr/W+RrZDMZYrmceLJQwD8SwYKIaoUJsC5OuH0uKNAiEAzLQrO+zMc+jw
QHCNLpD+Qvay66wJNu//TmB6jNsg1nE=
-----END CERTIFICATE-----



-----

Export the CA Certificate when i am the authority.

/home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -exportcert -noprompt -rfc -alias self -file pk1ca.crt -keystore rootcaprivatekey.p12 -storepass password -storetype PKCS12

$ /home/cdcadm/InfoSphereDataReplication/ReplicationEngineforFlexRep/jre64/jre/bin/keytool -exportcert -noprompt -rfc -alias self -file pk1ca.crt -keystore rootcaprivatekey.p12 -storepass password -storetype PKCS12
Certificate stored in file <pk1ca.crt>

$ ls -ltr
total 40
-rw-rw-r-- 1 cdcadm cdcadm    0 Oct 17  2024 secrets.lock
-rw-rw-r-- 1 cdcadm cdcadm   32 Oct 17  2024 secrets.b64
-rw-rw-r-- 1 cdcadm cdcadm   45 Oct 17  2024 encryptionprofile.properties
-rw-rw-r-- 1 cdcadm cdcadm 2333 Oct 17  2024 bak.secrets
-rw-rw-r-- 1 cdcadm cdcadm 3093 Sep 14 14:28 secrets.p12
-rw-r--r-- 1 cdcadm cdcadm 1171 Sep 25 06:35 privatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  615 Sep 25 06:39 pk1.crt
-rw-r--r-- 1 cdcadm cdcadm  535 Sep 25 06:41 pk1.csr
-rw-r--r-- 1 cdcadm cdcadm 1123 Sep 25 06:43 rootcaprivatekey.p12
-rw-r--r-- 1 cdcadm cdcadm  647 Sep 25 06:45 pk1signedbyrootca.crt
-rw-r--r-- 1 cdcadm cdcadm  541 Sep 25 06:49 pk1ca.crt


===============
