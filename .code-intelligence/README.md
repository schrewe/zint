# Fuzzing the Zint barcode encoder

## What is Zint?

Zint is a suite of programs to allow easy encoding of data in any of the wide
range of public domain barcode standards and to allow integration of this
capability into your own programs.

## What is fuzzing (in a nutshell)?

Fuzzing is a dynamic code analysis technique that supplies pseudo-random inputs
to a software-under-test (SUT), derives new inputs from the behaviour of the
program (i.e. how inputs are processed), and monitors the SUT for bugs.

As Zint and libzint is written mostly in C, we are particularly concerned with
memory corruption bugs such as heap or buffer overflows. These bugs can be
exploited to read or even write arbitrary data into the memory, resulting in
information leakage (think Heartbleed) or remote code execution.

## Fuzzing where raw data is handled

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

## A note regarding corpus data (and why there are more fuzz tests to explore)

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
