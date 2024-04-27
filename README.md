```sh
mkdir -p build
pushd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=~/Library/Android/sdk/ndk/25.1.8937393/build/cmake/android.toolchain.cmake
cmake --build . --target install
popd

mkdir -p maven/jp/eyrin/fastdds-prefab/2.13.4
cp build/fastdds-prefab-2.13.4.aar maven/jp/eyrin/fastdds-prefab/2.13.4/
cp build/fastdds-prefab-2.13.4.pom maven/jp/eyrin/fastdds-prefab/2.13.4/
```
