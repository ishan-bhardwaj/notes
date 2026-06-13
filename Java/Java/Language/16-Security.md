# Security

## Class Loaders

### The Class-Loading Process

- Three built-in class loaders:
    - __Bootstrap class loader__ — loads core platform modules (`java.base`, `java.logging`, etc.); no `ClassLoader` object — `String.class.getClassLoader()` returns `null`
    - __Platform class loader__ — loads remaining Java platform classes not in the bootstrap set
    - __System class loader__ — loads application classes from the module path and class path
- On demand only — classes are loaded when first needed
- __Resolving__ — loading all classes that a given class depends on (fields, superclasses, method return types, etc.)

### Class Loader Hierarchy

- Every class loader except bootstrap has a parent
- Delegation: parent is always asked first; child only loads if parent fails
- `URLClassLoader` — loads classes from JAR files or directories by URL:

```java
var url = new URL("file:///path/to/plugin.jar");
var pluginLoader = new URLClassLoader(new URL[] { url });
Class<?> cl = pluginLoader.loadClass("mypackage.MyClass");
```

- Same class name + different class loader = distinct class in the JVM — used by application servers to isolate apps from each other

### Context Class Loader

- __Classloader inversion__ — helper method loaded by system class loader trying to load a class that exists only in a plugin's class loader
- Each thread has a context class loader (default: system class loader):

```java
Thread.currentThread().setContextClassLoader(loader);
ClassLoader loader = Thread.currentThread().getContextClassLoader();
Class<?> cl = loader.loadClass(className);
```

- Used by JAXP, JNDI, and other frameworks to find the right loader

### Writing a Custom Class Loader

```java
class CryptoClassLoader extends ClassLoader {
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] classBytes = loadAndDecryptBytes(name);
            return defineClass(name, classBytes, 0, classBytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name);
        }
    }
}
```

- Override `findClass` — `loadClass` in the superclass handles delegation and caching
- Call `defineClass` to hand bytecodes to the JVM
- Use `.` as package separator in the class name; no `.class` suffix

### Bytecode Verification

- All non-platform classes are verified before execution
- Checks performed by the verifier:
    - Variables initialised before use
    - Method calls match object reference types
    - Private data access rules not violated
    - Local variable accesses stay within the runtime stack
    - Runtime stack does not overflow
- `VerifyError` thrown if checks fail
- `-noverify` / `-Xverify:none` options disable verification (deprecated; may be removed)

> [!NOTE]
> The verifier is needed because class files can be hand-crafted by someone with a hex editor, bypassing the compiler's own checks.

---

## User Authentication (JAAS)

### JAAS Framework

```java
var context = new LoginContext("Login1");  // name matches entry in JAAS config file
context.login();                           // throws LoginException if failed
Subject subject = context.getSubject();
for (Principal p : subject.getPrincipals())
    System.out.println(p.getClass().getName() + ": " + p.getName());
context.logout();
```

Launch with config file: `java -Djava.security.auth.login.config=jaas.config MyApp`

JAAS config format:

```
Login1 {
    com.sun.security.auth.module.UnixLoginModule required;
    com.whizzbang.auth.module.RetinaScanModule sufficient debug="true";
};
```

Module flags:
- `required` — must succeed; failure continues to remaining modules before rejecting
- `sufficient` — success short-circuits remaining modules; failure continues
- `requisite` — failure immediately rejects; success continues
- `optional` — result does not affect overall outcome

Authentication succeeds if all `required` and `requisite` modules pass, or (if none executed) at least one `sufficient` or `optional` module passes.

Built-in modules (`com.sun.security.auth.module`): `UnixLoginModule`, `NTLoginModule`, `Krb5LoginModule`, `JndiLoginModule`, `KeyStoreLoginModule`

### Custom Login Modules

Implement `LoginModule` — four methods:

```java
void initialize(Subject subject, CallbackHandler handler,
    Map<String,?> sharedState, Map<String,?> options)

boolean login()    // authenticate; populate subject's principals
boolean commit()   // called after all modules succeed (two-phase commit)
boolean abort()    // called on overall failure
boolean logout()   // clean up
```

Add principals in `login()`:

```java
subject.getPrincipals().add(new UserPrincipal(username));
subject.getPrincipals().add(new RolePrincipal(role));
```

### Callback Handlers

- Login modules request credentials via `Callback` objects — decoupled from UI
- Handler receives `Callback[]` and populates them:

```java
public void handle(Callback[] callbacks) {
    for (Callback callback : callbacks) {
        switch (callback) {
            case NameCallback c -> c.setName(username);
            case PasswordCallback c -> c.setPassword(password);
            default -> {}
        }
    }
}
```

- `TextCallbackHandler` — prompts from the console
- Custom handler — supply username/password from your own UI

---

## Digital Signatures

### Message Digests

```java
MessageDigest alg = MessageDigest.getInstance("SHA-256");
alg.update(bytes);                        // can call repeatedly
byte[] hash = alg.digest();               // finalise and return digest; resets object
byte[] hash = alg.digest(inputBytes);     // update + digest in one call
```

Supported algorithms: `SHA-1`, `SHA-256`, `SHA-512`, `SHA3-256`, `MD5` (MD5 deprecated for security)

List available algorithms:

```java
for (Provider p : Security.getProviders())
    for (Provider.Service s : p.getServices())
        if (s.getType().equals("MessageDigest"))
            System.out.println(s.getAlgorithm());
```

Properties of a secure digest:
- Changing any bit in the input changes the digest
- Computationally infeasible to find two inputs with the same digest
- Computationally infeasible to construct a message matching a given digest

### Password Hashing (PBKDF2)

```java
byte[] salt = new byte[16];
new SecureRandom().nextBytes(salt);

var spec = new PBEKeySpec(password, salt, ITERATIONS, 8 * HASH_BYTES);
SecretKeyFactory skf = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
byte[] hash = skf.generateSecret(spec).getEncoded();

// Store for later verification
String stored = ITERATIONS + "|"
    + Base64.getEncoder().encodeToString(salt) + "|"
    + Base64.getEncoder().encodeToString(hash);
```

- Never use plain `MessageDigest` for passwords — too fast; brute-forceable
- PBKDF2 is deliberately slow (many iterations) and salted (prevents precomputed table attacks)
- Store salt and iteration count alongside the hash; increase iterations over time as hardware improves

### Message Signing with `keytool` and `jarsigner`

Generate key pair:

```bash
keytool -genkeypair -keyalg DSA -keystore alice.jks -alias alice
```

Export certificate (public key):

```bash
keytool -exportcert -keystore alice.jks -alias alice -file alice.cer
```

Import into recipient's keystore:

```bash
keytool -importcert -keystore bob.jks -alias alice -file alice.cer
```

Sign a JAR:

```bash
jar cvf document.jar document.txt
jarsigner -keystore alice.jks document.jar alice
```

Verify a signed JAR:

```bash
jarsigner -verify -keystore bob.jks document.jar
```

### Certificate Signing by a CA

```bash
# CA generates root key and exports it
keytool -genkeypair -keystore acmesoft.jks -keyalg DSA -alias acmeroot
keytool -exportcert -keystore acmesoft.jks -alias acmeroot -file acmeroot.cer

# All employees import the CA root
keytool -importcert -keystore cindy.jks -alias acmeroot -file acmeroot.cer

# Employee submits a signing request
keytool -keystore alice.jks -alias alice -certreq -file alice.req

# CA signs Alice's certificate
keytool -keystore acmesoft.jks -gencert -alias acmeroot \
    -infile alice.req -outfile alice_signed.cer

# Cindy imports Alice's CA-signed certificate — no fingerprint verification needed
keytool -importcert -keystore cindy.jks -alias alice -file alice_signed.cer
```

> [!NOTE]
> Never import a certificate you don't fully trust. Once in a keystore, programs using that keystore will trust any signature it verifies.

### Authentication Problem

- A valid signature only proves the message was not tampered with and was signed by a matching private key
- It does NOT prove who owns that private key unless you trust the certificate chain
- Certificate Authorities (DigiCert, GlobalSign, Entrust) vouch for identity
- "Class 1" certificates only verify an email address; "Class 3" requires notarised identity verification

---

## Encryption

### Symmetric Ciphers (AES)

```java
Cipher cipher = Cipher.getInstance("AES");
cipher.init(Cipher.ENCRYPT_MODE, key);

// Encrypt blocks
int outputSize = cipher.getOutputSize(blockSize);
var outBytes = new byte[outputSize];
int outLength = cipher.update(inBytes, 0, blockSize, outBytes);

// Flush and pad final block
byte[] finalBytes = cipher.doFinal();                      // if no remaining data
byte[] finalBytes = cipher.doFinal(inBytes, 0, inLength);  // if partial block remains
```

Modes: `ENCRYPT_MODE`, `DECRYPT_MODE`, `WRAP_MODE`, `UNWRAP_MODE`

- AES — current standard; replaced DES (56-bit key; now insecure)
- `DES/CBC/PKCS5Padding` — algorithm/mode/padding format in the algorithm name string
- PKCS#5 padding — final block padded with bytes whose value equals the number of padding bytes added

### Key Generation

```java
KeyGenerator keygen = KeyGenerator.getInstance("AES");
keygen.init(new SecureRandom());
SecretKey key = keygen.generateKey();

// From raw bytes (e.g. password-derived)
byte[] keyData = new byte[16];  // 128-bit AES key
SecretKey key = new SecretKeySpec(keyData, "AES");
```

`SecureRandom`:

```java
var random = new SecureRandom();
byte[] seed = new byte[20];  // fill with truly random bits (hardware, user keystrokes, etc.)
random.setSeed(seed);
// If no seed provided, SecureRandom generates its own via thread timing measurements
```

- Never use `Random` for cryptographic keys — predictable from timestamp
- `SecureRandom` uses platform-level entropy sources

### Cipher Streams

```java
// Encrypting to a file
Cipher cipher = Cipher.getInstance("AES");
cipher.init(Cipher.ENCRYPT_MODE, key);
var out = new CipherOutputStream(new FileOutputStream(outputFile), cipher);
out.write(data);
out.flush();

// Decrypting from a file
cipher.init(Cipher.DECRYPT_MODE, key);
var in = new CipherInputStream(new FileInputStream(inputFile), cipher);
in.read(buffer);
```

- `CipherInputStream` / `CipherOutputStream` wrap `update` and `doFinal` calls transparently
- Padding is handled automatically on `flush()`

### Public Key Ciphers (RSA)

```java
// Generate key pair
KeyPairGenerator pairgen = KeyPairGenerator.getInstance("RSA");
pairgen.initialize(2048, new SecureRandom());
KeyPair keyPair = pairgen.generateKeyPair();
Key publicKey = keyPair.getPublic();
Key privateKey = keyPair.getPrivate();

// Wrap a symmetric key with the public key
Cipher cipher = Cipher.getInstance("RSA");
cipher.init(Cipher.WRAP_MODE, publicKey);
byte[] wrappedKey = cipher.wrap(aesKey);

// Unwrap with the private key
cipher.init(Cipher.UNWRAP_MODE, privateKey);
Key aesKey = cipher.unwrap(wrappedKey, "AES", Cipher.SECRET_KEY);
```

Hybrid encryption pattern (combining RSA + AES):
1. Generate a random AES key
2. Encrypt the plaintext with the AES key
3. Encrypt (wrap) the AES key with the recipient's RSA public key
4. Send both the wrapped AES key and the encrypted data
5. Recipient unwraps the AES key with their RSA private key
6. Recipient decrypts the data with the AES key

- RSA is far slower than AES — only use it for encrypting the small AES key
- RSA in the public domain since October 2000
- Recommended key size: 2048 bits minimum
