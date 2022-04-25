# What is this

A feature-complete decoder for the zstd compression format as defined in: [This document](https://github.com/facebook/zstd/blob/dev/doc/zstd_compression_format.md).

It is NOT a compressor. I don't plan on implementing that part either, at least not in the near future. (If someone is motivated enough I will of course accept a pull-request!)

This crate might look like it is not active, this is because there isn't really anything to do anymore, unless a bug is found or a new API feature is requested. I will of course respond to and look into issues!

# Current Status

[![Actions Status](https://github.com/KillingSpark/zstd-rs/workflows/CI/badge.svg)](https://github.com/KillingSpark/zstd-rs/actions?query=workflow%3A"CI")

This is started just as a toy project but I think it is in a usable state now. It does work correctly at least for the test-set of files I used, YMMV, it is not yet battle tested by any means.
For production use (or if you need a compressor) I would (at the time of writing, this might get out of date and there might come better projects along!) recommend to use the C binding located [here](https://github.com/gyscos/zstd-rs).

If you'd be willing to try this in your projects I would be very happy though! The crate is published to [crates.io](https://crates.io/crates/ruzstd) following the releases on this repo. The docs are located
[here](https://docs.rs/ruzstd) (These might lag behind the releases since docs.rs doesn't pull from crates.io immediately but they will show the latest version eventually).

## Speed

Measuring with the 'time' utility the original zstd and my decoder both decoding the same enwik9.zst file from aramfs, my decoder is about 3.5 times slower. Enwik9 is highly compressible, for less compressible data (like a ubuntu installation .iso) my decoder comes close to only being 1.4 times slower.

## Can do:

1. Parse all files in /decodecorpus_files. These were generated with [decodecorpus](https://github.com/facebook/zstd/tree/dev/tests) by the original zstd developers
1. Decode all of them correctly into the output buffer
1. Decode all the decode_corpus files (1000+) I created locally
1. Calculate checksums

## Cannot do

This decoder is pretty much feature complete but probably not bug free. If there are any wishes for new APIs or bug reports please file an issue, I will gladly take a look!

## Roadmap

1. Test/fuzz dictionary implementation
1. More Performance optimizations (targets would be sequence_decoding and reverse_bitreader::get_bits. Those account for about 50% of the whole time used)
1. More tests (especially unit-tests for the bitreaders and other lower-level parts)
1. Find more bugs

## Testing

Tests take two forms.

1. Tests using well-formed files that have to decode correctly and are checked against their originals
1. Tests using malformed input that have been generated by the fuzzer. These don't have to decode (they are garbage) but they must not make the decoder panic

## Fuzzing

Fuzzing has been done with cargo fuzz. Each time it crashes the decoder I fixed the issue and added the offending input as a test. It's checked into the repo in the fuzz/artifacts/fuzz_target_1 directory. Those get tested in the fuzz_regressions.rs test.
At the time of writing the fuzzer was able to run for over 12 hours on the random input without finding new crashes. Obviously this doesn't mean there are no bugs but the common ones are probably fixed.

Fuzzing has been done on

1. Random input with no initial corpus
2. The \*.zst in /fuzz_decodecorpus

### You wanna help fuzz?

Use `cargo +nightly fuzz run decode` to run the fuzzer. It is seeded with files created with decodecorpus.

If (when) the fuzzer finds a crash it will be saved to the artifacts dir by the fuzzer. Run `cargo test artifacts` to run the artifacts tests.
This will tell you where the decoder panics exactly. If you are able to fix the issue please feel free to do a pull request. If not please still submit the offending input and I will see how to fix it myself.

# How can you use it?

## Easy

The easiest is to wrap the io::Read into a StreamingDecoder which itself implements io::Read. It will decode blocks as necessary to fulfill the read requests

```
let mut f = File::open(path).unwrap();
let mut decoder = StreamingDecoder::new(&mut f);

let mut result = Vec::new();
decoder.read_to_end(&mut buffer).unwrap();
```

This might be a problem if you are accepting user provided data. Frames can be REALLY big when decoded. If this is the case you should either check how big the frame
actually is or use the memory efficient approach described below.

## Memory efficient

If memory is a concern you can decode frames partially. There are two ways to do this:

#### Streaming decoder

Use the StreamingDecoder and use a while loop to fill your buffer (see src/bin/zstd_stream.rs for an example). This is the
recommended approach.

#### Use the lower level FrameDecoder

For an example see the src/bin/zstd.rs file. Basically you can decode the frame until either a
given block count has been decoded or the decodebuffer has reached a certain size. Then you can collect no longer needed bytes from the buffer and do something with them, discard them and resume decoding the frame in a loop until the frame has been decoded completely.

# What you might notice

I already have done a decoder for zstd in golang. [here](https://github.com/KillingSpark/sparkzstd). This was a first try and it turned out very inperformant. I could have tried to rewrite it to use less allocations while decoding etc etc but that seemed dull (and unnecessary since klauspost has done a way better golang implementation that additionally can compress data [here](https://github.com/klauspost/compress/tree/master/zstd))

## Why another one

Well I wanted to show myself that I am actually able to learn from my mistakes. This implementation should be way more performant since I from the get go focussed on reusing allocated space instead of reallocating all the decoding tables etc.

Also this time I did most of the work without looking at the original source nor the [educational decoder](https://github.com/facebook/zstd/tree/dev/doc/educational_decoder).
So it is somewhat uninfluenced by those. (But I carried over some memories from the golang implementation).
I used it to understand the huffman-decoding process and the process of how to exactly distribute baseline/num_bits in the fse tables on which the documentation is somewhat ambiguous.
After having written most of the code I used my golang implementation for debugging purposes (known 'good' results of the different steps).

## Known bugs:

currently none
