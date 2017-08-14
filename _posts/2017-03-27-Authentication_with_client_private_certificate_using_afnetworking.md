---
layout: single
title:  "Authentication with client private certificate using AFNetworking"
date:   2017-03-27 00:00:00
description: Useful code snippet for authentication with client private certificate using AFNetworking.
categories:
- AFNetworking
- Authentication
permalink: authentication_with_client_private_certificate_using_afnetworking
comments: true
---

In this post, I want to share useful code snippet for authentication with client private certificate using [AFNetworking](https://github.com/AFNetworking/AFNetworking). This is a combined solution which is built from few existing solutions found on [Github](https://github.com).

___

Authentication is one of the most crucial parts of many iOS applications. In some cases, the application needs to do it using client private certificate.

Here I will show you how to solve the above problem using popular and one of the most used libraries in iOS applications - [AFNetworking](https://github.com/AFNetworking/AFNetworking).

The good thing is that [AFNetworking](https://github.com/AFNetworking/AFNetworking) provides us with a block called `setSessionDidReceiveAuthenticationChallengeBlock:` to deal with such situation. We will use it and implement in such way so that as user credentials certificate data will be sent to the server.

First of all, let's assume you have a simple API manager:

``` objc
AFHTTPSessionManager *clientManager; // your API client
```

Subclass of `AFHTTPSessionManager` is also enough for this implementation.

Next, in one place where you create your API manager, we must set `setSessionDidReceiveAuthenticationChallengeBlock:` block.

``` objc
[clientManager setSessionDidReceiveAuthenticationChallengeBlock:^NSURLSessionAuthChallengeDisposition(NSURLSession * _Nonnull session, NSURLAuthenticationChallenge * _Nonnull challenge, NSURLCredential *__autoreleasing  _Nullable * _Nullable credential) {
	// auth logic goes here
}];
```

As you can guess when the server will ask your API manager for authentication credentials this block will be fired.

* `NSURLSession *session` - your client session, which you can use for example to cancel ongoing tasks and invalidate the current session.
* `NSURLAuthenticationChallenge *challenge` - this parameter can be used to determine which kind of credential is needed to pass.
* `NSURLCredential *credential` - this parameter is used to pass specific credentials back to the server.

Then we need to start to implement that block. Inside paste this snippet:

``` objc
// This is checking the server certificate
if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
    SecTrustResultType result;
    // This takes the serverTrust object and checkes it against your keychain
    SecTrustEvaluate(challenge.protectionSpace.serverTrust, &result);
    
    // If we want to ignore invalid server for certificates, we just accept the server
    if (result == kSecTrustResultProceed || result == kSecTrustResultUnspecified) {
        // When testing this against a trusted server I got kSecTrustResultUnspecified every time. But the other two match the description of a trusted server
        *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
    }
    
    return NSURLSessionAuthChallengeUseCredential;
}
```

I kept the original comments because think they explain what is done here very well. In brief, this `if` statement checks trust or not to the server you are talking with and passes the credentials from Keychain if the trust is OK or is unspecified.

Next `if` statement will check if authentication challenge is asking for the client certificate. If it is, we just pass user certificate credentials.

``` objc
if ([[challenge protectionSpace] authenticationMethod] == NSURLAuthenticationMethodClientCertificate) {
    // This handles authenticating the client certificate

    // Here you should pass reference for client private certificate, it can be done in many ways
    SecIdentityRef identityApp;
    SecCertificateRef certRef;
    SecIdentityCopyCertificate(identityApp, &certRef);
    
    SecCertificateRef certArray[1] = { certRef };
    CFArrayRef myCerts = CFArrayCreate(NULL, (void *)certArray, 1, NULL);
    CFRelease(certRef);
        
    NSURLCredential *userCredential = [NSURLCredential credentialWithIdentity:identityApp certificates:(__bridge NSArray *)myCerts persistence:NSURLCredentialPersistenceForSession];
    CFRelease(myCerts);
        
    *credential = userCredential;
        
    return NSURLSessionAuthChallengeUseCredential;
}
```

In the next condition, we look if authentication method is the default. In that situation, we can return `NSURLSessionAuthChallengePerformDefaultHandling` to use basic authentication or ignore it.

``` objc
if ([[challenge protectionSpace] authenticationMethod] == NSURLAuthenticationMethodDefault || [[challenge protectionSpace] authenticationMethod] == NSURLAuthenticationMethodNTLM) {
    // For normal authentication based on username and password. This could be NTLM or Default.
    
    return NSURLSessionAuthChallengePerformDefaultHandling;
}
```

If none of above will be called we return `NSURLSessionAuthChallengeCancelAuthenticationChallenge` to cancel authentication process.

``` objc
// If everything fails, we cancel the challenge.
return NSURLSessionAuthChallengeCancelAuthenticationChallenge;
```

So, this how looks minimal implementation of the client certificate based authentication. The final code will look like this:

``` objc
AFHTTPSessionManager *clientManager; // your API client

[clientManager setSessionDidReceiveAuthenticationChallengeBlock:^NSURLSessionAuthChallengeDisposition(NSURLSession * _Nonnull session, NSURLAuthenticationChallenge * _Nonnull challenge, NSURLCredential *__autoreleasing  _Nullable * _Nullable credential) {
    // This is checking the server certificate
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        SecTrustResultType result;
        // This takes the serverTrust object and checkes it against your keychain
        SecTrustEvaluate(challenge.protectionSpace.serverTrust, &result);
        
        // If we want to ignore invalid server for certificates, we just accept the server
        if (result == kSecTrustResultProceed || result == kSecTrustResultUnspecified) {
            // When testing this against a trusted server I got kSecTrustResultUnspecified every time. But the other two match the description of a trusted server
            
            *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        }
        
        return NSURLSessionAuthChallengeUseCredential;
    } else if ([[challenge protectionSpace] authenticationMethod] == NSURLAuthenticationMethodClientCertificate) {
        // This handles authenticating the client certificate
        
        // Here you should pass reference for client private certificate, it can be done in many ways
        SecIdentityRef identityApp;
        SecCertificateRef certRef;
        SecIdentityCopyCertificate(identityApp, &certRef);
    
        SecCertificateRef certArray[1] = { certRef };
        CFArrayRef myCerts = CFArrayCreate(NULL, (void *)certArray, 1, NULL);
        CFRelease(certRef);
        
        NSURLCredential *userCredential = [NSURLCredential credentialWithIdentity:identityApp certificates:(__bridge NSArray *)myCerts persistence:NSURLCredentialPersistenceForSession];
        CFRelease(myCerts);
        
        *credential = userCredential;
        
        return NSURLSessionAuthChallengeUseCredential;
    } else if ([[challenge protectionSpace] authenticationMethod] == NSURLAuthenticationMethodDefault || [[challenge protectionSpace] authenticationMethod] == NSURLAuthenticationMethodNTLM) {
        
        // For normal authentication based on username and password. This could be NTLM or Default.
        
        return NSURLSessionAuthChallengePerformDefaultHandling;
    }
    
    // If everything fails, we cancel the challenge.
    return NSURLSessionAuthChallengeCancelAuthenticationChallenge;
}];
```

___

Useful links:

* [Main reference](https://github.com/AFNetworking/AFNetworking/issues/2316) - it is the link of [Github](https://github.com) issue with a couple of code snippets written in older versions of [AFNetworking](https://github.com/AFNetworking/AFNetworking).