# PKCS11 and HFL LAB

## PKCS11 and SOFTHSM2

SoftHSM is basically an implementation of a cryptographic store accessible through a PKCS #11 interface. The PKCS#11 interface is used to communicate or access the cryptographic devices such as HSM (Hardware Security Modules) and smart cards. The primary purpose of HSM devices is to generate cryptographic keys and sign/encrypt information without revealing the private key to the others.

To make it more easy to understand, it was not possible for OpenDNSSEC users to buy new hardware token for the storage of cryptographic keys. So, to counter this issue, OpenDNSSEC started providing "SoftHSM", a software implementation of a generic cryptographic device with a PKCS#11 interface. SoftHSM is designed to meet the requirements of OpenDNSSEC and also work with other cryptographic products. 

### Installing Dependencies
OpenSSL cryptographic libraries can be used with the SoftHSM project. I recommend to configured your LinuxONE or IBM Z Linux host with the following guidance [here](https://github.com/guikarai/PE-LinuxONE/blob/master/index.md) in order to be sure that your OpenSSL is backed up with ibmca engine in order to get access to hardware crypto acceleration. Limit yourself to the first chapter about OpenSSL, dm-crypt is not required for the following.

### Installing SoftHSM
SoftHSM is available from the OpenDNSSEC website, or can be installed thans to the following command:
```
blockchain@blkchn30:~$ sudo apt-get install softhsm2-util softhsm2-common
```

### Configure SoftHSM
Let’s identity together where is the the default location of the config file softhsm2.conf. Please issue the following command:
```
blockchain@blkchn30:~$ sudo find / -name softhsm2.conf
/etc/softhsm/softhsm2.conf
/usr/share/softhsm/softhsm2.conf
```

Let’s now set the SOFTHSM2_CONF environment variable, reusing output of the preceding command. Please issue the following command:
```
blockchain@blkchn30:~$ export SOFTHSM2_CONF=/etc/softhsm/softhsm2.conf
```

Now, let’s create the token repository folder, please issue the following command:
```
blockchain@blkchn30:~$ mkdir /var/lib/softhsm/tokens/
```

Let’s locate the softhsm pkcs11 library. This will be required for later. Please issue the following command:
```
blockchain@blkchn30:~$ sudo find / -name libsofthsm2.so
/usr/lib/softhsm/libsofthsm2.so
```

### Initialize Soft Token
The very first step to use SoftHSM is to use initialize it. We can use the "softhsm2-util" or the "PKCS#11" interface to initialize the device. The following snapshot shows the initialization of the SoftHSM device.

```
blockchain@blkchn30:~$ sudo softhsm2-util --init-token --slot 0 --label « ForFabric »
=== SO PIN (4-255 characters) ===
Please enter SO PIN: ********				#12345678
Please reenter SO PIN: ********			#12345678
=== User PIN (4-255 characters) ===
Please enter user PIN: ********			#87654321
Please reenter user PIN: ********			#87654321
The token has been initialized.
```

The Security Officer (SO) PIN is used to re-initialize the token and the user PIN is handed out to the application so it can interact with the token (like usage with Mozilla Firefox). That's why, set both SO and user PIN. Once a token has been initialized, more slots will be added automatically to a new uninitialized token. Initialized tokens will be reassigned to another slot based on the token serial number. It is recommended to find and interact with the token by searching for the token label or serial number in the slot list/token info.

Let’s check what the freshly created token looks like:
```
blockchain@blkchn30:~$ sudo softhsm2-util --show-slots
Available slots:
Slot 253912107
    Slot info:
        Description:      SoftHSM slot ID 0xf22642b                                       
        Manufacturer ID:  SoftHSM project                 
        Hardware version: 2.2
        Firmware version: 2.2
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project                 
        Model:            SoftHSM v2      
        Hardware version: 2.2
        Firmware version: 2.2
        Serial number:    bed6f60a8f22642b
        Initialized:      yes
        User PIN init.:   yes
        Label:            ForFabric
```

### Hyperledger Fabric, FABRIC CA, HSM and PKCS11

Hardware Security Module (HSM)
By default, the Fabric CA server and client store private keys in a PEM-encoded file, but they can also be configured to store private keys in an HSM (Hardware Security Module) via PKCS11 APIs. This behavior is configured in the BCCSP (BlockChain Crypto Service Provider) section of the server’s or client’s configuration file.

Configuring Fabric CA server to use softhsm2
This section shows how to configure the Fabric CA server or client to use a softhsm previously installed and configured.
We just installed it, and create a token, label it “ForFabric”, set the pin to ‘98765432’ (refer to softhsm documentation).
You can use both the config file and environment variables to configure BCCSP For example, set the bccsp section of Fabric CA server configuration file as follows.
Please issue the following command:

vi 
```
#############################################################################
# BCCSP (BlockChain Crypto Service Provider) section is used to select which
# crypto library implementation to use
#############################################################################
bccsp:
  default: PKCS11
  pkcs11:
    Library: /usr/lib/softhsm/libsofthsm2.so
    Pin: 87654321
    Label: ForFabric
    hash: SHA2
    security: 256
    filekeystore:
      # The directory used for the software file-based keystore
      keystore: msp/keystore
```

And you can override relevant fields via environment variables as follows:
    
FABRIC_CA_SERVER_BCCSP_DEFAULT=PKCS11
FABRIC_CA_SERVER_BCCSP_PKCS11_LIBRARY=/usr/lib/softhsm/libsofthsm2.so   #It refer instruction from above
FABRIC_CA_SERVER_BCCSP_PKCS11_PIN=87654321                              #It refer instruction from above
FABRIC_CA_SERVER_BCCSP_PKCS11_LABEL=ForFabric                           #It refer instruction from above


You are done.
