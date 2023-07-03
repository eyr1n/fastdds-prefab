```sh
mkdir -p build
pushd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=~/Library/Android/sdk/ndk/25.1.8937393/build/cmake/android.toolchain.cmake
cmake --build . --target install
popd

mkdir -p maven/jp/eyrin/fastdds-prefab/1.0.0
cp build/fastdds-prefab-1.0.0.aar maven/jp/eyrin/fastdds-prefab/1.0.0/
cp build/fastdds-prefab-1.0.0.pom maven/jp/eyrin/fastdds-prefab/1.0.0/
```
