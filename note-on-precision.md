# Notes on the precision

In these tests, `v00`, `v01`, `v10` and `v11` are single-precision floating point numbers.
This is explicit in Java and C/C++ code, but implicit and hard to see in the Python code.
However, the effect is visible in the following calculation:

```python
v0 = (v01 - v00) * xf + v00
v1 = (v11 - v10) * xf + v10
```

If the values are not cast to `double`, then `(v01 - v00)` and `(v11 - v10)` are computed with single precision.
The precision lost causes a faster drift from the expected results. In order to force a cast to double-precision,
the Java and C/C++ code use the following:

```java
double v0 = (v01 - (double) v00) * xf + v00;
double v1 = (v11 - (double) v10) * xf + v10;
```

In Python, adding 0.0 is a trick that appears to work:

```python
v0 = (v01 - (v00 + 0.0)) * xf + v00
v1 = (v11 - (v10 + 0.0)) * xf + v10
```

The differences are shown below.
Note that this issue is unrelated to NaN.


## With cast to double-precision and use of FMA
Below is the output of the Java and C/C++ tests
that use `Math.fma(…)` and `std::fma`, respectively.

```
Errors in the use of raster data with NaN values in big endian byte order:
   Count     Minimum     Average     Maximum   Number of "missing value" mismatches
   11732      0,0000      0,0000      0,0000      0
   10771      0,0000      0,0000      0,0000      0
   10809      0,0000      0,0000      0,0000      0
   10762      0,0000      0,0000      0,0000      0
   10851      0,0000      0,0000      0,0000      0
   10814      0,0000      0,0000      0,0016      0
   10826      0,0000      0,0000      0,0871      0
   10918      0,0000      0,0042     18,1068      0
   10734      0,0000      0,0208     65,3695      9
   10792      0,0000      0,1692    127,6496     35
```

## With cast to double-precision but without FMA
Below is the output of the Python test.
It is also the output of the Java and C/C++ tests when FMA is replaced by a
multiplication followed by an addition. We observe a small loss of accuracy.
Note that on the machine used for the test (Intel® Core™ i7-8750H) and the
`gcc` compiler used (14.2.1 20240912 (Red Hat 14.2.1-3)), we get this output
even with the `-ffast-math` option.

```
Errors in the use of raster data with NaN values in big endian byte order:
   Count     Minimum     Average     Maximum   Number of "missing value" mismatches
   11732      0,0000      0,0000      0,0000      0
   10771      0,0000      0,0000      0,0000      0
   10809      0,0000      0,0000      0,0000      0
   10762      0,0000      0,0000      0,0000      0
   10851      0,0000      0,0000      0,0000      0
   10814      0,0000      0,0000      0,0009      0
   10826      0,0000      0,0001      0,0871      0
   10918      0,0000      0,0047     18,1068      0
   10735      0,0000      0,0383     94,2899     10
   10789      0,0000      0,2025    127,6496     42
```

## Without cast and without FMA
Below is the output that we get if the `(v01 - v00)` and `(v11 - v10)`
are computed without casting at least one `float` value to `double`
before the subtraction. In Python, the cast is forced by `+ 0.0`.

```
Errors in the use of raster data with NaN values in big endian byte order:
   Count     Minimum     Average     Maximum   Number of "missing value" mismatches
   11732      0,0000      0,0000      0,0000      0
   10771      0,0000      0,0000      0,0011      0
   10809      0,0000      0,0006      0,1469      0
   10760      0,0000      0,0264     12,2387      6
   10814      0,0000      0,3692    130,8832     90
   10670      0,0000      1,1468    138,7692    379
   10522      0,0000      2,2290    144,7353    799
   10379      0,0000      3,5549    158,2099   1383
    9938      0,0000      5,3378    161,3066   2001
    9783      0,0000      7,2493    171,4665   2595
```
