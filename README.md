
# AntiFuzz: Impeding Fuzzing Audits of Binary Executables

Get the paper here: https://www.usenix.org/system/files/sec19fall_guler_prepub.pdf

## Usage:
antifuzz_generate.py generates a "antifuzz.h" file that you need to include in your project. The script takes multiple arguments to define which features you need.

To disable all features, supply:

      --disable-all

  
To break assumption (A), i.e. to break coverage-guided fuzzers, use:

      --enable-anti-coverage

You can specify how many random BBs you want to have by supplying --anti-coverage [num], default is 10000.

To break assumption (B), i.e. to prevent fuzzers from detecting crashes, use:

      --signal --crash-action exit

To break assumption (C), i.e. to decrease the performance of the application when being fuzzed, use:

      --enable-sleep --signal

Additionaly, you can supply --crash-action timeout to replace every crash with a timeout.

To break assumption (D), i.e. to boggle down symbolic execution engines, use:

      --hash-cmp --enable-encrypt-decrypt

To enable all features, use:

      --enable-anti-coverage --signal --crash-action exit --enable-sleep --signal --hash-cmp --enable-encrypt-decrypt

## Demo
To test it out, we supplied a demo application called antifuzz_test.c that just checks for "crsh" with single byte comparisons, and crashes if that's the case. It configures itself to fit the generated antifuzz header file, i.e. wenn hash comparisons are demanded via antifuzz_generate.py, antifuzz_demo will compare the hashes instead of the plain constants.

First, generate the antifuzz.h file:

    python antifuzz_generate.py --enable-anti-coverage --signal --crash-action exit --enable-sleep --signal --hash-cmp --enable-encrypt-decrypt

Next, compile the demo application (note that this may take minutes (!) depending on the number of random BBs added):

    afl-gcc antifuzz_test.c -o antifuzz_test # this can take minutes

Run it in AFL to test it out:

    mkdir inp; echo 1234 > inp/a.txt; afl-fuzz -i inp/ -o /dev/shm/out -- ./antifuzz_test @@

If you enabled all options, AFL may take a long time to start because the application is slowed down (to break assumption (C))

## Protecting Application
To include it in your own C project, follow these instructions (depending on your use-case and application, you might want to skip some of them):
1. Add 

    #include "antifuzz.h"
    
    to the header.
2. Jump to the line that opens the (main) input file, the one that an attacker might target as an attack vector, and call 
  antifuzz_init("file_name_here", FLAG_ALL); 
  This will initialize AntiFuzz, check if overwriting signals is possible and if ptrace is active, put the input through encryption and decryption, jump through random BBs, etc.
3. Find all lines that deal with malformed input files, most often these already contain some kind of error or warning message (e.h. "this is not a valid ... file"). Add a call to "antifuzz_onerror()" everywhere you deem appropriate.
4. Find comparisons to constants (e.g. magic bytes) that you think are important for this file format, and change the comparison to hash comparisons. Add your constant to antifuzz_constants.tpl.h like this:
char \*antifuzzELF = "ELF";
Our generator script will automatically change these lines to their respective SHA512 hashes, you do not have to do this manually.
Now change the lines from (as an example):
if(strcmp(header, "ELF") == 0)
to
if(antifuzz_str_equal(header, antifuzzELF))
See antifuzz.tpl.h for more comparison functions.
5. If you have more data that you want to protect from symbolic execution, use antifuzz_encrypt_decrypt_buf(char \*ptr, size_t fileSize) which does an in-memory encryption and decryption of your data.