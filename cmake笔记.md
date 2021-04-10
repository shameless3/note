## CMake

CMakeLists.txt由命令，注释和空格组成

命令：

- cmake_minimum_required:指定运行此配置文件所需的 CMake 的最低版本
- project：该命令表示项目的名称 
- add_executable：将名为 [main.cc](http://main.cc/) 的源文件编译成一个名称为 Demo 的可执行文件
- aux_source_directory(<dir><variable>) ：将指定目录所有源文件成为一个变量供调用
- set：显示的定义变量

cmake后得到Makefile使用make命令即可得到Demo1

