+++
date = '2026-06-04T20:15:46+08:00'
draft = false
title = 'mkosi 打包逐字节一致 linux 镜像'

+++

>   该工作的上下文是为可信执行环境 TEE 从 OVMF、shim/grub、kernel 到 rootfs 信任链链路提供异机可复现的一致度量 measurement。下面的演示基于的安全假设是：从 OVMF 侧拉起容器启动后，便禁用 ssh 等远程访问实例的方式，让运行时的环境在度量下保持只读。侧信道攻击属于威胁模型范围内，本文不做讨论。
>
>   另外 mkosi 仅覆盖了下面实验涉及到的部分指令，整体来说我也不是很熟练使用该工具，故有错误请指正。

## 做最基本的工作

```bash
sudo apt-get install -y python3-venv
python3 -m venv ~/mkosi-venv
. ~/mkosi-venv/bin/activate
pip install git+https://github.com/systemd/mkosi.git@v26
mkosi --version
```

建一个 mkosi 构建镜像的文件夹 `mkosi-workspace`。最初的构建配置仅涵盖用哪个发行版（Ubuntu noble 即 Ubuntu 24.04，由 mkosi 指定仓库）。`[Content]` 只装 `systemd`（当 PID 1。mkosi 有意不含 init，需要用户指定。我选择 systemd。反过来说，镜像本身含 Essential，不需要显式指定）。`[Output]` 的 `Format=directory` 让产物是目录树而非镜像文件。

toolstree 构建工具链先借 host 的 apt 包完成镜像构建（不指定 ToolsTree 时的默认行为），以及镜像内不含 kernel。

```bash
# ~/mkosi-workspace
cat > mkosi.conf << 'EOF'

[Distribution]
Distribution=ubuntu
Release=noble
Architecture=x86-64

[Content]
Packages=
  systemd

[Output]
Format=directory
Output=outputimage
EOF
```

然后 build，得到 `outputimage/`：

```bash
. ~/mkosi-venv/bin/activate
mkosi build
```

## 锁每个可能变化的环节

目前仓库版本、时间戳、machine-id、构建工具版本都没锁，换台机器或隔几天再 build，镜像的字节会发生变化。我们的目的是做到构建镜像的时空一致性：不分环境，不分时间。

随着时间推移的，首先是 mkosi 锚定的镜像仓库。这一步影响 `[Content] - Packages` 下以及 Essential 的包。这个很好解决，mkosi.conf 可以配置 Snapshot 字段，锁仓库版本。

另外构建时文件的修改时间 mtime 锁定在 1970-01-01，通过 `[Output] - SourceDateEpoch=0` 指定。 `[Output] - MachineId` 我不知道影不影响，不指定时默认 mkosi 往 `/etc/machine-id` 写固定字符串 `uninitialized`，可以实验打印。总之我定了。

build 两次做 diff，总有两个文件不一样：`/var/log/alternatives.log` 和 `/var/cache/ldconfig/aux-cache`。前者是 `update-alternatives` 维护链接时写的日志，每条带时间戳和 PID（见 [update-alternatives(1)](https://www.man7.org/linux/man-pages/man1/update-alternatives.1.html)）；后者是 `ldconfig` 的辅助缓存（见 [ldconfig(8)](https://man7.org/linux/man-pages/man8/ldconfig.8.html)）。装包过程会动这两个，时间戳和内容每次 build 都飘。它们都是运行时能重新生成的状态，删掉不影响功能，所以构建时用 `RemoveFiles=` 去掉。

更新后的 .conf：

```bash
[Distribution]
Distribution=ubuntu
Release=noble
Architecture=x86-64
Snapshot=20260430T230000Z   

[Content]
Packages=
  systemd

[Output]
Format=directory
Output=outputimage
SourceDateEpoch=0  
MachineId=ffffffffffffffffffffffffffffffff 
RemoveFiles=
  /var/log/alternatives.log
  /var/cache/ldconfig/aux-cache
```

目前按这个配置，在同一台 linux 下在不同时间段抹除缓存后编译出的两个镜像是逐字节一致的。目前有一个完整的 rootfs。没有 kernel。

## 引入常规服务 —— 以 http 服务为例

>   这里仅演示静态链接依赖的二进制文件服务。如果是动态链接，考虑到信任链仅覆盖 rootfs，需要动态的依赖包在 rootfs 下，需要注意这一点。

随便写一点 http 服务，在 8080 暴露一个接口 /hello。编译参数与 flag 如下：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -ldflags="-buildid=" -o go_service
```

`-trimpath` 去路径、`-buildid=` 去 build ID，为可复现。`CGO_ENABLED=0` 不考虑使用 cgo 以确保纯静态。

同机实验验 hash（实际跨机已可以（mac m 芯片上编译过），go 是交叉编译器。`GOOS`/`GOARCH` 指的是目标平台，跟 host OS 无关）：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags="-buildid=" -o g1
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags="-buildid=" -o g2
sha256sum g1 g2
```

准备好二进制，然后写 .service 放 overlay。`ExtraTrees=overlay` 的行为是：构建到某一步（我也不知道是哪一步，是 mkosi 官网这么写的）把 `overlay/` 这个目录的内容,按目录结构原样叠加（追加拷贝）到未完全处理完挂载的 rootfs 树上。叠加按照路径对路径——`overlay/usr/local/bin/go_service` 铺到 rootfs 的 `/usr/local/bin/go_service`，`overlay/usr/lib/systemd/system/go_service.service` 铺到 rootfs 的 `/usr/lib/systemd/system/go_service.service`。overlay 里的目录层级，就是它在 rootfs 里最终落的位置。

```bash
cd ~/mkosi-workspace

# 建叠加树目录,按 rootfs 根的布局摆
mkdir -p overlay/usr/local/bin
mkdir -p overlay/usr/lib/systemd/system
mkdir -p overlay/usr/lib/systemd/system/multi-user.target.wants

# 1. binary 放进去
cp ~/tdx-verity-exp/go_service overlay/usr/local/bin/go_service
chmod 755 overlay/usr/local/bin/go_service

# 2. service 文件(跟构建 2 同一个)
cat > overlay/usr/lib/systemd/system/go_service.service << 'EOF'
[Unit]
Description=attestation go_service
After=systemd-tmpfiles-setup.service

[Service]
ExecStart=/usr/local/bin/go_service
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 3. enable 软链(跟构建 2 同样,手动建好放进叠加树)
ln -sf ../go_service.service \
  overlay/usr/lib/systemd/system/multi-user.target.wants/go_service.service
```

## 等等，构建镜像包的工具链总得定下来吧？

「toolstree 构建工具链先借 host...」这是前文提到的。很可惜的是，虽然 mkosi 可以生成 toolstree 以不依赖 host 的情况相对可控的生成镜像，但 mkosi 没有提供 Toolstree 的可复现机制，或者说没有一个类似 ToolstreeSnapshot 的声明式字段。

实际也能理解，300+ 的编译用工具一个月内多少会有一些 CVE（Common Vulnerabilities and Exposures）要修。我写了个脚本实测在 4.30 到 6.3 一个月左右有 15/376 apt 包进行了版本变动。

一个可行的方案是镜像包里提供 toolstree pinned 版本，基于用户不信任 toolstree pinned 但信任上游最新包的原则，由用户 diff 上游 toolstree 的增量部分来断定是否可信。

首先生成来自 mkosi 指定的 Toolstree。

```bash
[Distribution]
Distribution=ubuntu
Release=noble
Architecture=x86-64
Snapshot=20260430T230000Z

[Content]
Packages=
  systemd
ExtraTrees=overlay

[Output]
Format=directory
Output=outputimage
SourceDateEpoch=0
MachineId=ffffffffffffffffffffffffffffffff
RemoveFiles=
  /var/log/alternatives.log
  /var/cache/ldconfig/aux-cache

[Build]
ToolsTree=default
ToolsTreeDistribution=ubuntu
ToolsTreeRelease=noble
```

得到 Toolstree 后（原名 `mkosi.tools`）改后缀为 `mkosi.toolstree.pinned.20260602T220000`。

然后构建时用的 conf：

```bash
[Distribution]
Distribution=ubuntu
Release=noble
Architecture=x86-64
Snapshot=20260430T230000Z

[Content]
Packages=
  systemd
ExtraTrees=overlay

[Output]
Format=directory
Output=outputimage
SourceDateEpoch=0
MachineId=ffffffffffffffffffffffffffffffff
RemoveFiles=
  /var/log/alternatives.log
  /var/cache/ldconfig/aux-cache

[Build]
ToolsTree=mkosi.tools.pinned.20260602T220000
```

## TEE 环境将信任链拓展至 rootfs——dm-verity

rootfs 是 ext4 文件系统，常规可读可写。dm-verity 用 erofs 文件系统（实际也可以用其他只写文件系统，这里仅 erofs 为例）将 rootfs 原本的目录树变为只读。信任链要求覆盖的目录树在运行时不发生修改，而实际遍历目录树判 hash 一致是计算成本困难的，所以很自然的想到用 merkle tree 做目录树的一致性校验，最终将 root hash 录入度量。dm-verity 也是这么做的，所以这是我使用 dm-verity 的原因。

首先将刚刚的镜像复制：

```bash
cd ~/outputimage-mkosi
sudo cp -a outputimage outputimage-rootfs
```

>   如果你在 conf 中指定了配置内核，这里需要将内核从 boot 挪走。
>
>   ```bash
>   sudo rm -rf outputimage-rootfs/boot/*
>   ```

之后打成只读 erofs 镜像。`-T 0` 把镜像里所有文件时间戳固定成 0，`-U` 固定镜像 UUID，这两个是为了可复现。参数顺序是「输出文件 输入目录」。

```bash
sudo mkfs.erofs -T 0 -U 11111111-2222-3333-4444-555555555555 -L outputimage_root \
  outputimage.erofs outputimage-rootfs/
ls -lh outputimage.erofs
```

>   如果 host 机没有 `mkfs.erofs`，安装这个：
>
>   ```bash
>   sudo apt install erofs-utils
>   ```

用 `veritysetup` 给 erofs 镜像建一棵 hash 树：把镜像按 4096 字节切块，每块算 sha256，层层往上聚合成一棵树，树根就是 root hash。`--uuid`、`--salt` 固定，`--root-hash-file` 把 root hash 写进文件。

```bash
sudo veritysetup format outputimage.erofs outputimage.verity \
  --uuid=66666666-7777-8888-9999-aaaaaaaaaaaa \
  --salt=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef \
  --root-hash-file=outputimage.roothash
cat outputimage.roothash; echo
```

实际到这一步，已经得到了核心的三文件：

```bash
outputimage.erofs      数据(只读 rootfs 镜像本身)
outputimage.verity     hash 树
outputimage.roothash    root hash(树根,锚定整个镜像)
```

`outputimage.erofs` 是数据、`outputimage.verity` 是 hash 树（两个独立文件）。这个 hash 将来进 cmdline 然后被度量（本文不展开 initrd/cmdline 的设计。或者后续再补）。下面有实验拆开步骤模拟之后 initrd/cmdline 自动化干的事情，让 AI 生成的。

```bash
# 生成 outputimage-verity-test 镜像并挂载到临时挂载点 /mnt/zs-test
cd ~/mkosi-workspace
sudo veritysetup open outputimage.erofs outputimage-verity-test outputimage.verity \
  "$(cat outputimage.roothash)"
sudo mkdir -p /mnt/zs-test
sudo mount -o ro /dev/mapper/outputimage-verity-test /mnt/zs-test

sudo sha256sum /mnt/zs-test/usr/local/bin/go_service
```

```bash
# 验证写被拒
echo test | sudo tee /mnt/zs-test/should-fail.txt
```

```bash
# 改一字节看 verity 如何拦截
cd ~/mkosi-workspace
sudo umount /mnt/zs-test
sudo veritysetup close outputimage-verity-test

# 改第 5000000 字节为 0xff
printf '\xff' | sudo dd of=outputimage.erofs bs=1 seek=5000000 count=1 conv=notrunc

# 同一个 root hash 重新打开挂载
sudo veritysetup open outputimage.erofs outputimage-verity-test outputimage.verity \
  "$(cat outputimage.roothash)"
sudo mount -o ro /dev/mapper/outputimage-verity-test /mnt/zs-test

# 读整棵树,强制扫到被改的块
sudo find /mnt/zs-test -type f -exec cat {} \; > /dev/null 2>/tmp/readerr2
sudo dmesg | grep -i verity | tail -3
cat /tmp/readerr2
```
