# Custom CPython Build: 0**0 = NaN


[![Python Version](https://img.shields.io/badge/Python-3.11-3776AB?style=flat&logo=python&logoColor=white)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/Platform-macOS-000000?style=flat&logo=apple&logoColor=white)](#)
[![Feature](https://img.shields.io/badge/0**0-NaN-FF5733?style=flat)](#)
[![Build Status](https://img.shields.io/badge/CPython%20Build-Custom-blueviolet?style=flat)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat)](https://opensource.org/licenses/MIT)


This guide provides step-by-step instructions to download, modify, and compile a custom version of Python on macOS. By default, Python evaluates `0**0` to `1`. These modifications change the behavior so that `0**0` and `0.0**0.0` both evaluate to `NaN` (Not a Number).

## Step 1: Install macOS Build Dependencies

You need Command Line Tools and specific C libraries to compile Python. We use Homebrew to install these without requiring admin (sudo) privileges.

1. Open your Terminal.
2. Install Apple's compiler tools (if not already installed):
   `xcode-select --install`
3. Update Homebrew:
   `brew update`
4. Install the required dependencies:
   `brew install pkg-config openssl@3 readline sqlite3 xz zlib tcl-tk bzip2 libffi`

## Step 2: Clone the CPython Source Code

Download the official Python source code to your home directory. We will use Python version 3.11 for this guide.

```bash
cd ~
git clone https://github.com/python/cpython.git
cd cpython
git checkout 3.11
```

## Step 3: Modify the Integer Power Logic

We need to intercept the math logic for integers before it defaults to returning 1.

1. Open the file Objects/longobject.c in a text editor.
2. Search for the function named long_pow.
3. Locate the if/else block that parses the x (modulo) argument. It ends with Py_RETURN_NOTIMPLEMENTED;.

4. Insert the new check exactly below that block, before the negative exponent check.

Change the code to look exactly like this:

```c
else if (x == Py_None)
        c = NULL;
    else {
        Py_DECREF(a);
        Py_DECREF(b);
        Py_RETURN_NOTIMPLEMENTED;
    }

    /* Check if base and exponent are both 0 */

    if (Py_SIZE(a) == 0 && Py_SIZE(b) == 0) {
        Py_DECREF(a);
        Py_DECREF(b);
        Py_XDECREF(c);
        return PyFloat_FromDouble(Py_NAN);
    }

    if (Py_SIZE(b) < 0 && c == NULL) {
```

## Step 4: Modify the Float Power Logic

Next, we handle floating-point numbers.

1. Open the file Objects/floatobject.c in a text editor.
2. Search for the function named float_pow.
3. Find the CONVERT_TO_DOUBLE macros and insert the new check immediately below them.

Change the code to look exactly like this:

```
    CONVERT_TO_DOUBLE(v, iv);
    CONVERT_TO_DOUBLE(w, iw);

    /* Check if base and exponent are both 0.0 */
    
    if (iv == 0.0 && iw == 0.0) {
        return PyFloat_FromDouble(Py_NAN);
    }

    /* Sort out special cases here instead of relying on pow() */
    if (iw == 0) {              /* v**0 is 1, even 0**0 */
        return PyFloat_FromDouble(1.0);
    }
```


## Step 5: Configure and Compile Python

Because we are on macOS, we must explicitly tell the compiler where Homebrew installed the required libraries. Run these commands one by one in your terminal from inside the cpython directory.

1. Set the environment variables:

```bash
export LDFLAGS="-L$(brew --prefix zlib)/lib -L$(brew --prefix sqlite3)/lib"
export CPPFLAGS="-I$(brew --prefix zlib)/include -I$(brew --prefix sqlite3)/include"
export PKG_CONFIG_PATH="$(brew --prefix openssl@3)/lib/pkgconfig:$(brew --prefix sqlite3)/lib/pkgconfig:$(brew --prefix zlib)/lib/pkgconfig"
```

2. Configure the build. This sets the installation path to a custom folder in your home directory so it does not interfere with your system Python:

```bash
./configure --prefix=$HOME/custom_python_build \
            --with-openssl=$(brew --prefix openssl@3) \
            --enable-optimizations
```
3. Compile the code using 4 CPU cores to speed up the process:

```bash
make -j 4
```

4. Install the compiled binaries to your custom folder:

```bash
make install
```

## Step 6: Create a Virtual Environment and Test
Your custom Python interpreter is now installed in ~/custom_python_build. You can use it to create isolated virtual environments.

1. Create a new virtual environment:

```bash
~/custom_python_build/bin/python3 -m venv ~/nan_venv
```

2. Activate the virtual environment:

```bash
source ~/nan_venv/bin/activate
```

3. Now, verify that the interpreter works correctly by running a test script.

```bash
python test.py
```