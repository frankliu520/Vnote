# 1. pciutils-maste嵌入式移植
# 2. 概述
在嵌入式系统中查看pcies设备使用lspci命令一般被阉割，在程序调试的时候不能看到更多详细的信息。若需要使用完整指令则需要重新编译替换。
# 3. 准备工作
## 3.1. 系统交叉编译工具
编译之前确保主机的交叉编译工具已经存在和交叉环境已经ok
## 3.2. pciutils源码
根据链接下载[源码链接](https://github.com/pciutils/pciutils)拷贝到主机中其目录截图如下
![目录树](_v_images/20251118172613481_16550.png)
# 4. 编译
根据主机环境配置交叉编译脚本
我的环境脚本内容如下：
```
#!/bin/bash
set -e

# ---------- 交叉环境 ----------
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-
export PATH="/home/liudl/toolchain/bin:$PATH"

# ---------- 源码目录 ----------
cd /home/liudl/Project/pciutils-master

# ---------- 检查 Makefile 是否存在 ----------
if [ ! -f "Makefile" ]; then
    echo "错误: Makefile 不存在!"
    exit 1
fi

# ---------- 创建备份 ----------
cp Makefile Makefile.backup

# ---------- 配置交叉 + 共享库 ----------
sed -i \
  -e "s|^CROSS_COMPILE.*|CROSS_COMPILE = $CROSS_COMPILE|" \
  -e 's|^HOST.*|HOST = aarch64-linux|' \
  -e 's|^ZLIB.*|ZLIB = no|' \
  -e 's|^DNS.*|DNS = no|' \
  -e 's|^SHARED.*|SHARED = yes|' \
  Makefile

# ---------- 检查修改结果 ----------
echo "Makefile 修改后的相关配置:"
grep -E "^(CROSS_COMPILE|HOST|ZLIB|DNS|SHARED)" Makefile

# ---------- 编译 ----------
echo "开始编译..."
make clean
make -j$(nproc)

# ---------- 检查生成的文件 ----------
echo "------ 编译完成 ------"
echo "可执行文件:"
file lspci setpci
ls -lh lspci setpci

echo "库文件:"
find . -name "*.so*" -type f 2>/dev/null
find . -name "*.a" -type f 2>/dev/null
```
运行编译脚本 ./run,sh
若脚本运行出现
```
 ./build.sh
-bash: ./build.sh: /bin/bash^M: bad interpreter: Text file busy
```
是因为Windows 行尾符导致的问题，把脚本转成 Unix 格式即可

```
dos2unix build.sh 
```
# 5. 拷贝运行
将目录下的lspci文件和setpci 文件拷贝到 /usr/bin/目录下同时给执行权限
```
# 1 拷贝
cp lspci setpci /usr/bin/

# 2 给执行权限
chmod 755 /usr/bin/lspci /usr/bin/setpci

# 3 验证
lspci -nn
```

