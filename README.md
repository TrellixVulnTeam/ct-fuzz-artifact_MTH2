## Testing Environment
We tested the artifact on a machine with a 4GHz 6770k with 32 GB of RAM.
The VM we used to test the artifact is based on Ubuntu 18.04 and we allocate 2GB memory to the VM.

## Instructions to Build ct-fuzz:

0.1. Login as root: sudo su -

0.2. Execute the command: echo core >/proc/sys/kernel/core_pattern

0.3. Logout root

1. Copy the zip file to the home directory of the VM.

2. Unzip it.

3. Go to `artifact` directory and execute `sh install.sh`. You may be asked to
type in the password of root access.

4. Go to `ct-fuzz/test` directory and execute `lit .`. You should expect to see
all regressions passed.

## Instructions to Run ct-fuzz:

We use the example in the paper to demonstrate the basic usage of ct-fuzz.
The example is `ex/encrypt.c`.

1. Go to `ex` directory.

2. Run `ct-fuzz encrypt.c --entry-point=encrypt -o encrypt --compiler-options="-g"`.
A seed is printed as `\x31\x32\x33\x34\x04\x00\x00\x00`.

3. Make an input directory to afl-fuzz: `mkdir in`

4. Duplicate the seed and save it as a binary file in directory `in`:
```
echo -n -e '\x31\x32\x33\x34\x04\x00\x00\x00\x31\x32\x33\x34\x04\x00\x00\x00' > in/test
```

5. Run afl-fuzz: `afl-fuzz -i in -o out ./encrypt`. It should report crashes immediately.
Type "Ctrl+C" to exit afl-fuzz.

6. Invoking the binary with DEBUG variable setting to `true` will dump the error trace:
`cat out/crashes/id\:000000\,sig\:42\,src\:000000\,op\:flip1\,pos\:0 | DEBUG=true ./encrypt`

6.1 You can pipe the error trace to a script provided by ct-fuzz to show the diffed trace:
`cat out/crashes/id\:000000\,sig\:42\,src\:000000\,op\:flip1\,pos\:0 | DEBUG=true ./encrypt | python ~/artifact/ct-fuzz/scripts/ct-fuzz-diff.py`
The last line is where the divergence occurs.

7 To check cache timing leakage, invoke `ct-fuzz` again,
```
ct-fuzz encrypt.c --entry-point=encrypt -o encrypt --compiler-options="-g" --memory-leakage=cache
```

8. Repeat 3-6. You should not see any crashes. Exit afl-fuzz in one minute.

## Instructions to Reproduce Results:

0. Go to directory `ct-benchmarks` and type `make`. It takes a while to build all benchmarks.

### To reproduce the case study:
1. Go to directory `botan/build`
2. Create input directory for afl-fuzz following similar instructions above:
```
$ mkdir in
$ cat aes_wrapper.seed # should print \x2A\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00 
$ echo -n -e '\x2A\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x2A\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00' > in/test
```
3. Run afl-fuzz:
```
AFL_NO_FORKSRV=1 afl-fuzz -m 500 -i in -o out ./aes_wrapper
```
It should report a crash soon. Exit it by "Ctrl+C".

4. Follow the same instruction to see the error trace.
We can see the content of the triggering input: `hexdump out/crashes/id\:000000\,sig\:42\,src\:000000\,op\:flip1\,pos\:0`.
The first byte is different.

5. Switch to cache model:
```
cd .. # assume we're still in `botan/build` directory
ct-fuzz --entry-point aes_wrapper -o build/aes_wrapper specs/aes_wrapper.cpp \
--seed-file build/aes_wrapper.seed \
--compiler-options="-I$HOME/artifact/ct-benchmarks/botan/src/build/include \
-L$HOME/artifact/ct-benchmarks/botan/src/ \
-Wl,-rpath,$HOME/artifact/ct-benchmarks/botan/src/ \
-g -std=c++11 \
-lbotan-2" --memory-leakage=cache
cd build
```

6. Rerun afl-fuzz with the same command. There should not be any crashes within reasonably long time.
You can exit afl-fuzz in one minute.

7. To see the bug before the fix (the current commit), execute the following commands,
```
cd ../src # assume we're still in `botan/build` directory
git checkout HEAD^
rm libbotan-2.so*
cd ..
make
cd build
```
Repeat 3-5 and you should be able to see AFL crashed even when the leakage model is cache.


### To reproduce evaluation results:
0. We provide a script in the artifact directory, `run.sh` that runs all benchmarks once for both constant-time analysis and more precise analysis with cache models. Simply invoke it: `sh run.sh <time-limit>` in the artifact directory. We suggest first set the time limit to 10. Setting it to 100 may take a while. If you are interested in more detailed exploration of these benchmarks, please follow the instructions below.

0.1 We also provide a script in the artifact directory `reproduce.sh` that tries to reproduce the evaluations done in the paper. You can invoke it by `sh reproduce.sh` in the artifact directory. Note that there will be differences in average runtime, executions done, total paths. But there should not be any differences in terms of whether a crash is found or not (i.e., unique_crashes > 0 or not). You can also see the trends like runtime of certain benchmarks are higher or cache models have less exeuctions done, etc.
*Warning: this script can take hours to run*

1. Go back to directory `ct-benchmarks`.
We have 8 top level `build` directories that contain binaries built by ct-fuzz. They are,
```
./wu-issta-2018/build
./BearSSL/build
./botan/build
./libsodium/build
./openssl/build
./poly1305-donna/build
./poly1305-opt/build
./s2n/build
```

2. We provide a script (`script/run.py`) to run a binary or a directory that contains binaries.
For example, in `wu-issta-2018/build/supercop`, we can run all the benchmarks in this directory,
```
python ~/artifact/ct-benchmarks/scripts/run.py . --time-limit=10
```
Or run one benchmark,
```
python ~/artifact/ct-benchmarks/scripts/run.py ./aes_core --time-limit=10
```

3. To obtain data similar to the evaluation results, we provide a script to analyze the output of `run.py`.
For example, in `BearSSL/build`, we can run all the binaries in this directory twice with time limit 10s.
```
for i in `seq 1 2`; do python ~/artifact/ct-benchmarks/scripts/run.py . --time-limit=10 --exit-on-crash; done | python ~/artifact/ct-benchmarks/scripts/analyze-result.py
```
It will print statistics of running each benchmarks.
Or we can run the same command on some binary,
```
for i in `seq 1 2`; do python ~/artifact/ct-benchmarks/scripts/run.py ./aes_small_wrapper --time-limit=10 --exit-on-crash; done | python ~/artifact/ct-benchmarks/scripts/analyze-result.py
```
Note that for some benchmarks, there will not be any violations. For these benchmarks, you may want to choose a small iteration number if the time limit is large (e.g., 100s).

4. Switch to cache model: first remove all the top level `build` directories; then checkout `cm` branch: `git checkout cm`; finally `make`

5. Now all the binaries are rebuilt with cache models. You can repeat 2 and 3 to test them.
