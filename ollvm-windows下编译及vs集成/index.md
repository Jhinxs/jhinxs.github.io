# Ollvm Windows下编译及VS集成


之前用的Ollvm基于LLVM4的实在在是太老了，看到了几个新一点的项目就拿来编译瞅瞅，在Windows下编译的过程过于坎坷，一共编译成功了两个项目，这里分别记录一下，以供查查    
https://github.com/bluesadi/Pluto-Obfuscator  
https://github.com/lemon4ex/obfuscator/tree/llvm-13.0  
*注：  
（1）都需要安装ninja编译工具：https://github.com/ninja-build/ninja/releases/tag/v1.10.2 （ninja编译速度快）  
（2）要在VS中安装Clang编译集成工具  
[<img src="/images/assets/post5/vsclang.PNG" width="100%"/>](/images/assets/post5/vsclang.PNG)  

### Ollvm13  
项目地址：https://github.com/lemon4ex/obfuscator/tree/llvm-13.0  

这个项目编译的过程:  
#### 编译命令  
```sh
进入VS开发者命令行建议管理员，建议关闭杀软
G:\VisualStudio2019\VisualStudio2019\VC\Auxiliary\Build\vcvarsall.bat x64
cd obfuscator-llvm-13.0
mkdir build
cd build
cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_NEW_PASS_MANAGER=OFF -DLLVM_INCLUDE_TESTS=OFF ..
ninja
编译完后再bin目录下看到生成的clang等exe表示成功，编译过程中产生的warning可忽略
```
#### 混淆命令
```sh
控制流扁平化
这个模式主要是把一些if-else语句，嵌套成do-while语句

-mllvm -fla：激活控制流扁平化
-mllvm -split：激活基本块分割。在一起使用时改善展平。
-mllvm -split_num=3：如果激活了传递，则在每个基本块上应用3次。默认值：1

指令替换
这个模式主要用功能上等效但更复杂的指令序列替换标准二元运算符(+ , – , & , | 和 ^)

-mllvm -sub：激活指令替换
-mllvm -sub_loop=3：如果激活了传递，则在函数上应用3次。默认值：1

虚假控制流程
这个模式主要嵌套几层判断逻辑，一个简单的运算都会在外面包几层if-else，所以这个模式加上编译速度会慢很多因为要做几层假的逻辑包裹真正有用的代码。

另外说一下这个模式编译的时候要浪费相当长时间包哪几层不是闹得！

-mllvm -bcf：激活虚假控制流程
-mllvm -bcf_loop=3：如果激活了传递，则在函数上应用3次。默认值：1
-mllvm -bcf_prob=40：如果激活了传递，基本块将以40％的概率进行模糊处理。默认值：30

#####  https://heroims.github.io/2019/01/06/OLLVM%E4%BB%A3%E7%A0%81%E6%B7%B7%E6%B7%86%E7%A7%BB%E6%A4%8D%E4%B8%8E%E4%BD%BF%E7%94%A8/
```
#### 错误处理
编译过程中可能会出的错误以及处理方法：  

1.编译中出现：修复常量中有换行符错误类似字眼
将文件编码改为utf-8-BOM， 使用 vscode 打开文件，选择“编码保存为”->"UTF8-8 BOM 编码“
基本上是这几个文件
\clang-tools-extra\clangd\Diagnostics.cpp(491): error C2001: 常量中有换行符
\clang-tools-extra\clangd\Hover.cpp(578): error C2001: 常量中有换行符
\clang-tools-extra\clangd\CodeComplete.h(81): error C2001: 常量中有换行符
\clang-tools-extra\clangd\Selection.cpp : error C2001: 常量中有换行符

2.clang编译时没问题，但是编译C++二进制文件后发现混淆不生效
cmake的时候加 -DLLVM_ENABLE_NEW_PASS_MANAGER=OFF ，这个是llvm 13.x版本的特性问题

3.lld-link.exe未找到
复制到vs的llvmlld-link.exe到Ollvm编译完的bin目录下，于clang同目录
G:\VisualStudio2019\VisualStudio2019\VC\Tools\Llvm\x64\bin\lld-link.exe  
### Pluto-Obfuscator
项目地址：https://github.com/bluesadi/Pluto-Obfuscator  
#### 编译命令
```sh
进入VS开发者命令行建议管理员，建议关闭杀软
G:\VisualStudio2019\VisualStudio2019\VC\Auxiliary\Build\vcvarsall.bat x64
cd Pluto-Obfuscator
mkdir build
cd build
cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang" -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD="X86" -DBUILD_SHARED_LIBS=off -DCMAKE_INSTALL_PREFIX="../install" ../llvm
ninja
编译完后再bin目录下看到生成的clang等exe表示成功，编译过程中产生的warning可忽略
```
#### 错误处理
1."Unknown endianness of the compilation platform, check this header aes_encrypt.h"  
打开Obfuscation/CryptoUtils.h文件,并为修改如下，之前应该是没有_WIN32宏定义：
```c++
#if defined(__i386) || defined(__i386__) || defined(_M_IX86) ||                \
    defined(INTEL_CC) || defined(_WIN64) || defined(_WIN32)
```
2.错误类似："FAILED: xxxxxx.cpp, warning C4819: 该文件包含不能
在当前代码页(936)中表示的字符。请将该文件保存为 Unicode 格式以防止数据丢失,error C3861: “alterVal”: 找不到标识符"  
这个错误因为这些文件中存在中文注释，格式有误导致，要么所有的文件修改格式为Unicode，要么删除苏所有文件中的中文注释。但是都很麻烦，文件太多了。这里可以打开build文件下的build.ninja文件，用vscode或者其他文件编辑工具，搜索并替换，加入/utf-8的编译选项，例如搜索 /DNDEBUG 并替换为
/DNDEBUG /utf-8。我对这个编译不熟，可能有其他的更方便的办法。。。。不过这个方法有效！
#### 混淆命令
```sh
-mllvm -mba -mllvm -mba-prob=50 -mllvm -fla-ex -mllvm -gle
```
或者参考项目地址的说明

### 总结
放到IDA里的效果来看的话，Pluto-Obfuscator效果更符合本人使用需求，并且一直在更新，可以试试。
