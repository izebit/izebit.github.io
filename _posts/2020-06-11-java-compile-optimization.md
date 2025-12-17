---
layout: post
tags: 
  - jvm
title: Not evident JIT optimizations
---

I would like to write about an interesting thing that I have found at my work.

The code is given:

```java
public String pack(final byte[] data) {
        int bufferIdx = 0;
        char[] buffer;
        if (data.length % 2 != 0) {
            buffer = new char[(data.length >> 1) + 2];
            buffer[0] = flag;
            bufferIdx = 1;
        } else {
            buffer = new char[data.length >> 1];
        }
        int idx = 0;
        for (int i = bufferIdx; i < buffer.length; i++) {
            int bpos = idx << 1;
            char c;
            if (bpos + 1 < data.length) {
                c = (char) (((data[bpos] &amp; 0x00FF) << 8) + (data[bpos + 1] &amp; 0x00FF));
            } else {
                c = (char) (((data[bpos] &amp; 0x00FF) << 8));
            }
            buffer[i] = c;
            idx++;
        }
        return new String(buffer);
}
```

As you can see, the method deserializes byte array into string.
The main point here is to convert 2 bytes to 1 char since the size of char in java is 2 bytes. 

I have faced that execution of the method is significantly slower on byte arrays with odd length than 
on byte arrays with not odd length.

It is clear that some job in the loop takes the most time of execution.
However, discovering the bytecode is pointless because the compiler does not have 
a clue what length of a byte array passed as a parameter.

According to this fact, there are some optimizations can be applied only in runtime by JIT compiler.
To confirm my guess I used the tool <a href="https://github.com/AdoptOpenJDK/jitwatch" rel="nofollow">__JitWatcher__</a>
The program shows generated assembler code after each stage of optimization passed in runtime.


Before going ahead, I have to give more details about JitWatcher works. 
It sets some specific jvm flags for your program:
```shell
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation -XX:+TraceClassLoading -XX:+PrintAssembly
```
and collects all information that jvm prints while your application is running. Hence, there is no magic and
you can manage without it but without _JitWatcher_ you have to analyze a bunch of assembler code. I bet you would not want to do it 😏.
_JitWatcher_ does it instead of you.  

To set the flag `-XX:+PrintAssembly` jvm requires _hsdis_ library. It has to be located in a directory `$JAVA_HOME/jre/lib/server/`
There are to ways to get the library. First of them is to build it on your own and the second one is to download already compiled <a href="https://github.com/a10y/hsdis-macos" rel="nofollow">artifact</a>.

After spending some time on analysing, I found out that _JIT_ makes some strange speculations. 
It is about <a href="https://www.felixcloutier.com/x86/MOVD:MOVQ.html" rel="nofollow">vector instructions</a>. In case with odd length of byte array
they are used. However, in case with not odd length they are not used by JIT.

It is well known, usage of the vector instructions can boost performance of your application significantly.
I decided to rewrite the loop in order to make JIT use them in all cases. This technique is called 
[__loop unrolling__](https://izebit.ru/cpu-optimizations.html)

```java
int countOfIteration = data.length / 2;
for (int index = 0; index < countOfIteration; index++) {
      final int position = index * 2;
      char firstPart = (char) ((data[position] &amp; 0x00FF) << 8);
      char secondPart = (char) (data[position + 1] &amp; 0x00FF);
      buffer[index] = (char) (firstPart + secondPart);
}
```

After making the changes, JIT does not have any chances. 
It started generate assembler code with vector instructions all the time 
regardless of a length of a byte array.

The table below shows the difference.

| Benchmark     | Size of array | Mode | Score |  Units   |
| ------------- |:-------------:|:----:|:-----:|:------:|
|with optimization|    100000     |  avgt  | 46777.0331| ms/op|
|with optimization|    100001     |  avgt  | 47718.458| ms/op|
|without optimization|    100000     |  avgt  | 47218.527| ms/op|
|without optimization|    100001     |  avgt  | 71312.515| ms/op|

