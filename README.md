# SSL Generation and Conversion processes

SSL is the backbone of modern security. This is a list of various processes collected over the years in one handy cheat sheet.

## Types of certificates and their uses
- **PEM** *(.key/.pem)*: PEM formatted keys and certs are flat-text files and are the most common certificates for Linux with Apache and NGINX
- **DER** *(.key/.der)*: Same private key format as PEM, with a binary-encoded certificate as opposed to plaintext certificate.
- **PKCS12** *(.p12)*: Microsoft's SSL import/export implementation (standards-based), these are a single password-protected bundle which contains the private key, certificate and possibly intermediaries and root CA certificates.
- **Keystore** *(.jks)*: Java's implementation of the PKCS12-style password-protected certificate bundle. Most commonly this is used for Apache Tomcat-based web applications.

##Generating certificates
By default, openssl does not create very secure certificate requests and keys. To try and maintain SSL security in the modern era, older sub-**2048bit** keys and **SHA-1** hashing algorithms certificates are no longer supported or issued (See the [CA/Browser Forum Baseline Requirements docs](https://cabforum.org/baseline-requirements-documents/) for further detail. Depending on expected load, going with **4096-bit** keys and **SHA256** hashed certificates are preferred now. For security, without an HSM it's recommended to generate the keys on the server to prevent key loss. The security of the key is of utmost importance because it unlocks the signed certificate and can be used for man-in-the-middle attacks.

## Linux
1. Generate a new key and CSR on the Linux system
    - `openssl req -new -newkey rsa:4096 -nodes -sha256 -keyout mykey.key -out mycsr.csr`
         >This creates a 4096-bit key and SHA256 PEM-encoded CSR
2. Follow the prompts, paying paticular attention to **Common Name**, which should be the FQDN of the server to be signed.
    >Note: IP address, non-public domains like .local or .internal, or domain names not purchased by you will not be signed by public CAs, drastically reducing certificate usefulness.
3. Take the `.csr` file and send it to your preferred CA, following their instructions to purchase and acquire a signed PEM certificate (most likely it'll be be a .pem certificate). Most should have the option to provide your own CSR.
4. Take the signed certificate and store it with the key generated earlier.

## Windows without IIS
Windows certificates can be used for many different functions proving identity, This focuses on generating SSL certificate for applications on Windows to use to secure web applications.

1. Create a `.inf` file on the Windows system with the following content
    - *Subject*: Specifies all the details prompted for on the Linux side, in the following format: CN=servername.example.com, OU=Department, O=Company, L=City, S=State, C=Country
        >State may be dropped if not applicable for the location
    - *KeySpec*: Set to "1" to allow the certificate to be used for encryption.
    - *KeyLength*: Set the length of the key here. Must be at least 2048, 4096 is recommended.
    - *HashAlgorithm*: Sets the signing hash algorithm. SHA-2 is required for public CAs.
    - *Exportable*: Sets whether or not the key can be exported. It's more secure to set to `FALSE`, however if the OS fails to boot or hardware fails the certifiate will need to be re-issued. Setting to `TRUE` allows for the certificate to be exported in a password-protected **PKCS12** file.
    - *MachineKeySet*: Windows PCs have multiple certificate stores, for each user of the system and the central computer store. Setting this to `TRUE` will install the key in the computer certificate store, where applications and services can find it.
    - *SMIME*: Sets whether the certificate will be used to sign email. As this will be used for web applications, enter `FALSE".
    - *PrivateKeyArchive*: This setting is for Active Directory-bound systems with an AD-managed internal CA. It allows the CA to backup the private key. For most purposes on standalone servers, set this to `FALSE`.
    - *UserProtected*: Marks the certificate to require approval every time the private key is used. Most cases will set this to `False`
    - *UseExistingKeySet*: This sets whether to re-use an existing key for a certificate renewal. It's always recommended to generate new keys with every certificate request for security so this should be set to `FALSE`
    - *RequestType*: Sets the type of request, whether it will make a standard request for a signed `PKCS10` bundle, self-signed certificate with `Cert` or use AD-bound CA with `CMC` (and the associated *PrivateKeyArchive* above )
    - *KeyUsage*: Sets the uses for the the key, For most typical SSL web application usage it should be set to `0xa0`.
    - *ProviderName*: Because certificates in Windows take many forms, there are many providers which to request certificates from. For standard web uses, we'll want to use `Microsoft RSA SChannel Cryptographic Provider`.
    - *FriendlyName*: Gives the certificate a user-friendly name, which can help delineate similar certificates from each other (i.e. certificate issue year for renewals))
    -There are also Version and EnhancedKeyUsageExtension pieces Microsoft recommends using for SSL certificates.
    
The request should look something like this:
```
   [Version]
   Signature="$Windows NT$"
    
   [NewRequest]
   Subject = "CN=servername.example.com, OU=Department, O=Company, L=City, S=State, C=Country"
   KeySpec = 1
   KeyLength = 4096
   HashAlgorithm = SHA256
   Exportable = TRUE
   MachineKeySet = TRUE
   SMIME = FALSE
   PrivateKeyArchive = FALSE
   UserProtected = FALSE
   UseExistingKeySet = FALSE
   RequestType = PKCS10
   KeyUsage = 0xa0
   ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
   FriendlyName = ""
    
   [EnhancedKeyUsageExtension]
   OID=1.3.6.1.5.5.7.3.1 ; Server Authentication
   ```
2. Use the `certreq` utility to request a certificate, and export a standard PEM CSR
    - `certreq -new mycert.inf -out myreq.csr`
3. Take the `.csr` file and send it to your preferred CA, following their instructions to purchase and acquire a signed binary DER certificate (most likely it'll be a .cer certificate). Most should have the option to provide your own CSR. Request a binary DER certificate to proceed.
4. Once you have the signed certificate, import it back into the computer certificate store.
    - `certreq -accept signedcert.cer`
>Microsoft's Certificate store is powerful and has many uses. Read more about these settings and others here: https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/certreq_1