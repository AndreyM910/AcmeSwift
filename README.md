# AcmeSwift

This is a **work in progress** Let's Encrypt (ACME v2) client written in Swift. 

It fully uses the Swift concurrency features introduced with Swift 5.5 (`async`/`await`).

Although it _might_ work with other certificate providers implementing ACMEv2, this has not been tested at all.


## Note
This library doesn't handle any ACME challenge at all by itself.
Publishing the challenge, either by creating DNS record or exposing the value over HTTP, is your full responsibility. 

Additionally, this library currently doesn't provide any way to generate a private key or a CSR.


## Usage

Create an instance of the client:
```swift
let acme = try await AcmeSwift()

```

When testing, preferably use the Let's Encrypt staging endpoint:
```swift
let acme = try await AcmeSwift(acmeEndpoint: AcmeServer.letsEncryptStaging)

```


### Account

- Create a new Let's Encrypt account:

```swift
let account = acme.account.create(contacts: ["my.email@domain.com"], validateTOS: true)
```

The information returned by this method is an `AcmeAccountInfo` object that can be directly reused for authentication. 
For example, you can encode it to JSON, save it somwewhere and then decode it in order to log into your account later.

⚠️ This Account information contains a private key and as such, **must** be stored securely.


<br/>

- Reuse a previously created account:

Option 1: Directly use the object returned by `account.create(...)`
```swift
try acme.account.use(account)
```

Option 2: Pass credentials "manually"
```swift
let credentials = try AccountCredentials(contacts: ["my.email@domain.tld"], pemKey: "private key in PEM format")
try acme.account.use(credentials)
```

If you created your account using AcmeSwift, the private key in PEM format is stored into the `AccountInfo.privateKeyPem` property.

<br/>

- Deactivate an existing account:

⚠️ Only use this if you are absolutely certain that the account needs to be permanently deactivated. There is no going back!

```swift

try await acme.account.deactivate()
```

### Orders (certificate requests)

 Create an order for a new certificate:
 
 ```swift
 
 let order = try await acme.orders.create(domains: ["mydomain.com", "www.mydomain.com"])
 ```

<br/>

Get the order authorizations and challenges: 
```swift
let authorizations = try await acme.orders.getAuthorizations(from: order)
```

<br/>

You now need to publish the challenges. AcmeSwift provides a way to list the pending HTTP or DNS challenges:
```swift
let challengeDescriptions = try await acme.orders.describePendingChallenges(from: order, preferring: .http)
for desc in challengeDescriptions {
    if desc.type == .http {
        print("\n • The URL \(desc.endpoint) needs to return \(desc.value)")
    }
    else if desc.type == .dns {
        print("\n • Create the following DNS record: \(desc.endpoint) TXT \(desc.value)")
    }
}
```
Achieving this depends on your DNS provider and/or web hosting solution and is outside the scope of AcmeSwift.

<br/>

Once the challenges are published, we can ask Let's Encrypt to validate them:
```swift
let updatedChallenges = try await acme.orders.validateChallenges(from: order, preferring: .http)
```

<br/>

Now we have to wait a bit, or periodically query the ACMEv2/LetsEncrypt provider about the status of the challenges.
AcmeSwift provides a convenience method that will do exactly this: periodically check the status of the order until all the challenges have been processed:
```swift
let failedAuthorizations = try await acme.orders.wait(order: order, timeout: 30 /* in seconds*/) 
guard failedAuthorizations.count == 0 else {
    fatalError("Some challenges were not validated! \(failedAuthorizations)")
}
```

<br/>

Once all the authorizations/challenges are valid, we can finalize the Order by sending the CSR in PEM format:
```swift
let finalizedOrder = try await acme.orders.finalize(order: order, withPemCsr: "...")
```
Note: AcmeSwift currently doesn't implement any way of generating CSRs or private keys.


### Certificates

Download a certificate:

> This assumes that the corresponding Order has been finalized successfully, meaning that the Order `status` field is `valid`.

```swift
let certs = try await acme.certificates.download(for: finalizedOrder)
```

This return a list of PEM-encoded certificates. The first item is the actual certificate for the requested domains.
The following items are the other certificates required to establish the full certification chain (issuing CA, root CA...).

The order of the items in the list is directly compatible with the way Nginx expects them; you can concatenate all the items into a single file and pass this file to the `ssl_certificate` directive.

<br/>

Revoke a certificate:
```swift
try await acme.certificates.revoke(certificatePEM: "....")
```
