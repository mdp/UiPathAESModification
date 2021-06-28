*In this article I'm going to detail a common encryption shortcoming and the ways an attacker can exploit your mis-use of an encryption process. In the process of writing about encryption in the automation space I discovered that UiPath was using an older encryption method in one of their activities.*

*I disclosed this issue to UiPath on June 10th and withheld writing about it until* [*they updated their encryption activity to an authenticated encryption option (AES-GCM) on June 23rd*](https://docs.uipath.com/releasenotes/docs/uipath-cryptography-activities#v121)*. This was not really a flaw in the UiPath implementation, but a common issue with any unauthenticated mode of encryption where the user may not fully understand the security implications. UiPath moved quickly to update their library by adding an authenticated option as the default and deprecating the older algorithms.*

Background
----------

When we talk about symmetric encryption (using a secret key to decrypt and encrypt a message), we're usually talking about the AES standard, a very common and secure cryptographic method. But AES is a very simple process and when you start encrypting real data it gets a bit more complex. One of those complexities is the 'mode' of encryption. Older modes (in this case CBC, or Cipher Block Chaining) left it up to the user to verify that the message was not altered in some way. Many people assume that without the secret key an attacker would not be able to alter an encrypted message, but, as we'll soon see, that's not always the case.

Let's give a real world, if convoluted, automation example: I have a process where an employee selects one of 4 bank accounts to transfer money to. I don't want to let them arbitrarily enter an account (they might enter their own), so I encrypt 4 bank accounts with a passphrase. In a dropdown the employee chooses between the accounts and behind the scenes it's actually sending the encrypted version of the bank account number. Later on in another process it decrypts the ciphertext with the passphrase to get back the account number to perform a transfer.

Because they don't know the passphrase I assume it's impossible to craft a ciphertext with their own bank account. But that may not be true depending on the encryption method. To put it simply, if a message is not authenticated before decryption then an outsider can alter the decrypted message. And if they know the original content, they can alter small messages to be anything they like. In this case, the employee could take a ciphertext which they know the plaintext of (eg. "Acct #12345678") and turn it into an account number of their choosing (eg. "Acct #5551337"), all without ever knowing the passphrase/secret key.

The nitty gritty
----------------

In order to simplify the attack, I'm only going to modify one character of a string, in this case "$1000.00" into "$9000.00".

Let start with the ciphertext text typically returned by an AES-CBC encryption function (this is verbatim from UiPath's older version of [Encrypt Text](https://docs.uipath.com/activities/docs/encrypt-text)).

`0cgbAblSlRp4ySuB1VTIkcJGFOdoAnEQImMqbiIiNfrLcQ8FMcZMrQ==`

This obviously looks like gibberish, but it's actually a base64 encoded string that breaks down into:

`salt[8 bytes] + initialization vector[16 bytes] + cipherblock [16 bytes per block]`

First let's start by converting the Base64 encoding of the ciphertext back to its original hex format for easier parsing.

`d1c81b01b952951a78c92b81d554c891c24614e76802711022632a6e222235facb710f0531c64cad`

Which can be broken down further into:

```
d1c81b01b952951a 78c92b81d554c891c24614e768027110
Salt             IV

22632a6e222235facb710f0531c64cad
CipherBlock
```

Although our encrypted text is "$1000.00" (8 bytes long), AES works on blocks of 16 bytes, so it's padded with extra bytes which tell the encryption algorithm which bytes to ignore.

The most important thing to understand is that if we know the plaintext, we can arbitrarily modify individual characters by modifying the IV value in the same position. This is because the first step of AES-CBC encryption is to XOR the first plaintext with the IV. *(*[*XOR'ing explanation for the uninitiated*](https://hackernoon.com/xor-the-magical-bit-wise-operator-24d3012ed821)*)*

So in our example, let's modify the '1' in "$1000.00" to a '9'. We know it's in the 2nd position, so let's modify the IV in the 2nd position ('0xC9') and change it to something that will result in a "9" decrypting.

The modification would look something like this. We want to turn a '1' character (0x31 in hex) into a '9' character (0x39 in hex). Because AES just XOR's the IV with the plaintext and we know that the character is a '1', we can find the value 0xC9 is XOR'd with to make 0x39. Basically, 0xC9 XOR 0x31 yields 0xF8, 0xF8 XOR 0x39 is 0xC1.

So now we just need to replace 'c9' in the ciphertext with 'c1'.

`d1c81b01b952951a78c12b81d554c891c24614e76802711022632a6e222235facb710f0531c64cad`

And Base64 encode it for the UiPath decryption routine to consume:

`0cgbAblSlRp4wSuB1VTIkcJGFOdoAnEQImMqbiIiNfrLcQ8FMcZMrQ==`

Now it looks very similar to our original ciphertext:

`0cgbAblSlRp4ySuB1VTIkcJGFOdoAnEQImMqbiIiNfrLcQ8FMcZMrQ==`

But you'll notice that the `Rp4y` became `Rp4w`

And lets run it through the old [Decrypt Text](https://docs.uipath.com/activities/docs/decrypt-text) again and see what we get:

Now instead of $1000.00 I'm getting $9000.00, which seems like it was worth the effort.

If you want to play with this yourself, you can find the [UiPath project on Github](https://github.com/mdp/UiPathAESModification/).

Conclusion
----------

The solution to the above problem is to use a mode of encryption that authenticates the ciphertext before decrypting it. In the case of UiPath, their latest version uses AES-GCM which does exactly that. If you're using UiPath's "Encrypt Text" activity it would be wise to move to the AES-GCM algorithm regardless of how you're planning to use it.

Add alt text![The new UiPath encryption options, with AES-GCM as the preferred option](https://media-exp3.licdn.com/dms/image/C5612AQE_arf1wwthRg/article-inline_image-shrink_1000_1488/0/1624892490755?e=1630540800&v=beta&t=om-VjNtfFtWYVT0mdYwxMFwAJb9nQNmt9XoNxMLd3Go)

But the real take-away from this article should be how easy it is to mess up an encryption process. As developers we tend to enjoy using encryption in clever ways, but the result may not only be less secure than we intended. And worse still, it may actually open us up to more attacks because of the false sense of security it imparts.

Encryption is an important tool in securing our automations, but only if you use it correctly. I'd think about it like this - use SSL, don't implement your own version of SSL.
