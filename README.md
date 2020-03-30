# Securing Websites with Certificates

This site explains how websites use certificates issued by a Certificate Authority to secure their site. Along the way, we'll explain how to examine the certificate a website uses, how the Certificate Authority system works, and how to get a certificate for your website for free using CertBot from Let's Encrypt.

## Introduction to Certificates

When you browse to a website that is secure, your web browser will show that it is using HTTPS and also some form of security indicator, typically a lock icon:

![lock icon](/images/lock-icon.png)

This indicates that the browser has validated the identity of the website using a `certificate`. A certificate is a collection of data that is digitally signed by a `certificate authority`. One important piece of information that is signed is the website's identity, in this case, `www.byu.edu`. Another important piece is the website's `public key`.

### An aside on public key cryptography...

In public key cryptography, an organization owns both a `private key` and a `public key`. The organization keeps its private key secret and freely advertises its public key to the world.

One operation that can be performed with these keys is that someone can encrypt data with the organization's public key, and this can only be decrypted by the organization's private key:

![public key encryption](/images/public-key-encryption.png)

Another operation that can be performed with these keys is that the organization can sign data with its private key, and this can be verified as authentic by anyone with the organization's public key:

![public key signing](/images/public-key-signing.png)

When DigiCert signs a certificate for BYU, it is attesting that the owner of the `public key` in the certificate has the authority to operate the domain `www.byu.edu`. Since BYU keeps its `private key` secret, it is the only entity that can prove it owns the `public key` listed in the certificate.

### An HTTPS connection means all data is encrypted...

The lock icon also indicates that the browser is using the HTTPS protocol. By using HTTPS your web browser is encrypting all of the data sent between your browser and the website so that only these two parties can access it. In this case, my browser is encrypting data it exchanges with `www.byu.edu`, and that site encrypts any data it sends back to my browser.

Thus when we say a website is `secure`, what we mean is that the identity of the website has been validated and that the connection to the website is encrypted.

## Certificate details

If you click on the lock icon, you can learn more about what kind of certificate the website is using. In this case, I'm using Firefox, so it shows this:

![certificate dropdown](/images/certificate-dropdown.png)

This doesn't add much new information! It just confirms that the site we are browsing is `www.byu.edu` and that the connection is secure. If you click the arrow, you will see:

![certificate dropdown #2](/images/certificate-dropdown-2.png)

This adds one new piece of information -- the certificate has been "verified" by DigiCert Inc. This means that DigiCert verified BYU as being the rightful owner of `www.byu.edu` and then issued a certificate to BYU that is signed by DigiCert.

### Some info on cryptography...

Now we'll click `More Information`:

![certificate information](/images/certificate-info.png)

This shows us a little more information. In addition to the website name and the authority that issued the certificate, it shows the expiration date. At the bottom it lists some of the technical details of how encryption is occurring. A class on computer security or cryptography would explain this in more detail, but briefly:

* TLS -- indicates the protocol being used is TLS, the standard protocol for securing connections on the Internet.

* ECDHE -- indicates that TLS is using Elliptic-curve Diffie-Helman to decide on an encryption key to use for this connection. Using ECDHE means that the connection derives ephemeral (short-lived) keys, providing an important property called forward secrecy.

* RSA -- indicates the algorithm used to verify the server's identity.

* AES_128_GCM -- indicates the AES algorithm is used to encrypt data for this connection, using a 128-bit key.

* SHA256 -- indicates the SHA-256 algorithm is used to ensure that messages exchanged cannot be modified without being detected, using 256-bit keys.

* TLS 1.2 -- a modern version of TLS. The newest version is 1.3.

### So what does this mean?

All of this is pretty standard, strong security for a website. You're getting important properties:

* authentication -- the website is who they say they are

* confidentiality -- all data you exchange with the website can only be seen by your browser and their web server

* integrity -- messages can't be changed without being detected by your browser and their web server

* forward secrecy -- if an attacker breaks the encryption key for one connection it doesn't help them break the encryption key for any other connection

The AES algorithm being used for encryption is strong enough that an attacker using the most advanced supercomputer in the world would need over 800 quadrillion years to crack it. This is essentially considered unbreakable.

You may notice some websites, such as `lastpass.com` using somewhat stronger encryption algorithms (AES with 256 bit keys instead of 128, SHA with 384 bits instead of 256, TLS 1.3).  In this case "stronger" is a relative term. It is hard to improve on something that is uncrackable over quadrillions of years.

### Certificate Chains

There is one more thing to do ... click on `View Certificate`. There is quite a lot you can see in this case, but I will focus on the most important thing:

![certificate-chain](/images/certificate-chain.png)

What you see here is a `certificate chain`. A certificate for `*.byu.edu` was signed by the `DigiCert SHA2 High Assurance Server CA` certificate, which was in turn signed by the `DigiCert High Assurance EV Root CA`. The right-most certificate, the `Digicert Root CA`, signed a certificate for the `DigiCert Server CA`, which in turn signed a certificate for `BYU`. Your web browser validates this entire chain of certificates to make sure everyone is who they say they are.

## The Certificate Authority System

This gets us to some really important questions. Who says that `DigiCert Root CA` is allowed to sign certificates? And who can they sign certificates for?

There is a collection of certificates, known as the `root store`, that has all of the certificates that are at the "root" of a chain of trust. These certificates are all assumed to be trusted, meaning the organizations that own the associated private keys for these certificates are allowed to sign certificates for other sites. `DigiCert Root CA` is one of the certificates in a browser or OS root store.

Who decides which certificates get put in the root store? Today, the major OS vendors (Apple, Microsoft, Google, Linux distributions) and the major browser vendors (Apple, Microsoft, Google, Mozilla) get to decide. Each OS and each browser ships with its root store. There is a great deal of overlap among these root stores, because nobody wants their website to work (be authenticated, get a lock icon) on some browsers or laptops or phones and not others. But there are some differences as well.

These major vendors have a lot of power. For example, [Google decided to remove certificates owned by Symantec its root stores](https://security.googleblog.com/2017/09/chromes-plan-to-distrust-symantec.html) after there were repeated instances where Symantec trusted some organizations with issuing certificates without appropriate oversight. Google likewise [removed the CNNIC (China Internet Network Information Center) root from its root store](https://security.googleblog.com/2015/03/maintaining-digital-certificate-security.html), and Mozilla followed suit.

### Some key points about Certificate Authorities

Technically, any Certificate Authority can sign a certificate for any other organization. In fact, they can delegate authority to other organizations, giving them the power to *also* sign certificates for other organizations. This is why a browser may receive a `chain` of certificates, which should eventually be rooted in one of the certificates in the root store.

Certificate Authorities have strong incentives to not misuse their level of trust, so they typically join an organization that requires regular, independent audits to ensure they are using best practices.

That said, there have been instances of Certificate Authorities being hacked. A notable instance is when [DigiNotar was hacked](https://slate.com/technology/2016/12/how-the-2011-hack-of-diginotar-changed-the-internets-infrastructure.html), leading to the compromise of thousands of Iranian Gmail accounts.

### Certificate Transparency

To help detect misuse, Google started a new system called Certificate Transparency. Essentially, what happens with this system is that every time a Certificate Authority creates a new certificate, they add it to a publicly-accessible log. When web browsers get a certificate from a website, they check the transparency log to make sure the certificate is listed before accepting it as valid.

Observe what this does for an attacker. If an attacker hacks into a Certificate Authority and issues a fake certificate for Gmail, it has two choices:

* If they don't add the certificate to the transparency system, no browser will accept the fake certificate.

* If they add the certificate to the transparency system, web browsers will accept it as real. *But*, web site owners can also monitor the transparency system, see the fake certificate, and report it. So the attacker's attempt will be detected and browser vendors will mark the certificate as bad as quickly as possible.

## Certificates are now free

Not too many years ago, getting a certificate for a website was a cumbersome and expensive process. The process was cumbersome because it required proving your identity to someone, which is typically done manually, by inspecting corporate documents and identities. The process was expensive relatively speaking -- typically several hundred dollars per year, which priced out many individuals. Many companies and organizations simply didn't bother. **As a result, most of the web was not secure.**

This situation changed drastically when a new organization, named [Let's Encrypt](https://letsencrypt.org/) created a system to provide free certificates to everyone *and* to provide software that automatically obtained and installed certificates. This made it both easy and cheap to obtain a certificate for a website -- the only cost being a small amount of time to set it up. This led to a major public relations effort to encourage every website to obtain a certificate -- to [encrypt the web](https://www.eff.org/encrypt-the-web).

Let's Encrypt now provides certificates to 200 million websites. And the vast majority of websites are now encrypted:

![https status](/images/https-status.png)

We've made so much progress that Google has talked about [phasing out the lock icon](https://blog.chromium.org/2018/05/evolving-chromes-security-indicators.html), and instead showing "Not Secure" on websites without a certificate.

![chrome-security-indicators](/images/chrome-security-indicators.png)

![chrome-not-secure](/images/chrome-not-secure.png)

## Getting a certificate

Get a certificate is really easy. The EFF has created [Certbot](https://certbot.eff.org/) to automatically get and install a certificate for your website. On their web page you can select your web server and OS to get customized instructions for how to use it:

![certbot](/images/certbot.png)

Here are the [instructions for Nginx on Ubuntu 18.04](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx.html). In step 4, use the first command shown, which will automatically configure Nginx with a certificate for all of your Nginx websites.

```
sudo certbot --nginx
```

Alternatively, you can get and configure a certificate for a single website:

```
sudo certbot --nginx -d example.com
```
