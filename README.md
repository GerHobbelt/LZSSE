# LZSSE-SIMDe

[LZSSE](https://github.com/ConorStokes/LZSSE/) has a hard dependency on SSE4.1 which prevents it from working on other architectures, or even x86/x86_64 machines without support for the SSE4.1 instruction set.  According to the [Steam Hardware Survey](http://store.steampowered.com/hwsurvey), SSE4.1 currently has just under 90% penetration, and of course that is only for machines with Steam installed (which is a pretty big bias).

This is a fork of [LZSSE](https://github.com/ConorStokes/LZSSE/) which uses [SIMDe](https://github.com/nemequ/simde) to allow for LZSSE (de)compression on platforms where SSE4.1 is not supported, including other architectures (such as ARM).

Note that, with the default block size from the example program, LZSSE-SIMDe will not work on 32-bit architectures due to memory requirements.  Reducing the block size resolves the issue, and the code has been tested on ARM and x86.  [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension) should also work, but has not been tested.

For machines with SSE4.1 support there should be no performance impact.  The SSE4.1 intrinsics will be called, and the compiler should be capable of optimizing away any overhead associated with SIMDe.

For machines which don't natively support the instructions used, SIMDe will emulate them using other SIMD APIs or, if that fails, portable fallbacks.

Note that a mix of the two is quite possible; for example, a CPU may support SSSE3 but not SSE4.1, in which case SSE4.1 functions will be emulated but SSSE3 and earlier instructions will used.

I'll try to keep this up to date with LZSSE, but I will not accept any changes directly to this repository not directly related to porting to SIMDe.  If you find a bug, please file it with LZSSE or SIMDe, whichever would be more appropriate.

## SIMDe Performance

This is based on some testing with g++ 10 using [raspbian-jessie-lite-20151121](https://github.com/nemequ/squash-corpus/blob/master/data/raspbian-jessie-lite-20151121.tar.xz).  Results are the average wall-clock time across 5 runs.

As you read this, please keep two things in mind:

First, this isn't likely to be the same for your code.  Performance will depend heavily on which functions you use, and what SIMDe's options for fallbacks are.  SIMDe is usually very easy to integrate into your project, so you should really run your own tests using your code and your data.

Second, SIMDe should never make your code slower, only more portable.  It doesn't really make sense to think of SIMDe as a performance hit since the alternative is that the code doesn't work at all; in that sense, SIMDe represents an infinite performance improvement.

That said, if you are currently maintaining a portable fallback and an SSE version, there is an excellent chance that SIMDe will be significantly faster than your portable fallback.

Now that that's out of the way, let's get to some data.

If provided the same compiler flags (in this case, `-msse4.1 -O3`), results for LZSSE and LZSSE-SIMDe are effectively the same.  So **SIMDe doesn't make things worse**, which is *very* important:

| Library     | Variant | Compress | Decompress |
| ----------- | ------- | -------- | ---------- |
| LZSSE       | LZSSE2  |  87.18 s |     0.55 s |
| LZSSE-SIMDe | LZSSE2  |  86.33 s |     0.55 s |
| LZSSE       | LZSSE4  |  73.83 s |     0.47 s |
| LZSSE-SIMDe |	LZSSE4  |  73.48 s |     0.47 s |
| LZSSE       | LZSSE8  |  79.16 s |     0.45 s |
| LZSSE-SIMDe |	LZSSE8  |  79.28 s |     0.44 s |

Things get a bit more interesting if we compile without SSE 4.1 support, forcing SIMDe to use portable implementations of the SSE 4.1 functions that LZSSE relies on:

| Flags    | Variant | Compress | Decompress |
| -------- | ------- | -------- | ---------- |
| -msse2   | LZSSE2  |  86.40 s |    13.70 s |
| -msse2   | LZSSE4  |  74.14 s |    10.78 s |
| -msse2   | LZSSE8  |  76.76 s |    10.07 s |
| -mssse3  | LZSSE2  |  86.47 s |     0.55 s |
| -mssse3  | LZSSE4  |  73.01 s |     0.48 s |
| -mssse3  | LZSSE8  |  78.81 s |     0.45 s |
| -msse4.1 | LZSSE2  |  86.33 s |     0.55 s |
| -msse4.1 | LZSSE4  |  73.48 s |     0.47 s |
| -msse4.1 | LZSSE8  |  79.28 s |     0.44 s |

Remember, there are no numbers here for upstream LZSSE since it simply doesn't work at all.  As you can see, moving from SSE2 to SSSE3 provides a *huge* performance increase for decompression; that's because SSSE3 supports the `_mm_shuffle_epi8` function, which we don't have a very good way to emulate on previous versions of SSE.  That makes it a great example of SIMDe performance can vary wildly depending on the functions you use and the platform you're targeting.  For what it's worth, AArch64 does have a good way to emulate it (`vqtbl1q_s8`), as does AltiVec (`vec_perm`), and ARMv7 has a decent option with a couple of `vtbl2_s8` calls.

In the worst case we can force SIMDe to always use the portable fallbacks and rely exclusively on the compiler to auto-vectorize by passing `-DSIMDE_NO_NATIVE`.  To be clear, you should never do this; it's really only there to help us test the fallbacks.  In this case, flags like `-msse4.1` only tell the compiler (GCC in this case) which extensions it is allowed to use; they are completely ignored by SIMDe.

| Flags    | Variant | Compress | Decompress |
| -------- | ------- | -------- | ---------- |
| -msse2   | LZSSE2  | 169.52 s |    13.72 s |
| -msse2   | LZSSE4  | 138.56 s |    10.82 s |
| -msse2   | LZSSE8  | 140.77 s |    10.08 s |
| -mssse3  | LZSSE2  | 169.48 s |    13.71 s |
| -mssse3  | LZSSE4  | 138.29 s |    10.80 s |
| -mssse3  | LZSSE8  | 140.71 s |    10.06 s |
| -msse4.1 | LZSSE2  | 169.41 s |     1.99 s |
| -msse4.1 | LZSSE4  | 138.28 s |     1.50 s |
| -msse4.1 | LZSSE8  | 140.02 s |     1.46 s |

Notice that this time there is not a significant change in compression speed with SSSE3.  That's because the compiler isn't smart enough to recognize that it should compile our portable implementation of `_mm_shuffle_epi8` to a `PSHUFB` instruction, which is a good example of why SIMDe generally significantly outperforms non-vectorized fallbacks even on architectures other than the one the original SIMD implementation is targeted at.  That said, with SSE 4.1 the compiler was able to use `PSHFLW`/`PSHFHW` (from SSE) with a blend from SSE4.1, recovering *most* of the performance.

It's tempting to think of this a bit like running the code on ARM, WASM, POWER, etc., but that's not accurate; on ARM SIMDe can use NEON to implement the SSE functions, and the code tends to be much faster.  Similarly, on WASM we can use WASM SIMD, and on POWER we can use AltiVec/VSX.

# LZSSE
[LZSS](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Storer%E2%80%93Szymanski) designed for a branchless SSE decompression implementation.

Three variants:
- LZSSE2, for high compression files with small literal runs.
- LZSSE4, for a more balanced mix of literals and matches.
- LZSSE8, for lower compression data with longer runs of matches.

All three variants have an optimal parser implementation, which uses a quite strong match finder (very similar to LzFind) combined with a Storer-Szymanski style parse. LZSSE4 and LZSSE8 have "fast" compressor implementations, which use a simple hash table based matching and a greedy parse.

Currently LZSSE8 is the recommended variant to use in the general case, as it generally performs well in most cases (and you have the option of both optimal parse and fast compression). LZSSE2 is recommended if you are only using text, especially heavily compressible text, but is slow/doesn't compress as well on less compressible data and binaries.

The code is approaching production readiness and LZSSE2 and LZSSE8 have received a reasonable amount of testing.

See these blog posts [An LZ Codec Designed for SSE Decompression](http://conorstokes.github.io/compression/2016/02/15/an-LZ-codec-designed-for-SSE-decompression) and [Compressor Improvements and LZSSE2 vs LZSSE8](http://conorstokes.github.io/compression/2016/02/24/compressor-improvements-and-lzsse2-vs-lzsse8) for a description of how the compression algorithm and implementation function. There are also benchmarks, but these may not be upto date (in particular the figures in the initial blog post no longer represent compression performance).
