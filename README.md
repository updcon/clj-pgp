clj-pgp
=======

[![Build Status](https://travis-ci.org/greglook/clj-pgp.svg?branch=master)](https://travis-ci.org/greglook/clj-pgp)
[![Coverage Status](https://coveralls.io/repos/greglook/clj-pgp/badge.png?branch=master)](https://coveralls.io/r/greglook/clj-pgp?branch=master)
[![Dependency Status](https://www.versioneye.com/user/projects/53718e2314c1589a89000149/badge.png)](https://www.versioneye.com/clojure/mvxcvi:clj-pgp/0.5.4)
**master**
<br/>
[![Build Status](https://travis-ci.org/greglook/clj-pgp.svg?branch=develop)](https://travis-ci.org/greglook/clj-pgp)
[![Coverage Status](https://coveralls.io/repos/greglook/clj-pgp/badge.png?branch=develop)](https://coveralls.io/r/greglook/clj-pgp?branch=develop)
[![Dependency Status](https://www.versioneye.com/user/projects/53718e1914c1581079000056/badge.png)](https://www.versioneye.com/clojure/mvxcvi:clj-pgp/0.6.0-SNAPSHOT)
**develop**

This is a Clojure library which wraps the Bouncy Castle OpenPGP implementation.

## Usage

Library releases are published on Clojars. To use the latest version with
Leiningen, add the following dependency to your project definition:

[![Clojars Project](http://clojars.org/mvxcvi/clj-pgp/latest-version.svg)](http://clojars.org/mvxcvi/clj-pgp)

The main interface to the library is the `mvxcvi.crypto.pgp` namespace, which
provides a high-level API for working with PGP keys and data.

### PGP Keys

PGP stores keys in _keyrings_, which are collections of related asymmetric keys.
Public keyrings store just the public key from each keypair, and may store keys
for other people as well as keys controlled by the user. Secret keyrings store
both the public and private parts of a keypair, encrypted with a secret
passphrase.

```clojure
(require
  '[clojure.java.io :as io]
  '[mvxcvi.crypto.pgp :as pgp])

(def keyring
  (-> "mvxcvi/crypto/pgp/test-keys/secring.gpg"
      io/resource
      io/file
      pgp/load-secret-keyring))

(pgp/list-public-keys keyring)
; => (#<PGPPublicKey {...}> #<PGPPublicKey {...}>)

(def pubkey (first *1))

(pgp/key-id pubkey)
; => -7909697412827827830

(def seckey (pgp/get-secret-key keyring *1))

(pgp/key-algorithm seckey)
; => :rsa-general

(= (pgp/key-info pubkey)
   {:master-key? true,
    :key-id "923b1c1c4392318a",
    :strength 1024,
    :algorithm :rsa-general,
    :fingerprint "4C0F256D432975418FAB3D7B923B1C1C4392318A",
    :encryption-key? true,
    :user-ids ["Test User <test@vault.mvxcvi.com>"]})
; => true
```

### Data Encryption

Encryption and decryption of arbitrary data is supported using PGP's literal
data packets. The content is encrypted using a symmetric key algorithm, then
the key is encrypted using the given public key.

```clojure
(def content (.getBytes "my sensitive data"))

(def ciphertext
  (pgp/encrypt
    content pubkey
    :algorithm :aes-256
    :compress :zip
    :armor true))

(println (String. ciphertext))
;; -----BEGIN PGP MESSAGE-----
;; Version: BCPG v1.49
;;
;; hIwDkjscHEOSMYoBBADGcRtjKmBSAh6L2fVe/1BCZtEbME4zp6GqilETzOYyi5HL
;; Vee++PI03KluhW32i359ycvOre92yHaApcDBRXGwdYBT/hx8ryXov3I1wvZMS/iK
;; Iex91VxkquJnvZvi6/qy3f6WFgLBHT2GCKy+Um4YU2OstykHZP7Gsbr5MZ04K8ks
;; 71TaictIOx2qukbpwnIVNzOl5GeaPy5FiVbntl0Wc3lESD2A9l2pDENyicg=
;; =cvks
;; -----END PGP MESSAGE-----

(defn get-privkey
  "Define a function to get an unlocked private key by id. This is used to
  check for a key matching the one the data packet is encrypted for."
  [id]
  (some->
    keyring
    (pgp/get-secret-key id)
    (pgp/unlock-key "test password")))

(def cleartext (pgp/decrypt ciphertext get-privkey))

(println (String. cleartext))
;; my sensitive data
```

There are also more primitive `encrypt-stream` and `decrypt-stream` functions
which will wrap output and input streams, respectively.

### Signatures

The library also provides support for signature generation and verification.

```clojure
(def privkey (pgp/unlock-key seckey "test password"))
(def sig (pgp/sign content :sha1 privkey))

(= (pgp/key-id sig) (pgp/key-id privkey))
; => true

(pgp/verify content sig pubkey)
; => true
```

### Serialization

The library provides functions for encoding in both binary and ASCII formats.

```clojure
(pgp/encode sig)
; => #<byte[] [B@51e4232>

(print (pgp/encode-ascii pubkey))
;; -----BEGIN PGP PUBLIC KEY BLOCK-----
;; Version: BCPG v1.49
;;
;; mI0EUr3KFwEEANAfzcKxWqBYhkUGo4xi6d2zZy2RAewFRKVp/BA2bEHLAquDnpn7
;; abgrpsCFbBW/LEwiMX6cfYLMxvGzbg5oTfQHMs27OsnKCqFas9UkT6DYS1PM9u4C
;; 3qlJytK9AFQnSYOrSs8pe6VRdeHZb7FM+PawqH0cuoYfcMZiGAylddXhABEBAAE=
;; =Hnjf
;; -----END PGP PUBLIC KEY BLOCK-----

(let [ascii (pgp/encode-ascii pubkey)]
  (= ascii (pgp/encode-ascii (pgp/decode-public-key ascii))))
; => true
```

## License

This is free and unencumbered software released into the public domain.
See the UNLICENSE file for more information.
