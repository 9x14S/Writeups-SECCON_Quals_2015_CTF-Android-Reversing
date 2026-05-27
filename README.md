# SECCON Quals 2015 CTF, Android Reverse Engineering
[Source](https://github.com/ctfs/write-ups-2015/tree/master/seccon-quals-ctf-2015/binary/reverse-engineering-android-apk-1)

## Initial steps

After downloading the binary, we need to extract the Android Package contents. I use `apktool` for this since `unzip` wouldn't correctly decode the `AndroidManifest.xml` file and would leave us a compressed `classes.dex` file.

The resulting file structure is as follows (I've omitted the `res` and the `smali/android` directories):
```
rps
├── AndroidManifest.xml
├── apktool.yml
├── lib
│   ├── armeabi
│   │   └── libcalc.so
│   ├── armeabi-v7a
│   │   └── libcalc.so
│   ├── mips
│   │   └── libcalc.so
│   └── x86
│       └── libcalc.so
├── original
│   ├── AndroidManifest.xml
│   └── META-INF
│       ├── CERT.RSA
│       ├── CERT.SF
│       └── MANIFEST.MF
└── smali
    └── com
        └── example
            └── seccon2015
                └── rock_paper_scissors
                    ├── BuildConfig.smali
                    ├── MainActivity$1.smali
                    ├── MainActivity.smali
                    ├── R$anim.smali
                    ├── R$attr.smali
                    ├── R$bool.smali
                    ├── R$color.smali
                    ├── R$dimen.smali
                    ├── R$drawable.smali
                    ├── R$id.smali
                    ├── R$integer.smali
                    ├── R$layout.smali
                    ├── R$mipmap.smali
                    ├── R$string.smali
                    ├── R$styleable.smali
                    ├── R$style.smali
                    └── R.smali
```

Here we can see the `.smali` files, which contain disassembled Dalvik Executable (dex) code.

The application's entry point files are `MainActivity.smali` (the actual entry point) and `MainActivity$1.smali`, which is an anonymous class used in the application and contains some extra logic.

The `R$<stuff>.smali` files are resource and reference files, which contain string references and other information not stored directly in the main classes. These are auto-generated and aren't really useful for solving this challenge (I didn't even read them while solving this challenge).

`BuildConfig.smali` usually contains the application's build configuration, such as whether the Debug mode was enabled, application versions, Java versions.

Another interesting thing is the `lib/` directory, which contains architecture-specific machine code libraries with exported functions. This is useful for this challenge since in the libraries there's a function used for generating the flag.

The next step would be here to decompile the `MainActivity.smali` and `MainActivity$1.smali` files to `.java` files. I used `jadx MainActivity.smali MainActivity\$1.smali` through the command-line.

## Taking a look into the program logic

After decompiling the application logic, we get this
```java

package com.example.seccon2015.rock_paper_scissors;

import android.app.Activity;
import android.os.Bundle;
import android.os.Handler;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import java.util.Random;

public class MainActivity extends Activity implements View.OnClickListener {
    Button P;
    Button S;
    int flag;
    int m;
    int n;
    Button r;
    int cnt = 0;
    private final Handler handler = new Handler();
    private final Runnable showMessageTask = new Runnable() { // from class: com.example.seccon2015.rock_paper_scissors.MainActivity.1
        @Override // java.lang.Runnable
        public void run() {
            TextView tv3 = (TextView) MainActivity.this.findViewById(2131492946);
            if (MainActivity.this.n - MainActivity.this.m == 1) {
                MainActivity.this.cnt++;
                tv3.setText("WIN! +" + String.valueOf(MainActivity.this.cnt));
            } else if (MainActivity.this.m - MainActivity.this.n == 1) {
                MainActivity.this.cnt = 0;
                tv3.setText("LOSE +0");
            } else if (MainActivity.this.m == MainActivity.this.n) {
                tv3.setText("DRAW +" + String.valueOf(MainActivity.this.cnt));
            } else if (MainActivity.this.m < MainActivity.this.n) {
                MainActivity.this.cnt = 0;
                tv3.setText("LOSE +0");
            } else {
                MainActivity.this.cnt++;
                tv3.setText("WIN! +" + String.valueOf(MainActivity.this.cnt));
            }
            if (1000 == MainActivity.this.cnt) {
                tv3.setText("SECCON{" + String.valueOf((MainActivity.this.cnt + MainActivity.this.calc()) * 107) + "}");
            }
            MainActivity.this.flag = 0;
        }
    };

    public native int calc();

    static {
        System.loadLibrary("calc");
    }

    @Override // android.app.Activity
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(2130968600);
        this.P = (Button) findViewById(2131492941);
        this.S = (Button) findViewById(2131492943);
        this.r = (Button) findViewById(2131492942);
        this.P.setOnClickListener(this);
        this.r.setOnClickListener(this);
        this.S.setOnClickListener(this);
        this.flag = 0;
    }

    @Override // android.view.View.OnClickListener
    public void onClick(View v) {
        if (this.flag != 1) {
            this.flag = 1;
            TextView tv3 = (TextView) findViewById(2131492946);
            tv3.setText("");
            TextView tv = (TextView) findViewById(2131492944);
            TextView tv2 = (TextView) findViewById(2131492945);
            this.m = 0;
            Random rm = new Random();
            this.n = rm.nextInt(3);
            String[] ss = {"CPU: Paper", "CPU: Rock", "CPU: Scissors"};
            tv2.setText(ss[this.n]);
            if (v == this.P) {
                tv.setText("YOU: Paper");
                this.m = 0;
            }
            if (v == this.r) {
                tv.setText("YOU: Rock");
                this.m = 1;
            }
            if (v == this.S) {
                tv.setText("YOU: Scissors");
                this.m = 2;
            }
            this.handler.postDelayed(this.showMessageTask, 1000L);
        }
    }
}
```

After quickly skimming over it, we can see where is the flag calculated and generated in the code, right in `showMessageTask`.

Let's take a deeper look in the flag-generating code:
```java
private final Runnable showMessageTask = new Runnable() { // from class: com.example.seccon2015.rock_paper_scissors.MainActivity.1
    @Override // java.lang.Runnable
    public void run() {
        TextView tv3 = (TextView) MainActivity.this.findViewById(2131492946);
        if (MainActivity.this.n - MainActivity.this.m == 1) {
            MainActivity.this.cnt++;
            tv3.setText("WIN! +" + String.valueOf(MainActivity.this.cnt));
        } else if (MainActivity.this.m - MainActivity.this.n == 1) {
            MainActivity.this.cnt = 0;
            tv3.setText("LOSE +0");
        } else if (MainActivity.this.m == MainActivity.this.n) {
            tv3.setText("DRAW +" + String.valueOf(MainActivity.this.cnt));
        } else if (MainActivity.this.m < MainActivity.this.n) {
            MainActivity.this.cnt = 0;
            tv3.setText("LOSE +0");
        } else {
            MainActivity.this.cnt++;
            tv3.setText("WIN! +" + String.valueOf(MainActivity.this.cnt));
        }
        if (1000 == MainActivity.this.cnt) {
            tv3.setText("SECCON{" + String.valueOf((MainActivity.this.cnt + MainActivity.this.calc()) * 107) + "}");
        }
        MainActivity.this.flag = 0;
    }
};
```

We can see here that there's some comparison logic which if true, prints the flag. This if statement checks if the `cnt` variable is exactly `1000` before it prints the flag.

The flag can be extracted directly from this statement alone, no need to check the rest of the code, but for sake of completion I will explain the rest of the code.

To get `cnt` to `1000`, we need to check the previous `if-else if-else` sequence. `cnt` is incremented in exactly two blocks, in the `else` block and the first `if`. Any other block either resets `cnt` to zero, or doesn't change it (in a `DRAW`).

Since the game is a rock-paper-scissors game, we can deduce that the variables used in the conditional checks are a way of representing the "hand shapes", which is confirmed by this block of code:
```java
public void onClick(View v) {
    if (this.flag != 1) {
        this.flag = 1;
        TextView tv3 = (TextView) findViewById(2131492946);
        tv3.setText("");
        TextView tv = (TextView) findViewById(2131492944);
        TextView tv2 = (TextView) findViewById(2131492945);
        this.m = 0;
        Random rm = new Random();
        this.n = rm.nextInt(3);
        String[] ss = {"CPU: Paper", "CPU: Rock", "CPU: Scissors"};
        tv2.setText(ss[this.n]);
        if (v == this.P) {
            tv.setText("YOU: Paper");
            this.m = 0;
        }
        if (v == this.r) {
            tv.setText("YOU: Rock");
            this.m = 1;
        }
        if (v == this.S) {
            tv.setText("YOU: Scissors");
            this.m = 2;
        }
        this.handler.postDelayed(this.showMessageTask, 1000L);
    }
}
```

Here we see that each time we click our choice button (I didn't run the application, so I don't have a visual representation of them), we generate a random number between 0 up to but not including 3. This number is used to set the Computer "hand" and translate numerically our hand (as `this.m`).

We now understand that the `v` is our runtime choice, passed as a "view" (from what I understand, we literally receive which button was pressed), which is then compared against the Paper (`this.P`), Rock (`this.r`) and Scissors (`this.S`), defaulting to Paper if for any case it breaks. At the end, we call after a delay, the flag generating function `showMessageTask`.

Back again to the `showMessageTask` function, we can now understand when is `cnt` incremented.

Whether we win or not is determined by checking if our "hand" is greater that the computer's "hand" by exactly one, with the last `else` being the default case for when checking for `Paper (0) < Scissors (2)`.

This means that `Paper (0, CPU)` vs `Rock (1, us)` will skip over the first check, since `0 - 1` results in `-1`. It will then go to the next block, which checks it backwards, as `1 - 0` and checks if it is exactly `1` (the CPU has advantage over us). This is true for our condition, so we execute the block and get a `LOSE`, and a reset to 0.

Another example would be `Scissors (2, CPU)` vs `Paper (0, us)`, which skips the first `if` block (`2 - 0 == 2`), the second and third if blocks (`0 - 2 == -2, m != n`) and enters the fourth block since the condition `m < n` is true. This gets us another `LOSE` and `cnt` reset.

At this point we can ponder on how to get `cnt` to equal `1000` (or, `999` if we win sequentially).

Some of the solutions that can be used are illustrated here (non-exhaustively):
 - Get the initial PRNG seed for `java.util.Random` to predict all future generated numbers.
 - Patch the application to always print the flag and skip all checks.
 - Be lucky enough to win a thousand times.
 - Skip the hassle and directly derive the flag from the decompiled code.

I'll continue with the static analysis approach to get the flag.

## Analyzing the libraries

What about the `lib/` directory we saw earlier? I said before that it contained interesting stuff to solve the challenge, and it does, so let's jump in.

The directory contains a bunch of `.so` files for each possible architecture the application might run on:
```bash
lib
├── armeabi
│   └── libcalc.so
├── armeabi-v7a
│   └── libcalc.so
├── mips
│   └── libcalc.so
└── x86
    └── libcalc.so
```

Let's check out the `x86` version, which I'm most comfortable with.
I'll use `radare2` for this task. I'm not going to explain using `radare2` for sake of brevity and if you want to learn more about it, there's the excellent `radare2` book to start with.

Doing an `afl` (analyze functions and list them) command returns this:
```
[0x00000340]> afl
0x00000330    1      6 sym.imp.__cxa_finalize
0x00000310    1      6 sym.imp.__cxa_atexit
0x00000320    1      6 sym.imp.__stack_chk_fail
0x00000340    1     36 entry0
0x000003f0    1      4 fcn.000003f0
0x00000400    1      6 sym.Java_com_example_seccon2015_rock_1paper_1scissors_MainActivity_calc
```

From previous binary exploitation experiences I know that the `sym.imp.__<something>` functions are `C Stdlib` imported functions that do some checking and deferred execution. These aren't relevant for this challenge.

We can also check the `entry0` function for the library entrypoint, but both `entry0` and the `fcn.000003f0` function are irrelevant.
The important function is the `sym.Java_com_example_seccon2015_rock_1paper_1scissors_MainActivity_calc` function, which provides the implementation for the
`calc()` function we saw earlier in the Java code.

The `calc` function's assembly looks like this:
```asm
mov eax, 7
ret
```

This is quite simple actually, since the `x86` architecture mandates that function return values are returned in the `eax` CPU register (registers are small-sized variables near the CPU that are used during execution to keep track of values, pointers and structures). This function returns the number `7` each time it is called, has no local variables and doesn't use any arguments it might receive.

The other architecture libraries provide similar implementations for the same return-7-when-called behaviour, so I'm skipping over them too.

This concludes the application's analysis since we got the final jigsaw puzzle piece to get the flag.

## Solution

This is the flag-printing block of code:
```java
if (1000 == MainActivity.this.cnt) {
    tv3.setText("SECCON{" + String.valueOf((MainActivity.this.cnt + MainActivity.this.calc()) * 107) + "}");
}
```

Let's take a deeper look at it. The flag is made by concatenating `"SECCON{"` with the string representation of `(cnt + calc()) * 107` and an ending `"}"`.

We know that at that point, `cnt` is always `1000` and `calc()` always returns `7`, so the expression evaluates to `107749`, and our flag is:
`SECCON{107749}`.

This challenge is solved! It was quite interesting to analyze it and learn more about Android application development.
