---
title: "Implementing MD2 in Rust"
date: 2021-08-05T13:01:47-04:00
---

When I'm trying to learn about some software, I personally find it most effective to try and implement it myself.
I find that by getting actual code working, it helps me understand what is going on on a deeper more meaningful level.
Lately I have been wanting to understand more about how [cryptographic hash functions](https://en.wikipedia.org/wiki/Cryptographic_hash_function) work.
Additionally, older things tend to be simpler and easier to understand before trying to build up to newer, more complex concepts.

[MD2](https://en.wikipedia.org/wiki/MD2_(cryptography)) was developed in 1989 and [RFC 1319](https://datatracker.ietf.org/doc/html/rfc1319) which defines it is only 17 pages long, including the example C implementation.

```terminal
$ cargo new md2
```

Lets do this

# Sections 1 and 2
These sections cover the executive summary, some details about the general behavior of message digest algorithms, licensing, and other definitions of terms.
It's important to note that the output digest of MD2 is a 128 bit fingerprint.

# Section 3
Section 3 covers the description of the algorithm itself.
This is where we get to start writing code to implement.

Before we start, [appendix 6](https://datatracker.ietf.org/doc/html/rfc1319#appendix-A.5) contains a test suite of input values and the hash that would result in processing them.
Lets throw that into some unit tests so we can have confidence we implemented things correctly when we are done.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_empty() {
        assert_eq!(md2(b""), "8350e5a3e24c153df2275c9f80692773");
    }
    
    #[test]
    fn test_a() {
        assert_eq!(md2(b"a"), "32ec01ec4a6dac72c0ab96fb34c0b5d1");
    }
    
    #[test]
    fn test_abc() {
        assert_eq!(md2(b"abc"), "da853b0d3f88d99b30283a69e6ded6bb");
    }
    
    #[test]
    fn test_message_digest() {
        assert_eq!(md2(b"message digest"), "ab4f496bfb2a530b219ff33031fe06b0");
    } 
    
    #[test]
    fn test_abcdefghijklmnopqrstuvwxyz() {
        assert_eq!(md2(b"abcdefghijklmnopqrstuvwxyz"), "4e8ddff3650292ab5a4108c3aa47940b");
    }
    #[test]
    fn test_ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789() {
        assert_eq!(md2(b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"), "da33def2a42df13975352846c30338cd");
    }
        
    #[test]
    fn test_123456789012345678901234567890123456789012345678901234567890123456789012345678() {
        assert_eq!(md2(b"12345678901234567890123456789012345678901234567890123456789012345678901234567890"), "d5976f79d83d3a0dc9806c3c66f3efd8");
    }
}
```

We are using the [Byte string literal](https://dev-doc.rust-lang.org/beta/book/appendix-02-operators.html#non-operator-symbols) syntax of `b""` to define our strings as a raw `[u8]` instead of the `&str` rust would insert normally.
MD2 is intended to operate on raw bytes and this saves us the step of having to convert the strings ourself.

Running this gives us the expected error.

```terminal
❯ cargo test
   Compiling md2 v0.1.0 (/Users/evan/Developer/md2/rust)
error[E0425]: cannot find function `md2` in this scope
  --> src/main.rs:11:20
   |
11 |         assert_eq!(md2(""), "8350e5a3e24c153df2275c9f80692773");
   |                    ^^^ not found in this scope
```

Lets get started.

# Padding
There are 3 main takeaways to the padding process according to [section 3.1](https://datatracker.ietf.org/doc/html/rfc1319#section-3.1)
- The message is padded to 0 modulo 16, meaning the final message size is always evenly divisible by 16.
- The message is _always_ padded.
- The value of the bytes added for padding contain the number of bytes we are padding. Meaning if we need to add two bytes to the message, we are appending `0x02 0x02`.

Because we are always going to pad the message, and the padded message must be evenly divisible by 16, we are at most going to append 16 bytes to a message already evenly divisible by 16.
What we can do is start with 16, and then remove what the existing message length modulo 16 is, which will give us the amount of padding needed to achieve a message length evenly divisible by 16.

After we calculate the length we need to pad, we can use the [vec! macro](https://doc.rust-lang.org/std/macro.vec.html) to create a [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) with the correct size and contents easily.
We then create a new `padded_message` which contains our initial message with our padding appended to it.

```rust
fn md2(message: &[u8]) -> String {
    let padding_length = 16 - (message.len() % 16);
    let mut padding = vec![padding_length as u8; padding_length];

    let mut padded_message: Vec<u8> = vec![];
    padded_message.append(&mut message.clone().to_vec());
    padded_message.append(&mut padding);
    
    //...
```

We are going to have to use `as u8` a lot throughout this code. By default rust will want to use [usize](https://doc.rust-lang.org/std/primitive.usize.html) but we are explicitly dealing with 1 byte values as per the RFC.

# Appending the checksum
[Section 3.2](https://datatracker.ietf.org/doc/html/rfc1319#section-3.2) covers calculation and appending of a checksum value.
It's important to note
- The checksum is 16 bytes long
- It uses a pre-generated [S Box](https://en.wikipedia.org/wiki/S-box) that's found in the appendix
- We operate on our message in 16 byte blocks

We can copy the  s box from the appendix, and because it's considered a constant we can define it as such

```rust
const S_BOX: [u8; 256] = [
    41, 46, 67, 201, 162, 216, 124, 1, 61, 54, 84, 161, 236, 240, 6, 19, 98, 167, 5, 243, 192, 199,
    115, 140, 152, 147, 43, 217, 188, 76, 130, 202, 30, 155, 87, 60, 253, 212, 224, 22, 103, 66,
    111, 24, 138, 23, 229, 18, 190, 78, 196, 214, 218, 158, 222, 73, 160, 251, 245, 142, 187, 47,
    238, 122, 169, 104, 121, 145, 21, 178, 7, 63, 148, 194, 16, 137, 11, 34, 95, 33, 128, 127, 93,
    154, 90, 144, 50, 39, 53, 62, 204, 231, 191, 247, 151, 3, 255, 25, 48, 179, 72, 165, 181, 209,
    215, 94, 146, 42, 172, 86, 170, 198, 79, 184, 56, 210, 150, 164, 125, 182, 118, 252, 107, 226,
    156, 116, 4, 241, 69, 157, 112, 89, 100, 113, 135, 32, 134, 91, 207, 101, 230, 45, 168, 2, 27,
    96, 37, 173, 174, 176, 185, 246, 28, 70, 97, 105, 52, 64, 126, 15, 85, 71, 163, 35, 221, 81,
    175, 58, 195, 92, 249, 206, 186, 197, 234, 38, 44, 83, 13, 110, 133, 40, 132, 9, 211, 223, 205,
    244, 65, 129, 77, 82, 106, 220, 55, 200, 108, 193, 171, 250, 36, 225, 123, 8, 12, 189, 177, 74,
    120, 136, 149, 139, 227, 99, 232, 109, 233, 203, 213, 254, 59, 0, 29, 57, 242, 239, 183, 14,
    102, 88, 208, 228, 166, 119, 114, 248, 235, 117, 75, 10, 49, 68, 80, 180, 143, 237, 31, 26,
    219, 153, 141, 51, 159, 17, 131, 20,
];
```

An interesting thing to note about the s box is that it's explicitly generated from the digits of [pi](https://en.wikipedia.org/wiki/Pi) making this a [nothing up my sleeve number](https://en.wikipedia.org/wiki/Nothing-up-my-sleeve_number).

We can define our 16 byte checksum and initialize it to `0` using the normal [array](https://doc.rust-lang.org/std/primitive.array.html) syntax.
Because we want to operate on 16 byte chunks, the pseudo code used in the RFC first divides the length of the padded message by 16, then uses `i*16` with relative offsets to operate within that block for a given loop.
We are instead going to use the [chunks](https://doc.rust-lang.org/std/slice/struct.Chunks.html) method to iterate over our Vec in 16 byte chunks so we can omit the complexity of including those indexed offsets while addressing values.


After we construct our `checksum`, we then append it to our `padded_message`.

```rust
    //...
    
    let mut checksum: [u8; 16] = [0; 16];
    let mut l = 0;
    for block in padded_message.chunks(16) {
        for j in 0..16 {
            let c = block[j];
            checksum[j] ^= S_BOX[(c ^ l) as usize];
            l = checksum[j as usize];
        }
    }
    padded_message.append(&mut checksum.to_vec());
    
    // ...
```

Here we also see the inverse of our `as u8` above with `as usize`. Rust wants to do those array index lookups with a usize. Thankfully a u8 can always perfectly be converted to a usize which is larger so we can just cast it in place.

# Generating the digest
Now that we have our checksummed value, we can proceed with sections [3.3](https://datatracker.ietf.org/doc/html/rfc1319#section-3.3) and [3.4](https://datatracker.ietf.org/doc/html/rfc1319#section-3.4) which is how we actually calculate the hash.


First we inititalize a 48 byte space with zeros and then iterate though the message blocks following the described algorithm.
Once again we are going to use the `chunks` method to make it easer to calculate the correct addresses.


This line of code is also of particular note

```
            Set t to (t+j) modulo 256.
```

This basically defines intentional integer overflowing.
In C we could do something like this

```c
#include <stdio.h>

int main() {
  char x = 256; // We use `char` here as an explicit 8 bit value for this example
  x = x + 5;
  printf("%d\n", x);
}
```

That program would generate this output

```
$ gcc example.c && ./a.out
5
$
```

Below in the example C implementation the following is done to bitmask 255 and give a similar result for a `t` variable size larger than 8 bits.

```c
  t = (t + i) & 0xff;
```

Because we are going to explicitly define a `u8` we are just going to use [wrapping_add](https://doc.rust-lang.org/std/primitive.u8.html#method.wrapping_add) instead.


```rust
    //...
    
    let mut digest: [u8; 48] = [0; 48];
    for block in padded_message.chunks(16) {
        for j in 0..16 {
            digest[16 + j] = block[j];
            digest[32 + j] = digest[16 + j] ^ digest[j];
        }

        let mut t: u8 = 0;

        for j in 0..18 {
            for k in 0..48 {
                digest[k] = digest[k] ^ S_BOX[t as usize];
                t = digest[k]
            }

            t = t.wrapping_add(j);
        }
    }
    
    //...
```

# Formatting our result
Now that we have a generated digest, we can output the first 16 bytes as hexidecemal as desctibed in [section 3.5](https://datatracker.ietf.org/doc/html/rfc1319#section-3.5).
We will take the first 16 bytes, map them into a hexadecimal value, collect them into a string and return that value as our output.

```rust
    //...
    
    digest[0..16]
        .into_iter()
        .map(|x| format!("{:x}", x))
        .collect::<Vec<String>>()
        .join("")
}
```

# Testing our code

Lets check in with that test suite

```console
❯ cargo test
   Compiling md2 v0.1.0 (/Users/evan/Developer/md2/rust)
    Finished test [unoptimized + debuginfo] target(s) in 0.50s
     Running unittests (target/debug/deps/md2-20d94f32c0345aa7)

running 7 tests
test tests::test_a ... FAILED
test tests::test_ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789 ... FAILED
test tests::test_123456789012345678901234567890123456789012345678901234567890123456789012345678 ... FAILED
test tests::test_abc ... FAILED
test tests::test_abcdefghijklmnopqrstuvwxyz ... FAILED
test tests::test_empty ... ok
test tests::test_message_digest ... FAILED

failures:

---- tests::test_a stdout ----
thread 'tests::test_a' panicked at 'assertion failed: `(left == right)`
  left: `"32ec1ec4a6dac72c0ab96fb34c0b5d1"`,
 right: `"32ec01ec4a6dac72c0ab96fb34c0b5d1"`', src/main.rs:78:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

....
```

Oh.

Oh no.

It looks like we have made a mistake.

However it looks like our problem may be in the output.
If we were generating the digest incorrectly, or the checksum or padding, given the characteristics of a cryptographic hash the values should be wildly different.
Instead it looks like our length is wrong somtimes.

If we look closer at the specific example, we can see we expected `01` to follow `ec` but instead we got `1e`.
This makes me think we didn't correctly define how long each formatted block should be.
If the value was small enough, it could be represented with one hexadecimal value but we _always_ want it to be represented with two.

Lets try this

```rust
    digest[0..16]
        .into_iter()
        .map(|x| format!("{:02x}", x))
        .collect::<Vec<String>>()
        .join("")
}
```

And the test suite says...

```console
❯ cargo test
   Compiling md2 v0.1.0 (/Users/evan/Developer/md2/rust)
    Finished test [unoptimized + debuginfo] target(s) in 0.76s
     Running unittests (target/debug/deps/md2-20d94f32c0345aa7)

running 7 tests
test tests::test_a ... ok
test tests::test_ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789 ... ok
test tests::test_123456789012345678901234567890123456789012345678901234567890123456789012345678 ... ok
test tests::test_abc ... ok
test tests::test_abcdefghijklmnopqrstuvwxyz ... ok
test tests::test_empty ... ok
test tests::test_message_digest ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

🎉

# Things I learned

1. I recognized the term s box before but it was nice to use one and really understand their function.
2. The idea of a nothing up your sleeve number was new and I'm very happy to have a term to cover the concept for how to give some confidence in otherwise opaque magic numbers.
3. I _really_ wish the RFC included tests for each major step of the process.
  Having clear examples of how a specific message should be padded, or a way to verify the checksum was generated correctly would have helped my confidence a lot.
  In the future when writing similar documentation myself, I want to help enable people trying to implement my spec [TDD](https://en.wikipedia.org/wiki/Test-driven_development) their way though it.
4. I wish information on how the S box was generated was included in the RFC.
  From this document alone I have no way to verify it was actually generated in some repeatable way from pi.

# Final Code
Here is my complete implementation of MD2.
Please don't actually use it for anything.

```rust
const S_BOX: [u8; 256] = [
    41, 46, 67, 201, 162, 216, 124, 1, 61, 54, 84, 161, 236, 240, 6, 19, 98, 167, 5, 243, 192, 199,
    115, 140, 152, 147, 43, 217, 188, 76, 130, 202, 30, 155, 87, 60, 253, 212, 224, 22, 103, 66,
    111, 24, 138, 23, 229, 18, 190, 78, 196, 214, 218, 158, 222, 73, 160, 251, 245, 142, 187, 47,
    238, 122, 169, 104, 121, 145, 21, 178, 7, 63, 148, 194, 16, 137, 11, 34, 95, 33, 128, 127, 93,
    154, 90, 144, 50, 39, 53, 62, 204, 231, 191, 247, 151, 3, 255, 25, 48, 179, 72, 165, 181, 209,
    215, 94, 146, 42, 172, 86, 170, 198, 79, 184, 56, 210, 150, 164, 125, 182, 118, 252, 107, 226,
    156, 116, 4, 241, 69, 157, 112, 89, 100, 113, 135, 32, 134, 91, 207, 101, 230, 45, 168, 2, 27,
    96, 37, 173, 174, 176, 185, 246, 28, 70, 97, 105, 52, 64, 126, 15, 85, 71, 163, 35, 221, 81,
    175, 58, 195, 92, 249, 206, 186, 197, 234, 38, 44, 83, 13, 110, 133, 40, 132, 9, 211, 223, 205,
    244, 65, 129, 77, 82, 106, 220, 55, 200, 108, 193, 171, 250, 36, 225, 123, 8, 12, 189, 177, 74,
    120, 136, 149, 139, 227, 99, 232, 109, 233, 203, 213, 254, 59, 0, 29, 57, 242, 239, 183, 14,
    102, 88, 208, 228, 166, 119, 114, 248, 235, 117, 75, 10, 49, 68, 80, 180, 143, 237, 31, 26,
    219, 153, 141, 51, 159, 17, 131, 20,
];

fn md2(message: &[u8]) -> String {
    let padding_length = 16 - (message.len() % 16);
    let mut padding = vec![padding_length as u8; padding_length];

    let mut padded_message: Vec<u8> = vec![];
    padded_message.append(&mut message.clone().to_vec());
    padded_message.append(&mut padding);

    let mut checksum: [u8; 16] = [0; 16];
    let mut l = 0;
    for block in padded_message.chunks(16) {
        for j in 0..16 {
            let c = block[j];
            checksum[j] ^= S_BOX[(c ^ l) as usize];
            l = checksum[j as usize];
        }
    }
    padded_message.append(&mut checksum.to_vec());

    let mut digest: [u8; 48] = [0; 48];
    for block in padded_message.chunks(16) {
        for j in 0..16 {
            digest[16 + j] = block[j];
            digest[32 + j] = digest[16 + j] ^ digest[j];
        }

        let mut t: u8 = 0;

        for j in 0..18 {
            for k in 0..48 {
                digest[k] = digest[k] ^ S_BOX[t as usize];
                t = digest[k]
            }

            t = t.wrapping_add(j);
        }
    }

    digest[0..16]
        .into_iter()
        .map(|x| format!("{:02x}", x))
        .collect::<Vec<String>>()
        .join("")
}

fn main() {
    println!("{}", md2(b"abcdefghijklmnopqrstuvwxyz"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_empty() {
        assert_eq!(md2(b""), "8350e5a3e24c153df2275c9f80692773");
    }
    
    #[test]
    fn test_a() {
        assert_eq!(md2(b"a"), "32ec01ec4a6dac72c0ab96fb34c0b5d1");
    }
    
    #[test]
    fn test_abc() {
        assert_eq!(md2(b"abc"), "da853b0d3f88d99b30283a69e6ded6bb");
    }
    
    #[test]
    fn test_message_digest() {
        assert_eq!(md2(b"message digest"), "ab4f496bfb2a530b219ff33031fe06b0");
    } 
    
    #[test]
    fn test_abcdefghijklmnopqrstuvwxyz() {
        assert_eq!(md2(b"abcdefghijklmnopqrstuvwxyz"), "4e8ddff3650292ab5a4108c3aa47940b");
    }

    #[test]
    #[allow(non_snake_case)]
    fn test_ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789() {
        assert_eq!(md2(b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"), "da33def2a42df13975352846c30338cd");
    }
        
    #[test]
    fn test_123456789012345678901234567890123456789012345678901234567890123456789012345678() {
        assert_eq!(md2(b"12345678901234567890123456789012345678901234567890123456789012345678901234567890"), "d5976f79d83d3a0dc9806c3c66f3efd8");
    }
}
```
