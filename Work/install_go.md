```bash
vim install_go.sh
```



```bash
#!/bin/bash

# 检查参数
if [ $# -eq 0 ]; then
    echo "Usage: $0 <go_version>"
    echo "Example: $0 1.25.1"
    exit 1
fi

GO_VERSION=$1
DOWNLOAD_URL="https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
INSTALL_DIR="/data/lib/go${GO_VERSION}"
TEMP_FILE="/tmp/go${GO_VERSION}.linux-amd64.tar.gz"

echo "正在下载 Go ${GO_VERSION}..."
echo "下载地址: ${DOWNLOAD_URL}"
echo "安装目录: ${INSTALL_DIR}"

# 创建安装目录
echo "创建安装目录..."
sudo mkdir -p "${INSTALL_DIR}"

# 下载 Go
echo "开始下载..."
if ! wget -O "${TEMP_FILE}" "${DOWNLOAD_URL}"; then
    echo "错误: 下载失败"
    exit 1
fi

echo "下载完成，正在解压..."

# 解压到指定目录
if ! sudo tar -xzf "${TEMP_FILE}" -C "${INSTALL_DIR}" --strip-components=1; then
    echo "错误: 解压失败"
    rm -f "${TEMP_FILE}"
    exit 1
fi

# 清理临时文件
echo "清理临时文件..."
rm -f "${TEMP_FILE}"

echo "安装完成！"
echo "Go ${GO_VERSION} 已安装到: ${INSTALL_DIR}"
echo ""
echo "要使用这个版本的 Go，请将以下路径添加到你的 PATH 环境变量中:"
echo "export PATH=${INSTALL_DIR}/bin:\$PATH"
echo ""
echo "或者创建符号链接:"
echo "sudo ln -sf ${INSTALL_DIR}/bin/go /usr/local/bin/go"
echo "sudo ln -sf ${INSTALL_DIR}/bin/gofmt /usr/local/bin/gofmt"
```



```bash
chmod +x ./install_go.sh
./install_go.sh 1.25.1
```

