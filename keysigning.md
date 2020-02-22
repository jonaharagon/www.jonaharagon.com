---
layout: page
permalink: /keys/keysigning/
title: Keysigning Tutorial
icon: "fas fa-key"
---

### What is keysigning?

Keysigning refers to [digitally signing](https://en.wikipedia.org/wiki/Digital_signature){:.itl} someone else's public key using your own. Users of PGP sign one another's keys to indicate to any third party that the signer trusts the signee. This enables someone who trusts the signer to extend her trust to the signee as well. In this way, a [web of trust](https://en.wikipedia.org/wiki/Web_of_trust){:.itl} is built.

To sign the keys of others you will need your own GPG key/identity. If you haven't already done so, download GPG and [create a key](https://mikaela.info/r/gpg){:.itl}.

Note that you should never sign the keys of people you don't know. Keys can contain any email address, so it would be bad if the key of an imposter account was signed. Be careful with which keys you choose to sign with your own.

### Importing my public key

Before you can sign a key, it will need to exist in your keychain. Download my public key: [gpg.asc](/assets/files/gpg.asc){:.itl}:

```
$ wget https://jonaharagon.com/assets/files/gpg.asc
```

Import the public key you've just downloaded into your GPG keyring:

```
$ gpg --import gpg.asc
```

Alternatively you can find my key via `keys.openpgp.org` (a public keyring):

```
$ gpg --keyserver keys.openpgp.org --search-keys 0x6A957C9A9A9429F7
gpg: data source: http://keys.openpgp.org:11371
(1)	Jonah Aragon <jonah@privacytools.io>
	Jonah Aragon <jonah@triplebit.net>
	  256 bit EDDSA key 0x6A957C9A9A9429F7, created: 2020-02-20
Keys 1-1 of 1 for "jonah@triplebit.net".  Enter number(s), N)ext, or Q)uit > 1
```

### Signing the key

Just enter the following command:

```
$ gpg --sign-key
```

It will ask you if you want to sign the key with your own, enter `Y`. It may also ask you if you want to sign all the IDs on the key, which you can enter `Y` for as well. And that's it!

### Exporting the key

Now you've signed my GPG key, but that signature only exists on your machine. You'll want to send it back to me with your signature attached, saying you're vouching for my key. You can export my key:

```
$ gpg --export --armor 0x6A957C9A9A9429F7
```

Which will show a long string that looks like this, which you'll want to copy and send me:

<pre class="pre-scrollable"><code>
{% include gpg.asc %}
</code></pre>

You can also export to a file:

```
$ gpg --export --armor 0x6A957C9A9A9429F7 > exported_key.asc
```

(Which will export to a file named `exported_key.asc` in your current directory)

You can send me the signed key via any [contact method](/){:.itl}. And thank you! Key signatures help build trust in the PGP world.
