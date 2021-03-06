clevis(1) -- Automated decryption policy framework
==================================================

## SYNOPSIS

`clevis` COMMAND [OPTIONS]

## OVERVIEW

Clevis is a framework for automated decryption policy. It allows you to define
a policy at encryption time that must be satisfied for the data to decrypt.
Once this policy is met, the data is decrypted.

Clevis is pluggable. Our plugins are called pins. The job of a pin is to
take a policy as its first argument and plaintext on standard input and to
encrypt the data so that it can be automatically decrypted if the policy is
met. Lets walk through an example.

## HTTP ESCROW

When using the HTTP pin, we create a new, cryptographically-strong, random key.
This key is stored in a remote HTTP escrow server (using a simple PUT or POST).
Then at decryption time, we attempt to fetch the key back again in order to
decrypt our data. So, for our configuration we need to pass the URL to the key
location:

    $ clevis encrypt http '{"url":"https://escrow.srv/1234"}' < PT > JWE

To decrypt the data, simply provide the ciphertext (JWE):

    $ clevis decrypt < JWE > PLAINTEXT

Notice that we did not pass any configuration during decryption. The decrypt
command extracted the URL (and possibly other configuration) from the JWE
object, fetched the encryption key from the escrow and performed decryption.

For more information, see `clevis-encrypt-http`(1).

## TANG BINDING

Clevis provides support for the Tang network binding server. Tang provides
a stateless, lightweight alternative to escrows. Encrypting data using the Tang
pin works much like our HTTP pin above:

    $ clevis encrypt tang '{"url":"http://tang.srv"}' < PT > JWE
    The advertisement contains the following signing keys:

    _OsIk0T-E2l6qjfdDiwVmidoZjA

    Do you wish to trust these keys? [ynYN] y

As you can see above, Tang utilizes a trust-on-first-use workflow.
Alternatively, Tang can perform entirely offline encryption if you pre-share
the server advertisment. Decryption, too works like our first example:

    $ clevis decrypt < JWE > PT

For more information, see `clevis-encrypt-tang`(1).

## SHAMIR'S SECRET SHARING

Clevis provides a way to mix pins together to create sophisticated unlocking
and high availability policies. This is accomplished by using an algorithm
called Shamir's Secret Sharing (SSS).

SSS is a thresholding scheme. It creates a key and divides it into a number of
pieces. Each piece is encrypted using another pin (possibly even SSS
recursively). Additionally, you define the threshold `t`. If at least `t`
pieces can be decrypted, then the encryption key can be recovered and
decryption can succeed.

For example, let's create a high-availability setup using Tang:

    $ cfg='{"t":1,"pins":{"tang":[{"url":...},{"url":...}]}}'
    $ clevis encrypt sss "$cfg" < PT > JWE

In this policy, we are declaring that we have a threshold of 1, but that there
are multiple key fragments encrypted using different Tang servers. Since our
threshold is 1, so long as any of the Tang servers are available, decryption
will succeed. As always, decryption is simply:

    $ clevis decrypt < JWE > PT

For more information, see `clevis-encrypt-tang`(1).

## LUKS BINDING

Clevis can be used to bind an existing LUKS volume to its automation policy.
This is accomplished with a simple command:

    $ clevis bind luks -d /dev/sda tang '{"url":...}'

This command performs four steps:

1. Creates a new key with the same entropy as the LUKS master key.
2. Encrypts the new key with Clevis.
3. Stores the Clevis JWE in the LUKS header with LUKSMeta.
4. Enables the new key for use with LUKS.

This disk can now be unlocked with your existing password as well as with
the Clevis policy. Clevis provides two unlockers for LUKS volumes. First,
we provide integration with Dracut to automatically unlock your root volume
during early boot. Second, we provide integration with UDisks2 to
automatically unlock your removable media in your desktop session.

For more information, see `clevis-bind-luks`(1).

## AUTHOR

Nathaniel McCallum &lt;npmccallum@redhat.com&gt;

## SEE ALSO

`clevis-encrypt-http`(8),
`clevis-encrypt-tang`(8),
`clevis-encrypt-sss`(8),
`clevis-bind-luks`(8),
`clevis-decrypt`(8),
