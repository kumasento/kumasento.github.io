---
title: 'glog, gflags, and C++ ABI'
date: 2020-06-12
comments: true
---
I recently got back to one of my previous project, which uses the two excellent Google-developed libraries: `glog` and `gflags`. My project used to work well, but since the time I've been away is long, and the system has been upgraded and modified by others somehow, it unfortunately cannot be successfully built, and the major problem lied in `glog` and `gflags`. The error message is as follows:

```bash
undefined reference to `google::base::CheckOpMessageBuilder::NewString[abi:cxx11]()'
```

Luckily, I have fixed this problem in the end and realized that the key problem was related to the C++ ABI, but the whole progress of locating and resolving the problem was exhaustive and tedious. Therefore, I would like to share my experience and thoughts on this problem, in case others suffer from the same trap.

# Setup

The test code -

```cpp
#include <glog/logging.h>

int main(int argc, char *argv[]) {
  google::InitGoogleLogging(argv[0]);

  CHECK_EQ(1, 2) << "1 doesn't equal to 2";

  return 0;
}
```

The compiler I use is GCC 7.3.0. `cmake` should be available as well.

To compile the previous code snippet, I've downloaded `glog==0.4.0` and `gflags==2.2.2`, both from their corresponding GitHub repositories.

[gflags/gflags](https://github.com/gflags/gflags/releases/tag/v2.2.2)

[google/glog](https://github.com/google/glog/releases/tag/v0.4.0)

Here are the commends that I used to compile these two libraries:

```bash
# compile gflags
wget https://github.com/gflags/gflags/archive/v2.2.2.tar.gz
tar xvf v2.2.0.tar.gz
cd gflags-2.2.2 && mkdir build && cd build 
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/.local -DBUILD_SHARED_LIBS=ON -DCMAKE_CXX_FLAGS="-std=c++11 -D_GLIBCXX_USE_CXX11_ABI=1"
make install
# compile glog
wget https://github.com/google/glog/archive/v0.4.0.tar.gz
tar xvf v0.4.0.tar.gz
cd glog-0.4.0 && mkdir build cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/.local -DBUILD_SHARED_LIBS=ON -DCMAKE_CXX_FLAGS="-std=c++11 -D_GLIBCXX_USE_CXX11_ABI=1"
make install
```

The `CMAKE_INSTALL_PREFIX` can be anywhere you prefer. Please note that there is a `-D_GLIBCXX_USE_CXX11_ABI=1` in `-DCMAKE_CXX_FLAGS` parameter of the `cmake` command. We will get back to this later.

Once you've successfully compiled them, you can create a file `[main.cc](http://main.cc)` that includes the test code above, and then compile it by

```bash
g++ main.cc -o main -I$HOME/.local/include -L$HOME/.local/lib -lglog -lgflags
```

And you will find the following error messages:

```bash
/tmp/ccoKxaIW.o: In function `std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >* google::MakeCheckOpString<int, int>(int const&, int const&, char const*)':
main.cc:(.text._ZN6google17MakeCheckOpStringIiiEEPNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEERKT_RKT0_PKc[_ZN6google17MakeCheckOpStringIiiEEPNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEERKT_RKT0_PKc]+0x6c): undefined reference to `google::base::CheckOpMessageBuilder::NewString[abi:cxx11]()'
$HOME/.local/lib/libglog.so: undefined reference to `google::FlagRegisterer::FlagRegisterer<std::string>(char const*, char const*, char const*, std::string*, std::string*)'
collect2: error: ld returned 1 exit status
```

Basically, you cannot find references to:

1. `google::base::CheckOpMessageBuilder::NewString[abi:cxx11]()`; 

2. `google::FlagRegisterer::FlagRegisterer<std::string>(...)`; 

Now it is our turn to find out what the actual problem is.

# Two types of string

The error messages say we cannot find references to some **symbols**, but is this really the case? We can use `nm` to find it out.

```bash
nm $HOME/.local/lib/libgflags.so | grep FlagRegisterer | c++filt
```

This command looks up `[libgflags.so](http://libgflags.so)` and tries to find whether there exists `FlagRegisterer`. Its output is as follows:

```bash
0000000000026406 W google::FlagRegisterer::FlagRegisterer<bool>(char const*, char const*, char const*, bool*, bool*)
00000000000266e0 W google::FlagRegisterer::FlagRegisterer<double>(char const*, char const*, char const*, double*, double*)
0000000000026498 W google::FlagRegisterer::FlagRegisterer<int>(char const*, char const*, char const*, int*, int*)
000000000002652a W google::FlagRegisterer::FlagRegisterer<unsigned int>(char const*, char const*, char const*, unsigned int*, unsigned int*)
00000000000265bc W google::FlagRegisterer::FlagRegisterer<long>(char const*, char const*, char const*, long*, long*)
000000000002664e W google::FlagRegisterer::FlagRegisterer<unsigned long>(char const*, char const*, char const*, unsigned long*, unsigned long*)
0000000000026772 W google::FlagRegisterer::FlagRegisterer<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >(char const*, char const*, char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)
0000000000026406 W google::FlagRegisterer::FlagRegisterer<bool>(char const*, char const*, char const*, bool*, bool*)
00000000000266e0 W google::FlagRegisterer::FlagRegisterer<double>(char const*, char const*, char const*, double*, double*)
0000000000026498 W google::FlagRegisterer::FlagRegisterer<int>(char const*, char const*, char const*, int*, int*)
000000000002652a W google::FlagRegisterer::FlagRegisterer<unsigned int>(char const*, char const*, char const*, unsigned int*, unsigned int*)
00000000000265bc W google::FlagRegisterer::FlagRegisterer<long>(char const*, char const*, char const*, long*, long*)
000000000002664e W google::FlagRegisterer::FlagRegisterer<unsigned long>(char const*, char const*, char const*, unsigned long*, unsigned long*)
0000000000026772 W google::FlagRegisterer::FlagRegisterer<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >(char const*, char const*, char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)
```

So `[libgflags.so](http://libgflags.so)` has `FlagRegisterer`, while it is not specialized for type `std::string`. Such that we cannot find a reference to `FlagRegisterer<std::string>`. 

There is a specialization quite close to our need:

```bash
0000000000026772 W google::FlagRegisterer::FlagRegisterer<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >(char const*, char const*, char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)
```

It is clear that this is a C++ ABI problem. `libglog.so`, which the reference problems originate from, wants a `string` type that is not C++11 ABI compatible. Since we compile `gflags` with `-D_GLIBCXX_USE_CXX11_ABI=1`, which is the default setting for GCC ≥ 5.1 ([source](https://gcc.gnu.org/onlinedocs/gcc-5.2.0/libstdc++/manual/manual/using_dual_abi.html)), `FlagRegisterer` is only specialized for `string` under namespace `std::__cxx11`.

We can compile again the `gflags` library with `-D_GLIBCXX_USE_CXX11_ABI=0`. This time, we can get `FlagRegisterer` related symbols as follows:

```bash
0000000000022ac6 W google::FlagRegisterer::FlagRegisterer<std::basic_string<char, std::char_traits<char>, std::allocator<char> > >(char const*, char const*, char const*, std::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::basic_string<char, std::char_traits<char>, std::allocator<char> >*)
```

See? There is no `std::__cxx11` in the template argument.

# Mystery glog

You may come up with a question: since you've compiled both `glog` and `gflags` in with C++11 ABI, why the problem still exists? In another word, shouldn't `glog` search for `std::__cxx11::string` instead?

I'm afraid I've got no answer for that. And I don't think I can figure out an answer in short time. If anyone can show a hint I will be very happy 😃

Anyway, it seems that we should recompile both `glog` and `gflags` with `-D_GLIBCXX_USE_CXX11_ABI=0`. 

# Summary

The final solution to my problem is that, I should compile both `glog` an `gflags` without C++11 ABI, as well as the application that linked to them:

```bash
g++ main.cc -o main -I$HOME/.local/include -L$HOME/.local/lib -lglog -lgflags -D_GLIBCXX_USE_CXX11_ABI=0
```

The solution looks simple, but the path to find it was not that easy. The final takeaway might be, keep that C++ ABI problem in mind, and when you cannot correctly get a reference to some symbol, use `nm` to look into the library file.

Happy coding!

# References

[Converting std::__cxx11::string to std::string](https://stackoverflow.com/questions/33394934/converting-std-cxx11string-to-stdstring)

[Dual ABI](https://gcc.gnu.org/onlinedocs/gcc-5.2.0/libstdc++/manual/manual/using_dual_abi.html)

[Cxx11AbiCompatibility - GCC Wiki](https://gcc.gnu.org/wiki/Cxx11AbiCompatibility)

[undefined reference to 'FlagRegisterer::FlagRegisterer' · Issue #203 · gflags/gflags](https://github.com/gflags/gflags/issues/203#issuecomment-351559225)
