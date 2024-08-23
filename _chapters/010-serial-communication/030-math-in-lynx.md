---
title: Math in Lynx
abstract: Math capabilities of the Atari Lynx explained.
usemathjax: true
---
# Math capabilities in Mikey
The Atari Lynx has Mikey, a customized 65SC02 (65C02 in Lynx II) processor. The 6502, 65C02 and 65SC02 processors can perform simple math in the form of single byte add and subtract operations. How you interpret the individual bits in a byte allows you to see this as either signed or unsigned numbers representations. 
The processors have a `N` flag in the status register to indicate that a number has become negative by an operation. A number is negative when the 7th (leftmost and most significant) bit of a byte is set, in line with the 'twos complement' notation for negative numbers. Whenever the 7th bit of a changed byte is set as the result of an operation, the `N` flag is also set. 
Similarly, the carry `C` flag is set when additions carry over or subtractions do not borrow, as well as when comparing two numbers. This is about as far as math in the CPU goes and it is well documented. Your code can leverage this add, subtract and comparison functionality with the native 65X02 CPU instruction set of Mikey.

# Extra math capabilities in Suzy
The Atari Lynx also has hardware support for performing multiplications and divisions. It is capable of multiplying two 16-bit numbers for a 32-bit result and can optionally accumulate the result over multiple calculcations. The divisions take a 32-bit number divided by 16-bit to give a 16-bit result with a 16-bit remainder. It can also detect and indicate a divide by zero error. Since the math engine is implemented in hardware, it is much faster in execution than doing this in assembler (or C for that matter).

All math capabilities are implemented in the Suzy chip. Suzy offers registers via hardware addresses for the two math operations. Some of these are for reading or writing (or both) depending on the type of operation.

There are some specifics and peculiarities we need to look into to understand and use the math operations in Suzy correctly. Everything is discussed as we cover each of the math capabilities.

## Suzy registers 
Suzy has a total of 14 value registers with specific purposes. Additionally, it has a status register `SPRSYS` at `$FC92` for setting math behavior and reading details, such as errors.

You can see the registers and addresses in the table below. Note that the address space is not contiguous and has some unused address ranges.

|Register|Address|Register|Address|Register|Address|Register|Address|
|:---|:---|:---|:---|:---|:---|:---|:---|
|D|`$FC52`|H|`$FC60`|M|`$FC6C`|P|`$FC56`|
|C|`$FC53`|G|`$FC61`|L|`$FC6D`|N|`$FC57`|
|B|`$FC54`|F|`$FC62`|K|`$FC6E`|SPRSYS|`$FC92`|
|A|`$FC55`|E|`$FC63`|J|`$FC6F`|||

The addresses form pairs of two or four bytes depending on the mathematical operation you want to perform. The combinations are `ABCD`, `EFGH`, `NP`, `JKLM` as can be seen from the grouping in the address list above. Suzy also uses `AB` and `CD` in case of multiplication. It might seem strange that the order of the individual bytes in a group is reversed when looking from low to high addresses. We will see how this does make sense in a moment.

The behavior of the individual addresses deserves some attention. You will write 2-byte (16-bit) and 4-byte values (32-bit) to the addresses in a specific order and byte by byte. When writing to the addresses of `B`, `D`, `F`, `H`, `K`, `M` or `P`, a `$00` value will automatically be set in the nearest byte with a higher address. That will be `A`, `C`, `G`, `E`, `L`, `J` and `N` respectively. This is useful in a couple of situations as you will also see later.

> The original Epyx documentation does not mention the `NP` pair, but it does show the same behavior of clearing `N` to $00 when writing to `P`. 

## Multiplications in Suzy
In mathematical form multiplications are defined as follows:

$multiplicand \times multiplier = product$

We will refer to these mathematical names for the various parts of a multiplication.

Suzy can perform both signed and unsigned multiplications of integer numbers. Signed numbers can be both positive and negative, as they can have a negative sign. On the other hand, unsigned numbers never have a negative sign and will therefore always be positive. 
If your numbers have a possibility to be negative, you must choose signed multiplications. If not, your choice doesn't really matter except for one difference between signed and unsigned numbers: the range of values is not the same. 

The table shows the various minimum and maximum values of single byte (8-bit), double byte (16-bit) and quad byte (32-bit) numbers. These are number types that exist in higher level languages, such as C in cc65 or newcc65.

|#bits|#bytes|Signed|Min|Max|Min|Max|
|:---|:---|:---|:---|:---|:---|:---|
|8-bit|1|No|`0`|`255`|`$00`|`$FF`|
|8-bit|1|Yes|`-128`|`127`|`$80`|`$7f`|
|16-bit|2|No|`0`|`65535`|`$0000`|`$FFFF`|
|16-bit|2|Yes|`-32768`|`32767`|`$8000`|`$7FFF`|
|32-bit|4|Yes|`-2147483648`|`2147483647`|`$80000000`|`$7FFFFFFF`|
|32-bit|4|No|`0`|`4294967295`|`$00000000`|`$FFFFFFFF`|

For the 6502 processor and Suzy chip in particular negative numbers in signed math use "twos complement" for 8-bit numbers in the CPU and 16-bit and 32-bit numbers in the math engine of Suzy.

The main distinction between signed and unsigned math is the meaning of a certain binary value as integer number. To illustrate this, have a look at the next table.

|Value (hex)|Value (bin)|Signed byte|Unsigned byte|
|---|---|---|---|
|`$00`|`b00000000`|`0`|`0`|
|`$7f`|`b01111111`|`127`|`127`|
|`$80`|`b10000000`|`-128`|`128`|
|`$81`|`b10000001`|`-127`|`129`|
|`$FF`|`b11111111`|`-1`|`255`|

You can see that the number for a hex value (or its binary representation) differs as soon as the most significant bit is set. You need to take this into account when performing working with the variables and math operations. This becomes relevant when choosing your variable types in C. We will get back to that later.

## Unsigned multiplications
Let's look at unsigned multiplications first. Suzy uses the registers `A` and `B` for the 16-bit multiplicand, `C` and `D` for the 16-bit multiplier and `E`, `F`, `G` and `H` for a 32-bit resultant product. 

Suzy allows you to calculate the product by specifying the multiplicand and multiplier. You specify each by writing to the `AB` and `CD` pairs of bytes and read the resulting product from `EFGH`:

$AB \times CD => EFGH$

> Multiplications and divisions for the Lynx are best understood by using the hexadecimal representation of numbers, as this aligns best with the addresses and registers of Suzy. We will use hexadecimal numbers from here on in.
{: .block-tip }

Some examples of unsigned multiplications in Suzy with the values in the math registers:

| Multiplicand | `A` | `B` | Multiplier | `C` | `D` | Product | `E` | `F` | `G` | `H` |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
|`$1234`|`$12`|`$34`|`$5678`|`$56`|`$78`|`$06260060`|`$06`|`$26`|`$00`|`$60`|
|`$0000`|`$00`|`$00`|`$5678`|`$56`|`$78`|`$00000000`|`$00`|`$00`|`$00`|`$00`|
|`$FFFF`|`$FF`|`$FF`|`$FFFF`|`$FF`|`$FF`|`$FFFE0001`|`$FF`|`$FE`|`$00`|`$01`|

Unsigned 16-bit numbers represent values ranging from `0` to `65535` (`$0000` to `$FFFF`) for each of the two factors `AB` and `CD`. Similarly, the 32-bit product is ranged from `0` to `4294967295` (for `$00000000` to `$FFFFFFFF`) like an unsigned long value. These registers for the individual bytes `A` through `H` are located at the respective hardware addresses of `MATHA` to `MATHH`. Remember that these are not sequential in the address space. Refer to the table above to see the corresponding address for each register.

Multiplications in Suzy are performed by writing the multiplier to registers `D` and `C`, then the multiplicand to `B` and `A`. The final write of `A` starts the multiplication. After a certain amount of ticks the product of `AB` and `CD` will be available in `EFGH` and you can read the result as a 32-bit number.

As you saw before, the hardware addresses for `AB` and `CD` are laid out in reverse order: `MATHD`, `MATHC`, `MATHB` and `MATHA`. You will write the bytes for the values in the multiplication in that exact order for two reasons:

1. Writing to address `MATHA` starts the multiplication process. At that point all bytes must have been written to `AB` and `CD`, so `MATHA` must come last. Because `MATHA` is the highest address a writing order from `MATHD` to `MATHA` is most logical.
2. Since a write to `MATHD` clears `MATHC`, and a write to `MATHB` clears `MATHA`, you need to set `MATHD` before `MATHC`, and `MATHB` before `MATHA`. This avoids overwriting `MATHC` or `MATHA` values. You could not really write to `MATHA` first, as this triggers the start of the operation.

You read the product of the multiplication from most significant byte (MSB) in `E` to least significant byte (LSB) in `H`. As it is typical to have LSB before MSB in memory it is quite convenient that the addresses run from lowest `MATHH` to highest `MATHE`.

Normal unsigned multiplications take 44 ticks to complete. You need to wait this time before reading the result from `EFGH` or you will have an intermediate (and often incorrect) result. Suzy offers a `MATHWORKING` flag of value 0x80 in `SPRSYS` that indicates whether a math operation is in progress. When the `MATHWORKING` flag is set, an operation is still ongoing and you shouldn't read from or write to the math registers. 
You can check whether the math operation is finished with a few CPU instructions:

```asm
notready:
  bit SPRSYS
  bmi notready
```

## Signed multiplications

When performing 16 by 16-bit multiplications with signed math, you can use numbers in the range of negative `-32768` to positive `+32767` decimal (`$8000` to `$7FFF` in hexadecimal) for the multiplicand `AB` and multiplier `CD`. The result is a 32-bit number and can range from 
`-2147483648` to `2147483647` (`$80000000` to `$7FFFFFFF` in hexadecimal).

Signed multiplications in Suzy use the 'twos complement' notation for negative numbers. The most significant (leftmost) bit for a number acts as the negative flag and values run in opposite direction for high to low. 

You can see the usefulness of this notation when you count backwards by repeatedly subtracting a value of `1`. Take a single byte and start at `$02` to get `$01` and `$00` next. Reducing by `1` more will make the byte wrap around to `$FF`. This should represent `-1`, which it does in twos complement form. Counting down further you will get to `$80` at `-128`. Subtracting `1` yields `$7F`, which no longer has the most significant (7th) bit set, representing the positive value of `127`.

You can calculate the hexadecimal notation for a negative number by following these steps:
1. Use absolute (positive) value of the number
2. Perform an exclusive or (XOR) operation with the maximum value (`$FF` for 8-bit, `$FFFF` for 16-bit numbers and `$FFFFFFFF` for 32-bit numbers)
3. Add a value of 1

As an example, the negative number `-4660` in decimal (`-$1234` in hexadecimal) translates to a positive value of `4660` or `$1234`. Using the three steps outlined above, this turns into `$1234 ^ $FFFF + $1` = `$EDCB + $1` = `$EDCC` as a signed 'twos complement' hexadecimal value for `-4660`.

Here are some additional examples for 16-bit by 16-bit to 32-bit multiplications in hexadecimal notation:

```
-$1234 * $5678 = $EDCC * $5678 = $F9D9FFA0 (signed) = -$6260060 = -103.153.760d
-$1234 * -$5678 = $EDCC * $A988 = $ (signed) = -$6260060 = -103.153.760d
```

The same Suzy registers are used in unsigned and signed math. The outcome in `EFGH` differs for negative numbers obviously. 

| Multiplicand | A | B | Multiplier | C | D | Product | E | F | G | H |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
|`-$1234`|`$ED`|`$CC`|`$5678`|`$56`|`$78`|`$F9D9FFA0`|`$F9`|`$D9`|`$FF`|`$A0`|
|`-$1234`|`$ED`|`$CC`|`-$5678`|`$A9`|`$88`|`$06260060`|`$06`|`$26`|`$00`|`$60`|

Suzy only performs multiplications of positive numbers as it turns out. Internally Suzy will change negative values for `AB` or `CD` after writing the upper byte (MSB) of `C` and `A`. Negative numbers will be converted to positive numbers and have their sign stored. The signs of `AB` and `CD` will be applied to the result in `EFGH` after completion. 

This behavior means that for signed math the original values will change if they are negative. In signed math you cannot reuse the original values of `AB` and `CD` as you would in unsigned math. Moreover, because the write to `MATHA` and `MATHC` will trigger the conversion to positive numbers, you must always write to `MATHC`, even if the 'automatic' value of `$00` caused by a write to `MATHD` would be correct for `CD` values smaller than 256. It is only important and relevant for `MATHC` as you will always write to `MATHA` last anyway for the multiplication to start.

Due to a hardware bug in Suzy the value of `$0000` is regarded as a negative number, where this would normally be considered a positive number. This does not matter for the correctness of the result as the outcome will have the incorrect sign of zero re-negated in the result.  However, it does also set the `MATHSIGN` flag in `SPRSYS`. You should not rely on only reading `MATHD` when you multiply by zero.

## Divisions

$ \frac{dividend}{divisor} = quotient + remainder $

$$\frac{EFGH}{NP} = ABCD + LM$$

asdas

The real benefit is that when you have a low (single byte) value for AB or CD you can stop after writing the D or B value.

## Specifics of math operations
- Cannot access Suzy registers during certain operations (which?). If you do, the `UNSAFE_ACCESS` bit in `SPRSYS` is set to `1`
- Unsafe access is broken for math. Reset the bit after every math operation