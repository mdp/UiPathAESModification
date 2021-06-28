# UiPathAESModification

An example of modifying an AES-CBC ciphertext in UiPath

## Introduction

This is just a simple UiPath project to demonstrate how it's possible to modify an AES-CBC ciphertext. It's not a problem that's unique to UiPath,
but instead it's a fundemental issue with the AES-CBC mode. UiPath has subsequently updated their cryptography activity to use AES-GCM which does
not suffer from this issue.

## Usage

Because the salt is changed on each encryption, you'll need to run the project and save the ciphertext output in the logs. Then modify this
ciphertext by modifying the IV key, as described in [this article](https://www.linkedin.com/pulse/encryption-authentication-mark-percival/)

Example:

In the project "$1000.00" becomes

`vh2SlWQynX/UxBVuLdKie0JPhdFEHzVCTv1IoRzM8sckV5RI6hWZvg==`

`be1d929564329d7fd4c4156e2dd2a27b424f85d1441f35424efd48a11cccf2c724579448ea1599be`

Which can be broken down further to:

```
be1d929564329d7f d4c4156e2dd2a27b424f85d1441f3542 
8 byte salt      16 byte IV Key

                 4efd48a11cccf2c724579448ea1599be
                 16 byte ciphertext block           
```

From the above we need to modify the 2nd byte of the IV key (0xC4) to change the '1' (0x31) character into a '9' (0x39)

Since we know that the first step in the encryption process is to XOR 0xC4 with 0x31, we just need to change the IV key to yield a 0x39
when that process is reversed.

So...

```
OxC4 ^ 0x31 -> 0xf5
0xf5 ^ 0x39 -> 0xCC
```

Let's change c4 in the IV key to cc and wrap it back up:

`be1d929564329d7fd4cc156e2dd2a27b424f85d1441f35424efd48a11cccf2c724579448ea1599be`

Which in base64 is:

`vh2SlWQynX/UzBVuLdKie0JPhdFEHzVCTv1IoRzM8sckV5RI6hWZvg==`

Now lets compare the two ciphertexts:

Old -> `vh2SlWQynX/UxBVuLdKie0JPhdFEHzVCTv1IoRzM8sckV5RI6hWZvg==`

New -> `vh2SlWQynX/UzBVuLdKie0JPhdFEHzVCTv1IoRzM8sckV5RI6hWZvg==`

And then we'll run this back through the UiPath decrypt process, which should yield "$9000.00"

