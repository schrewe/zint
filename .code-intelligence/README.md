# Fuzzing the Zint barcode encoder

## What is Zint?

Zint is a suite of programs to allow easy encoding of data in any of the wide
range of public domain barcode standards and to allow integration of this
capability into your own programs.

For more info on Zint click [here](https://zint.org.uk/).

## The problem

As Zint is written in C and handles arbitrary input in its decoding functions,
care must be taken that the program only reads and writes to and from previously
allocated memory regions (i.e., avoiding buffer overflows). These bugs have been
notoriously hard to hunt down completely, especially in an automated fashion.

## The solution

By sending unexpected inputs to the decoder, fuzzing can trigger erroneous behaviour.
In case of C/C++ applications, CI Fuzz detects and reports memory corruptions, which can
originate from programming mistakes in code that employs pointer arithmetics or
low level memory operations (such as `memcpy`, `strcpy`, ...). These bugs can be
exploited to read or even write arbitrary data into the memory, resulting in
information leakage (think Heartbleed) or remote code execution.

Fuzzing is a dynamic code analysis technique that supplies pseudo-random inputs
to a software-under-test (SUT), derives new inputs from the behaviour of the
program (i.e. how inputs are processed), and monitors the SUT for bugs.


## The setup

### Fuzzing where raw data is handled

Fuzzing is most efficient where raw data is parsed, because in this case no
assumptions can be made about the format of the input. Zint allows you to pass
arbitrary data to an encode function (called `ZBarcode_Encode`) and populates a
`zint_symbol` from that.

The most universal example of this type of fuzz test can be found in
[`.code-intelligence/fuzz_targets/all_barcodes_fuzzer.cpp`](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/all_barcodes_fuzzer.cpp).
Let me walk you through the heart of the fuzz test:

```C++
// 1. The fuzzer calls the FUZZ macro with pseudo-random data and size.
extern "C" int FUZZ(const unsigned char *Data, size_t Size)
{
  if (Size < 4 || Size > 1000)
    return 0;

  // 2. The FuzzedDataProvider is a convenience-wrapper around Data and Size
  //    and offers methods to get portions of the data casted into different
  //    types.
  FuzzedDataProvider dp(Data, Size);

  struct zint_symbol *my_symbol = ZBarcode_Create();

  // 3. The FuzzedDataProvider is used to select a specific barcode type to
  //    encode into.
  my_symbol->symbology = dp.PickValueInArray(BARCODES);

  // 4. The remaining bytes of the fuzzer input are fed into ZBarcode_encode(),
  //    calling the function to be tested.
  auto remaining = dp.ConsumeRemainingBytes<unsigned char>();
  ZBarcode_Encode(my_symbol, remaining.data(), remaining.size());

  // 5. Finally, cleaning up happens.
  ZBarcode_Delete(my_symbol);

  return 0;
}
```

If you haven't done already, you can now explore what the fuzzer found when
running this fuzz test.

### A note regarding corpus data (and why there are more fuzz tests to explore)

For each fuzz test that we write, a corpus of interesting inputs is built up.
Over time, the fuzzer will add more and more inputs to this corpus, based
coverage metrics such as newly-covered lines, statements or even values in an
expression.

The rule of thumb for a good fuzz test is that the format of the inputs should
be roughly the same. Therefore, it is sensible to split up the big fuzz test for
all barcode types into individual fuzz tests. You can see how this is done in
practice in the following individual fuzz tests (all in
`.code-intelligence/fuzz_targets`):

-   [auspost_fuzzer.cpp](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/auspost_fuzzer.cpp)
-   [codablockf_fuzzer.cpp](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/codablockf_fuzzer.cpp)
-   [codeone_fuzzer.cpp](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/codablockf_fuzzer.cpp)
-   [dotcode_fuzzer.cpp](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/codablockf_fuzzer.cpp)
-   [eanfuzzer_fuzzer.cpp](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/codablockf_fuzzer.cpp)
-   [vin_fuzzer.cpp](https://github.com/ci-fuzz/zint/blob/master/.code-intelligence/fuzz_targets/codablockf_fuzzer.cpp)

### Fuzzing in CI/CD

CI Fuzz allows you to configure your pipeline to automatically trigger the run of fuzz tests.
Most of the fuzzing runs that you can inspect here were triggered automatically (e.g. by pull or merge request on the GitHub project).
As you can see in this [`pull request`](https://github.com/ci-fuzz/zint/pull/53) the fuzzing results are automatically commented by the github-action and developers
can consume the results by clicking on "View Finding" which will lead them directly to the bug description with all the details
that CI Fuzz provides (input that caused the bug, stack trace, bug location).
With this configuration comes the hidden strength of fuzzing into play:  
Fuzzing is not like a penetration test where your application will be tested one time only.
Once you have configured your fuzz test it can help you for the whole rest of your developing cycle.
By running your fuzz test each time when some changes where made to the source code you can quickly check for
regressions and also quickly identify new introduced bugs that would otherwise turn up possibly months 
later during a penetration test or (even worse) in production. This can help to significantly reduce the bug ramp down phase of any project.

While these demo projects are configured to trigger fuzzing runs on merge or pull requests
there are many other configuration options for integrating fuzz testing into your CI/CD pipeline
for example you could also configure your CI/CD to run nightly fuzz tests.
