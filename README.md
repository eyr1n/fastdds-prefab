```sh
cmake -DCMAKE_TOOLCHAIN_FILE=<path-to-android-ndk>/build/cmake/android.toolchain.cmake -B build -S .
cmake --build build --target install
```
