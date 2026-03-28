# run_with_snapshot.sh

自动为命令中引用的脚本、配置等文件创建带标签的快照副本，再执行原命令。适用于需要频繁调整配置、并行运行多个实验实例的场景。

## 功能简介

- 自动扫描命令参数，识别其中真实存在的文件
- 将识别到的文件复制为带时间戳（或自定义标签）的快照副本
- 用副本路径替换原命令中的文件参数后执行
- 同一文件在命令中多次出现时，只复制一份
- 支持以空格形式分隔的参数与 `--option=file.yaml` 形式的参数

**示例：**
```bash
$ snap -c exp01 -f config.yaml python eval.py --config config.yaml --gpu 0
Snapshot: config.yaml -> config_exp01.yaml
# 实际执行: python eval.py --config config_exp01.yaml --gpu 0
```

## 脚本

```bash
#!/bin/bash
# run_with_snapshot.sh

set -euo pipefail

# 解析可选的 -c 和 -f 参数
TAG=""
EXPLICIT_FILES=()

while [[ $# -gt 0 && ("$1" == "-c" || "$1" == "-f") ]]; do
    if [[ $# -lt 2 ]]; then
        echo "Error: $1 需要一个参数" >&2
        exit 1
    fi
    if [[ "$1" == "-c" ]]; then
        TAG="$2"
        shift 2
    elif [[ "$1" == "-f" ]]; then
        if [[ ! -f "$2" ]]; then
            echo "Error: -f 指定的文件不存在: $2" >&2
            exit 1
        fi
        EXPLICIT_FILES+=("$2")
        shift 2
    fi
done

# 检查是否提供了要执行的命令
if [[ $# -eq 0 ]]; then
    echo "Usage: $0 [-c <tag>] [-f <file> ...] <command> [args...]" >&2
    exit 1
fi

# 在最开始统一计算 SUFFIX，确保同一次调用的所有文件使用相同后缀
SUFFIX="${TAG:-$(date +%Y%m%d_%H%M%S)}"

# 快照单个文件，返回快照路径（通过全局变量 SNAPSHOT_RESULT）
declare -A file_map
SNAPSHOT_RESULT=""

snapshot_file() {
    local filepath="$1"
    if [[ -z "${file_map[$filepath]+x}" ]]; then
        local DIR BASE EXT NAME SNAPSHOT
        DIR=$(dirname "$filepath")
        BASE=$(basename "$filepath")
        EXT="${BASE##*.}"
        NAME="${BASE%.*}"
        if [[ "$EXT" == "$BASE" || -z "$NAME" ]]; then
            # 没有扩展名（如 Makefile）或隐藏文件（如 .env）
            SNAPSHOT="${DIR}/${BASE}_${SUFFIX}"
        else
            SNAPSHOT="${DIR}/${NAME}_${SUFFIX}.${EXT}"
        fi
        cp "$filepath" "$SNAPSHOT"
        file_map[$filepath]="$SNAPSHOT"
        echo "Snapshot: $filepath -> $SNAPSHOT" >&2
    fi
    SNAPSHOT_RESULT="${file_map[$filepath]}"
}

# 判断一个参数是否为需要快照的目标文件
is_snapshot_target() {
    local arg="$1"
    if [[ ${#EXPLICIT_FILES[@]} -gt 0 ]]; then
        for f in "${EXPLICIT_FILES[@]}"; do
            if [[ "$arg" == "$f" ]]; then
                return 0
            fi
        done
        return 1
    else
        # 自动模式：不以 - 开头、包含 . 或 /、且文件真实存在
        if [[ "$arg" != -* ]] && [[ "$arg" == *.* || "$arg" == */* ]] && [[ -f "$arg" ]]; then
            return 0
        fi
        return 1
    fi
}

# 遍历所有参数，找出需要替换的文件并复制
new_args=()
for arg in "$@"; do
    # 处理 --option=value 形式（仅当 prefix 以 - 开头时才视为选项赋值）
    if [[ "$arg" == -* && "$arg" == *=* ]]; then
        prefix="${arg%%=*}"
        value="${arg#*=}"
        if is_snapshot_target "$value"; then
            snapshot_file "$value"
            new_args+=("${prefix}=${SNAPSHOT_RESULT}")
        else
            new_args+=("$arg")
        fi
    elif is_snapshot_target "$arg"; then
        snapshot_file "$arg"
        new_args+=("$SNAPSHOT_RESULT")
    else
        new_args+=("$arg")
    fi
done

# 检查 -f 指定的文件是否全部在命令参数中被匹配到
if [[ ${#EXPLICIT_FILES[@]} -gt 0 ]]; then
    for f in "${EXPLICIT_FILES[@]}"; do
        if [[ -z "${file_map[$f]+x}" ]]; then
            echo "Warning: -f 指定的文件未出现在命令参数中: $f" >&2
        fi
    done
fi

exec "${new_args[@]}"
```

## 参数说明

| 参数 | 必填 | 说明 |
|------|------|------|
| `-c <tag>` | 否 | 自定义快照后缀，替代默认时间戳 |
| `-f <file>` | 否 | 显式指定要创建快照的文件（必须存在），可重复使用。启用后禁用自动识别 |
| `<command>` | 是 | 要执行的完整命令 |

`-c` 和 `-f` 须放在 `<command>` 之前，顺序可任意混排。

**文件识别模式：**

- **自动模式**（未使用 `-f`）：参数同时满足以下三个条件时被视为文件：
  1. 不以 `-` 开头
  2. 包含 `.` 或 `/`（看起来像文件路径）
  3. 在文件系统中真实存在

- **显式模式**（使用了 `-f`）：仅处理 `-f` 指定的文件，命令中的其他参数不做识别。如果 `-f` 指定的文件不存在，脚本会报错退出；如果 `-f` 指定的文件未出现在后续命令参数中，脚本会输出警告

两种模式均可识别 `--config=config.yaml` 形式的参数，自动拆分等号两侧，仅对值部分做快照替换。只有以 `-` 开头的参数才会按等号拆分，避免误处理非选项参数中的等号。

## 使用示例

### 自动模式

**基础用法：**
```bash
./run_with_snapshot.sh python -m eval --config config.yaml --gpu 0
# 生成: config_20240315_142301.yaml
```

**自定义标签（`-c`）：**
```bash
./run_with_snapshot.sh -c exp01 python -m eval --config config.yaml --gpu 0
# 生成: config_exp01.yaml
```

**`--option=value` 形式：**
```bash
./run_with_snapshot.sh -c exp01 python -m eval --config=config.yaml --gpu 0
# 生成: config_exp01.yaml，实际执行 --config=config_exp01.yaml
```

**多个文件自动识别：**
```bash
./run_with_snapshot.sh python -m eval --config config.yaml --extra extra.yaml
# config.yaml 和 extra.yaml 分别生成快照副本
```

> **提示：** 如果命令中包含 `eval.py` 等脚本路径参数，自动模式会将其一并快照。要精确控制快照范围，请使用 `-f` 显式模式。

### 显式模式（`-f`）

使用 `-f` 指定要快照的文件，命令中其他文件参数不受影响。

**只快照指定文件：**
```bash
snap -f config.yaml python eval.py --config config.yaml --vocab vocab.txt
# 只有 config.yaml 被快照，eval.py 和 vocab.txt 保持不变
```

**多个 `-f`：**
```bash
snap -f config.yaml -f extra.yaml python eval.py --config config.yaml --extra extra.yaml
```

**组合 `-c` 和 `-f`：**
```bash
snap -c exp01 -f config.yaml python eval.py --config config.yaml --vocab vocab.txt
```

**`-f` 配合 `--option=value`：**
```bash
snap -c exp01 -f config.yaml python eval.py --config=config.yaml --gpu 0
# 等号内的值也能匹配 -f 指定的文件
```

### 环境变量、重定向与并行

**传入环境变量：**
```bash
CUDA_VISIBLE_DEVICES=0 snap -f config.yaml python eval.py --config config.yaml

# 或 export 后调用
export CUDA_VISIBLE_DEVICES=0
snap -f config.yaml python eval.py --config config.yaml
```

**重定向与后台运行：**
```bash
snap -f config.yaml python eval.py --config config.yaml > output.log 2>&1 &
```

**并行运行多个实例：**
```bash
snap -c gpu0_coco -f config.yaml python eval.py --config config.yaml --gpu 0 &
snap -c gpu1_voc  -f config.yaml python eval.py --config config.yaml --gpu 1 &
```

> **注意：** 并行时建议使用 `-c` 指定不同标签，避免极小概率的时间戳碰撞。

## 限制

- **自动模式会识别命令中所有符合条件的文件参数。** 例如 `python eval.py --config config.yaml` 中，`eval.py` 也满足自动识别条件（不以 `-` 开头、包含 `.`、文件存在），会被一同快照。如果只想快照特定文件，请使用 `-f` 显式指定。
- **不支持命令内部的管道和 shell 语法。** 脚本最终通过 `exec` 直接执行命令，无法解析管道（`|`）、命令替换（`$()`）、逻辑连接符（`&&` `||`）等 shell 语法。如果需要管道，应将管道写在脚本外层，例如：`snap python eval.py --config config.yaml | tee log.txt`，此时 `snap` 只负责 `python eval.py --config config.yaml` 部分，管道由外层 shell 处理。
- **重定向和后台运行（`>` `2>&1` `&`）由调用者的 shell 处理**，不是脚本自身的能力，因此可以正常使用。
- **`-f` 参数与命令中的路径必须字符串完全一致。** `-f config.yaml` 不会匹配命令中的 `./config.yaml`，反之亦然。

## 安装与配置

### 安装脚本

将脚本保存到你的 PATH 目录下并赋予执行权限：

```bash
cp run_with_snapshot.sh ~/.local/bin/run_with_snapshot.sh
chmod +x ~/.local/bin/run_with_snapshot.sh
```

### 配置 alias

**Bash** — 编辑 `~/.bashrc`：

```bash
alias snap='~/.local/bin/run_with_snapshot.sh'
```

生效：
```bash
source ~/.bashrc
```

**Zsh** — 编辑 `~/.zshrc`：

```zsh
alias snap='~/.local/bin/run_with_snapshot.sh'
```

生效：
```zsh
source ~/.zshrc
```

配置完成后即可直接使用：

```bash
snap -f config.yaml python eval.py --config config.yaml --gpu 0
snap -c exp01 -f config.yaml python eval.py --config config.yaml --gpu 0
```
