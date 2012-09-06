# keychaindump
Keychaindump is a proof-of-concept tool for reading OS X keychain passwords as root. It hunts for unlocked keychain master keys located in the memory space of the securityd process, and uses them to decrypt keychain files.

See the [blog post](http://juusosalonen.com/post/30923743427/breaking-into-the-os-x-keychain) for a much more readable description.

## How?
Build instructions:

    $ gcc keychaindump.c -o keychaindump -lcrypto

Basic usage:

    $ sudo ./keychaindump [path to keychain file, leave blank for default]

Example with truncated and censored output:

    $ sudo ./keychaindump 
    [*] Searching process 15 heap range 0x7fa809400000-0x7fa809500000
    [*] Searching process 15 heap range 0x7fa809500000-0x7fa809600000
    [*] Searching process 15 heap range 0x7fa809600000-0x7fa809700000
    [*] Searching process 15 heap range 0x7fa80a900000-0x7fa80ac00000
    [*] Found 17 master key candidates
    [*] Trying to decrypt wrapping key in /Users/juusosalonen/Library/Keychains/login.keychain
    [*] Trying master key candidate: b49ad51a672bd4be55a4eb4efdb90b242a5f262ba80a95df
    [*] Trying master key candidate: 22b8aa80fa0700605f53994940fcfe9acc44eb1f4587f1ac
    [*] Trying master key candidate: 1d7aa80fa0700f002005043210074b877579996d09b70000
    [*] Trying master key candidate: 88edbaf22819a8eeb8e9b75120c0775de8a4d7da842d4a4a
    [+] Found master key: 88edbaf22819a8eeb8e9b75120c0775de8a4d7da842d4a4a
    [+] Found wrapping key: e9acc39947f1996df940fceb1f458ac74b877579f54409b7
    xxxxxxx:192.168.1.1:xxxxxxx
    xxxxxxx@gmail.com:login.facebook.com:xxxxxxx
    xxxxxxx@gmail.com:smtp.google.com:xxxxxxx
    xxxxxxx@gmail.com:imap.google.com:xxxxxxx
    xxxxxxx:twitter.com:xxxxxxx
    xxxxxxx@gmail.com:www.google.com:xxxxxxx
    xxxxxxx:imap.gmail.com:xxxxxxx
    ...

## Who?
Keychaindump was written by [Juuso Salonen](http://twitter.com/juusosalonen), the guy behind [Radio Silence](http://radiosilenceapp.com) and [Private Eye](http://radiosilenceapp.com/private-eye).

## License
Do whatever you wish. Please don't be evil.

## Blog post contents

The attack

The passwords in a keychain file are encrypted many times over with various different keys. Some of these keys are encrypted using other keys stored in the same file, in a russian-doll fashion. The key that can open the outermost doll and kickstart the whole decryption cascade is derived from the user’s login password using PBKDF2. I’ll call this key the master key.

If anyone wants to read the contents of a keychain file, they have to know either the user’s login password or the master key. This means that securityd, the process that handles keychain operations in OS X, has to know at least one of these secrets.

I put this observation to the test on my own laptop.

Scanning securityd’s whole memory space did not reveal any copies of my login password. Next, I used PBKDF2 with my login password to get my 24-byte master key. Scanning the memory again, a perfect copy of the master key was found in securityd’s heap.

This lead to the next question. If the master keys are indeed stored in securityd’s memory, is there a good way to find them? Testing every possible 24-byte sequence of the memory space is not very elegant.

Instead of fully inspecting securityd’s whole memory space, possible master keys can be pinpointed with a couple of identifying features. They are stored in securityd’s heap in an area flagged as MALLOC_TINY by vmmap. In the same area of memory, there’s also always structure pointing to the master key. The structure contains an 8-byte size field with the value of 0x18 (24 in hex) and a pointer to the actual data.

The search is rather simple:

    Find securityd’s MALLOC_TINY heap areas with vmmap

    Search each found area for occurrences of 0x0000000000000018

    If the next 8-byte value is a pointer to the current heap area, treat the pointed-to data as a possible master key

With this kind of pattern recognition, the number of possible master keys is reduced to about 20. Each of the candidates can be used to try to decrypt the next key (which I call the wrapping key). Only the real master key should spit out a sensible value for the wrapping key. It’s not a foolproof method, but with the low number of candidates it seems to be good enough. I haven’t run into a single false positive yet.

The rest of the decryption process is rather tedious. The master key reveals the wrapping key. A hardcoded obfuscation key reveals the encrypted credential key. The wrapping key reveals the credential key. The credential key finally reveals the plaintext password. All glory to [Matt Johnston](http://matt.ucc.asn.au/) for his research on these decryption steps.