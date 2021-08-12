---
title: "Generating the MD2 S-Table"
date: 2021-08-12
---

One of my open questions from [Implementing MD2 in Rust]({{< ref "/posts/implementing-md2-in-rust" >}}) was how the `PI_SUBST` [s-box](https://en.wikipedia.org/wiki/S-box) in [appendiex 3](https://datatracker.ietf.org/doc/html/rfc1319#appendix-A.3) was generated.
Some follow-up searching led me to [a crypto stack exchange post](https://crypto.stackexchange.com/questions/11935/how-is-the-md2-hash-function-s-table-constructed-from-pi) with a [response](https://crypto.stackexchange.com/a/18444) from someone who emailed Ron Rivest to get the answer.
Additionally [ryanc](https://crypto.stackexchange.com/users/11778/ryanc) was super helpful in providing a [python implementation](https://crypto.stackexchange.com/a/90139) of that explanation.

Now I generally don't use python but I do know ruby so to try and understand what is going on I ported it to force myself to type everything out.

```ruby
#!/usr/bin/env ruby

S = (0..255).to_a

# First 722 digits of pi
$PI = "31415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608".chars.map(&:to_i)

def rnd(n)
  if n <= 10
    x = $PI.shift
    y = 10
  elsif n <= 100
    x = $PI.shift(2).join.to_i
    y = 100
  elsif
    x = $PI.shift(3).join.to_i
    y = 1000
  end
  
  puts "> #{x} #{y}"

  if x < (n * (y / n))
    return x % n
  else
    return rnd(n)
  end
end

(1..255).each do |i|
  j = rnd(i+1)
  S[j], S[i] = S[i], S[j]
end

puts S.join(',')
```

Which gives us some output that looks like

```console
❯ ruby gen.rb
41,46,67,201,162,216,124,1,61,54,84,161,236,240,6,19,98,167,5,243,192,199,115,140,152,147,43,217,188,76,130,202,30,155,87,60,253,212,224,22,103,66,111,24,138,23,229,18,190,78,196,214,218,158,222,73,160,251,245,142,187,47,238,122,169,104,121,145,21,178,7,63,148,194,16,137,11,34,95,33,128,127,93,154,90,144,50,39,53,62,204,231,191,247,151,3,255,25,48,179,72,165,181,209,215,94,146,42,172,86,170,198,79,184,56,210,150,164,125,182,118,252,107,226,156,116,4,241,69,157,112,89,100,113,135,32,134,91,207,101,230,45,168,2,27,96,37,173,174,176,185,246,28,70,97,105,52,64,126,15,85,71,163,35,221,81,175,58,195,92,249,206,186,197,234,38,44,83,13,110,133,40,132,9,211,223,205,244,65,129,77,82,106,220,55,200,108,193,171,250,36,225,123,8,12,189,177,74,120,136,149,139,227,99,232,109,233,203,213,254,59,0,29,57,242,239,183,14,102,88,208,228,166,119,114,248,235,117,75,10,49,68,80,180,143,237,31,26,219,153,141,51,159,17,131,20
```

Which matches the output in the RFC.

This should be fairly straight forward to add to our existing rust code and see if our tests continue to pass.

```rust
fn gen_sbox() -> [u8; 256] {
    let mut s = [0 as u8; 256];
    for i in 0..=255 {
        s[i as usize] = i;
    }

    let mut pi: Vec<u8> = "31415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608"
        .chars()
        .map(|x| x.to_digit(10))
        .map(Option::unwrap)
        .map(|x| x as u8)
        .collect();

    for i in 1..=255 {
        let j = s_box_digit(i+1, &mut pi);
        let tmp = s[j];

        s[j] = s[i];
        s[i] = tmp;
    }

    s
}

fn s_box_digit(n: usize, pi: &mut Vec<u8>) -> usize {
    let (x, y) = match n {
        0..=10 => (pi.drain(0..1), 10),
        11..=100 => (pi.drain(0..2), 100),
        _ => (pi.drain(0..3), 1000),
    };

    let x: usize = x
        .map(|x| x.to_string() )
        .collect::<Vec<String>>()
        .concat()
        .parse()
        .unwrap();

    if x < (n * (y / n)) {
        x % n
    } else {
        s_box_digit(n, pi)
    }
}

fn md2(message: &[u8]) -> String {
    let s_box = gen_sbox();
    //...
```

And our tests pass.

Now I'm not super happy with this for a number of aesthetic reasons but the biggest issue is the performance hit we just introduced.
In our previous implementation, this array was hard coded and added no overhead to use.
This version does explain how the S box was generated, so that's the improvement we wanted, but it comes at a cost we would like to avoid.

Lets talk about `const fn`.

# Compile Time Work

Rust gives us an interesting ability to separate concerns into runtime concerns and compile time concerns.
What we can do is implement this using [constant evaluation](https://doc.rust-lang.org/reference/const_eval.html) letting us encode the generation method into the code while embedding only the generated s box in the binary giving us the same performance as when we hard coded it while preserving the understanding of how we got there.

There are some constraints on a `const fn` function that will force us to modify our implementation.

The first issue is there is no support for using `for` loops.
Really any iterator is going to cause us problems, but work is being done in [issue 87575](https://github.com/rust-lang/rust/issues/87575) to stabilize the `#![feature(const_for)]` feature.
I would prefer to use stable rust for this so instead we will use index variables with `while` loops to traverse our data.

We also cannot use `&mut` references in `const fn` functions.
There is a [tracking issue](https://github.com/rust-lang/rust/issues/57349) for this as well, but we will need to use an alternative to our mutable `Vec` which contains the digits of PI.
Instead what we an do is pass in an immutable array containing the digits and an index so we know which values to read next.
We can also return from this function the number of digits we have read, which allows us to keep the index accurate.
This retains all the mutability in one function while still allowing us to break out the digit generation into it's own function for clarity.

Finally we need to avoid calling any destructors as specified in [E0493](https://doc.rust-lang.org/error-index.html#E0493).
This means our use of a `String` to build up the potentially 3 digit number will have to be ported into a pure arithmetic expression.

These changes give us an implementation that looks like this.

```rust
const S_BOX: [u8; 256] = gen_sbox();

const fn gen_sbox() -> [u8; 256] {
    let mut s = [0 as u8; 256];

    let pi_digits = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5, 8, 9, 7, 9, 3, 2, 3, 8, 4, 6, 2, 6, 4, 3, 3, 8, 3, 2, 7, 9, 5, 0, 2, 8, 8, 4, 1, 9, 7, 1, 6, 9, 3, 9, 9, 3, 7, 5, 1, 0, 5, 8, 2, 0, 9, 7, 4, 9, 4, 4, 5, 9, 2, 3, 0, 7, 8, 1, 6, 4, 0, 6, 2, 8, 6, 2, 0, 8, 9, 9, 8, 6, 2, 8, 0, 3, 4, 8, 2, 5, 3, 4, 2, 1, 1, 7, 0, 6, 7, 9, 8, 2, 1, 4, 8, 0, 8, 6, 5, 1, 3, 2, 8, 2, 3, 0, 6, 6, 4, 7, 0, 9, 3, 8, 4, 4, 6, 0, 9, 5, 5, 0, 5, 8, 2, 2, 3, 1, 7, 2, 5, 3, 5, 9, 4, 0, 8, 1, 2, 8, 4, 8, 1, 1, 1, 7, 4, 5, 0, 2, 8, 4, 1, 0, 2, 7, 0, 1, 9, 3, 8, 5, 2, 1, 1, 0, 5, 5, 5, 9, 6, 4, 4, 6, 2, 2, 9, 4, 8, 9, 5, 4, 9, 3, 0, 3, 8, 1, 9, 6, 4, 4, 2, 8, 8, 1, 0, 9, 7, 5, 6, 6, 5, 9, 3, 3, 4, 4, 6, 1, 2, 8, 4, 7, 5, 6, 4, 8, 2, 3, 3, 7, 8, 6, 7, 8, 3, 1, 6, 5, 2, 7, 1, 2, 0, 1, 9, 0, 9, 1, 4, 5, 6, 4, 8, 5, 6, 6, 9, 2, 3, 4, 6, 0, 3, 4, 8, 6, 1, 0, 4, 5, 4, 3, 2, 6, 6, 4, 8, 2, 1, 3, 3, 9, 3, 6, 0, 7, 2, 6, 0, 2, 4, 9, 1, 4, 1, 2, 7, 3, 7, 2, 4, 5, 8, 7, 0, 0, 6, 6, 0, 6, 3, 1, 5, 5, 8, 8, 1, 7, 4, 8, 8, 1, 5, 2, 0, 9, 2, 0, 9, 6, 2, 8, 2, 9, 2, 5, 4, 0, 9, 1, 7, 1, 5, 3, 6, 4, 3, 6, 7, 8, 9, 2, 5, 9, 0, 3, 6, 0, 0, 1, 1, 3, 3, 0, 5, 3, 0, 5, 4, 8, 8, 2, 0, 4, 6, 6, 5, 2, 1, 3, 8, 4, 1, 4, 6, 9, 5, 1, 9, 4, 1, 5, 1, 1, 6, 0, 9, 4, 3, 3, 0, 5, 7, 2, 7, 0, 3, 6, 5, 7, 5, 9, 5, 9, 1, 9, 5, 3, 0, 9, 2, 1, 8, 6, 1, 1, 7, 3, 8, 1, 9, 3, 2, 6, 1, 1, 7, 9, 3, 1, 0, 5, 1, 1, 8, 5, 4, 8, 0, 7, 4, 4, 6, 2, 3, 7, 9, 9, 6, 2, 7, 4, 9, 5, 6, 7, 3, 5, 1, 8, 8, 5, 7, 5, 2, 7, 2, 4, 8, 9, 1, 2, 2, 7, 9, 3, 8, 1, 8, 3, 0, 1, 1, 9, 4, 9, 1, 2, 9, 8, 3, 3, 6, 7, 3, 3, 6, 2, 4, 4, 0, 6, 5, 6, 6, 4, 3, 0, 8, 6, 0, 2, 1, 3, 9, 4, 9, 4, 6, 3, 9, 5, 2, 2, 4, 7, 3, 7, 1, 9, 0, 7, 0, 2, 1, 7, 9, 8, 6, 0, 9, 4, 3, 7, 0, 2, 7, 7, 0, 5, 3, 9, 2, 1, 7, 1, 7, 6, 2, 9, 3, 1, 7, 6, 7, 5, 2, 3, 8, 4, 6, 7, 4, 8, 1, 8, 4, 6, 7, 6, 6, 9, 4, 0, 5, 1, 3, 2, 0, 0, 0, 5, 6, 8, 1, 2, 7, 1, 4, 5, 2, 6, 3, 5, 6, 0, 8, 2, 7, 7, 8, 5, 7, 7, 1, 3, 4, 2, 7, 5, 7, 7, 8, 9, 6, 0, 9, 1, 7, 3, 6, 3, 7, 1, 7, 8, 7, 2, 1, 4, 6, 8, 4, 4, 0, 9, 0, 1, 2, 2, 4, 9, 5, 3, 4, 3, 0, 1, 4, 6, 5, 4, 9, 5, 8, 5, 3, 7, 1, 0, 5, 0, 7, 9, 2, 2, 7, 9, 6, 8, 9, 2, 5, 8, 9, 2, 3, 5, 4, 2, 0, 1, 9, 9, 5, 6, 1, 1, 2, 1, 2, 9, 0, 2, 1, 9, 6, 0, 8];
    let mut pi_index: usize = 0;

    let mut i = 0;
    while i < 256 {
        s[i] = i as u8;
        i = i + 1;
    }

    let mut i = 1;
    while i < 256 {
        let (j, pi_index_bump) = gen_sbox_digit(i + 1, pi_digits, pi_index);

        let tmp = s[j as usize];
        s[j as usize] = s[i as usize];
        s[i as usize] = tmp;

        pi_index = pi_index + pi_index_bump;
        i = i + 1;
    }

    s
}

const fn gen_sbox_digit(n: usize, pi_digits: [u8; 722], pi_index: usize) -> (u8, usize) {
    let (d1, d2, d3, y, pi_index_bump) = match n {
        0..=10 => {
            (0, 0, pi_digits[pi_index], 10, 1)
        },
        11..=100 => {
            (0, pi_digits[pi_index], pi_digits[pi_index + 1], 100, 2)
        },
        _ => {
            (pi_digits[pi_index], pi_digits[pi_index + 1], pi_digits[pi_index + 2], 1000, 3)
        },
    };

    let x: usize = (d1 as usize * 100) + (d2 as usize * 10) + d3 as usize;
    
    if x < (n * (y / n)) {
        return ((x % n) as u8, pi_index_bump);
    } else {
        let (j, pi_index_bump_add) = gen_sbox_digit(n, pi_digits, pi_index + pi_index_bump);
        return (j, pi_index_bump + pi_index_bump_add);
    }
}
```

And checking in with our tests

```console
❯ cargo test
   Compiling md2 v0.1.0 (/Users/evan/Developer/md2/rust)
    Finished test [unoptimized + debuginfo] target(s) in 1.78s
     Running unittests (target/debug/deps/md2-20d94f32c0345aa7)

running 7 tests
test tests::test_123456789012345678901234567890123456789012345678901234567890123456789012345678 ... ok
test tests::test_ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789 ... ok
test tests::test_a ... ok
test tests::test_abc ... ok
test tests::test_empty ... ok
test tests::test_abcdefghijklmnopqrstuvwxyz ... ok
test tests::test_message_digest ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

They still pass.

# Was it worth it?
I think so.
Pre-generating something like an s box at compile time feels like a perfect use case for this language feature.
We get to maintain the knowledge of how the data was generated but at runtime we are acting as if we used the opaque hard coded table provided in the rfc.
The resulting code however does feel less rusty so I wouldn't be jumping yet at every opportunity to port functionality, but I will be following those tracking issues with interest as they develop.
