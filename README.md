# SafeNetTokenSigner
A SignTool alternative that can unlock SafeNet hardware tokens without user interaction

## But why?
Okay, so the story goes like this:

You are creating an awesome application for a client, with a mobile frontend. Development is slowly nearing completion so it's time to start thinking about the realities of product deployment to the end-users. Of course, mobile clients being mobile clients, you need to sign your frontend with a code signing certificate.

So you go to your client and nicely ask them for a code signing certificate - they want to see their name on the app, after all. And after a week or two, the client returns with a publicly trusted certificate. However, this certificate is stored on this ultra-secure custom USB stick/token that requires a special application (SafeNet Authentication Client) to access it. AND every time you want to use the certificate, you

1. Must be in an interactive session
2. Must enter a PIN
3. Cannot do steps 1 and 2 over remote desktop

And just like that, your beautiful automated CI/CD pipeline is useless because someone needs to do the signing manually every time (since SignTool doesn't take a PIN for this scenario and the private key cannot be extracted from the USB stick). Bummer.

## The challenge
Not sure about you, but I like our automated build pipeline. I don't want to do manual work on deployments, even if it takes 5 minutes once a month. So where SignTool fails, StackOverflow might have answers, right? Indeed, after some browsing I came across a question where someone needed to solve basically the same issue: https://stackoverflow.com/a/47894907/1453109

The answer was to create a custom signing application that calls some Win32 APIs to perform the signing. Using these APIs, it is possible to actually unlock the SafeNet token. Awesome, with one huge caveat: this works nicely for standard executables like EXE and DLL but fails to sign an appxbundle (a UWP distribution format for Windows 10 Store apps).

Since I'm not a C++ developer by nature, I struggled with the Win32 APIs a lot and ended up asking [this StackOverflow question](https://stackoverflow.com/questions/48804073/signing-an-appxbundle-using-cryptuiwizdigitalsign-api/48905245). And that finally led me to the solution.

## The solution
Since we needed to sign both a PE executable (an installer) and the appxbundle (the Windows 10 Store app), we basically had to emulate what SignTool does, with the added capability to unlock a SafeNet hardware token. The second part was taken from the first SO question linked above, while the ability to sign appxbundles was taken from the answer to the second one.

We are doing four things here:

1. Unlocking the USB token by
    * acquiring the cryptogtaphic provider context using [CryptAcquireContext](https://msdn.microsoft.com/en-us/library/windows/desktop/aa379886%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396)
    * providing the PIN to the cryptographic provider to allow access to the private keys using [CryptSetProvParam](https://msdn.microsoft.com/en-us/library/windows/desktop/aa380276(v=vs.85).aspx)
2. Opening Windows' certificate store 
    *  The SafeNet app automatically exports the token's public keys into current user's personal certificate store
    * Using [CertOpenStore](https://msdn.microsoft.com/en-us/library/windows/desktop/aa376559%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396)
3. Finding the correct code signing certificate by it's SHA1 thumbprint
    * Using [CertFindCertificateInStore](https://msdn.microsoft.com/en-us/library/windows/desktop/aa376064(v=vs.85).aspx)
4. Signing the target file
    * Must support both standard EXE/DLL etc. and the appxbundle format
    * Using [SignerSignEx2](https://msdn.microsoft.com/en-us/library/windows/desktop/hh968155(v=vs.85).aspx)
    
And that's it! This requires zero user interaction and can be used without an interactive session within a TFS CI build, given the USB token is plugged into the build agent machine and the correct command line arguments are passed.

## Usage
If you want to use this, feel free to compile the exe from source. The command line arguments of the app look like this:

`signer.exe <thumbprint> <container name> <PIN> <timestamp URL> <mode> <path>`

Arguments:
* the signing certificate thumbprint (SHA1, e.g. `8cfaa82e5c3842ffee22d82d8ff812b3a1de1ebb`)
* private key container name - you can get this by looking into the SafeNet app (see the first SO question)
* the PIN used to protect the token's private keys
* timestamp URL (the code will perform timestamping automatically, e.g. `http://timestamp.verisign.com/scripts/timstamp.dll`)
* mode is either `pe` or `appx`, depending on what file you are trying to sign (`pe` for normal EXE/DLL etc.)
* the path to the file you want to sign (can be relative)

## Limitations
This code is provided without any guuarantees (well, duh!). I tested it on our project and it seems to be performing according to expectations. Tested on Windows 10, signing a PE EXE and a Visual Studio-produced UWP appxbundle.

## Thanks where it's due
It was only possible to create this tool because of SO users [draketb](https://stackoverflow.com/users/1751253/draketb) and [RbMm](https://stackoverflow.com/users/6401656/rbmm) (and others). So big thanks to them and the whole SO community!
