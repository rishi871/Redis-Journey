# Redis & RediSearch on macOS (ARM64) — Working Notes

These are the steps/commands that worked for Rishi on macOS Apple Silicon to build, run, and debug Redis/RediSearch, plus terminal key bindings fixes.

---

## 0) System setup (Homebrew + core toolchain)
```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# Add brew to shell (Apple Silicon)
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
source ~/.zprofile

# Install required tools
brew update
brew install cmake llvm rust

# Prefer Homebrew LLVM (clang/clang++) in this shell
export PATH="/opt/homebrew/opt/llvm/bin:$PATH"
export LDFLAGS="-L/opt/homebrew/opt/llvm/lib"
export CPPFLAGS="-I/opt/homebrew/opt/llvm/include"
export CC=clang
export CXX=clang++

# Sanity checks
which cmake && cmake --version
which clang && clang --version
which cargo && cargo --version
```
> Notes: `cmake` fixes the immediate build error; `llvm` provides the expected LLVM Clang; `rust` provides `cargo` used by some deps.

---

## 1) Fresh, clean build of RediSearch
From the RediSearch source dir (e.g. `redis/modules/redisearch/src`):
```bash
make clean || true
rm -rf bin build CMakeFiles CMakeCache.txt

cmake . \
  -DCOORD_TYPE=oss \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_SHARED_LIBRARY_SUFFIX=.so \
  -UCMAKE_TOOLCHAIN_FILE \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DSVS_SHARED_LIB=OFF

make -j$(sysctl -n hw.logicalcpu)
```
**Expected output**: `src/bin/macos-aarch64-release/search-community/redisearch.so`

One‑liner (reset + build):
```bash
make clean || true && rm -rf bin build CMakeFiles CMakeCache.txt && \
cmake . -DCOORD_TYPE=oss -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_SHARED_LIBRARY_SUFFIX=.so \
-UCMAKE_TOOLCHAIN_FILE -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DSVS_SHARED_LIB=OFF && \
make -j$(sysctl -n hw.logicalcpu)
```

---

## 2) Run Redis (with or without RediSearch)
From Redis repo root (`redis/`):

**Redis only (no modules):**
```bash
src/redis-server
# Background mode
src/redis-server --daemonize yes
```
Quick check:
```bash
src/redis-cli ping   # -> PONG
```

**Redis with RediSearch:**
```bash
src/redis-server --loadmodule modules/redisearch/src/bin/macos-aarch64-release/search-community/redisearch.so
```
Verify module:
```bash
src/redis-cli
127.0.0.1:6379> MODULE LIST
```

---

## 3) iTerm2 key bindings (Fn+← / Fn+→ as Home/End)
**iTerm2 → Preferences → Profiles → Keys → Key Bindings**
- Add binding for **Fn+←** → *Send Hex Code* → `0x01` (Ctrl+A = beginning of line)
- Add binding for **Fn+→** → *Send Hex Code* → `0x05` (Ctrl+E = end of line)

Optional (word-jump via Option keys): set **Left/Right Option Key** = *Esc+* in the same Keys tab.

Zsh readline helpers (optional):
```bash
echo 'bindkey "^[[H" beginning-of-line' >> ~/.zshrc
echo 'bindkey "^[[F" end-of-line' >> ~/.zshrc
source ~/.zshrc
```

---

## 4) VS Code debugging (LLDB)

### 4.1 Install LLDB
```bash
xcode-select --install   # installs clang, lldb, make, etc.
# or
brew install llvm && echo 'export PATH="/opt/homebrew/opt/llvm/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
lldb --version
```

### 4.2 Build Redis with debug symbols
From repo root:
```bash
make distclean
make BUILD_WITH_DEBUG=yes CFLAGS="-O0 -g"
```

### 4.3 VS Code configs
Create **.vscode/tasks.json** at repo root:
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build",
      "type": "shell",
      "command": "make",
      "options": { "cwd": "${workspaceFolder}" },
      "problemMatcher": "$gcc",
      "group": { "kind": "build", "isDefault": true }
    }
  ]
}
```

Create **.vscode/launch.json**:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Redis Server",
      "type": "cppdbg",
      "request": "launch",
      "MIMode": "lldb",
      "program": "${workspaceFolder}/src/redis-server",
      "args": [],
      "cwd": "${workspaceFolder}",
      "externalConsole": true,
      "stopAtEntry": false,
      "preLaunchTask": "build"
    }
  ]
}
```
> If you see **“Could not find the task 'build'”**, confirm the task label matches and both files are under the same folder’s `.vscode/`. For multi‑root workspaces, keep task + launch in the same folder or reference by folder.

**Load RediSearch during debug (optional):** add to `args` in `launch.json`:
```json
"args": ["--loadmodule", "modules/redisearch/src/bin/macos-aarch64-release/search-community/redisearch.so"]
```

**Attach to a running server:**
Add another config to `launch.json`:
```json
{
  "name": "Attach to Redis Server",
  "type": "cppdbg",
  "request": "attach",
  "processId": "${command:pickProcess}",
  "MIMode": "lldb"
}
```

---

## 5) Common gotchas
- `cmake: command not found` → `brew install cmake` and ensure `eval "$($(brew --prefix)/bin/brew shellenv)"` is in your shell.
- Script expects **LLVM Clang** → ensure `/opt/homebrew/opt/llvm/bin` is **before** `/usr/bin` on `PATH` in the shell you build from.
- `Could not find the task 'build'` → task label mismatch or wrong folder; reload VS Code window after edits.
- Fresh build fails after tool changes → wipe `bin/ build/ CMake*` and re‑cmake.

---

## 6) Quick reference
```bash
# Export LLVM for current shell
export PATH="/opt/homebrew/opt/llvm/bin:$PATH"; export CC=clang; export CXX=clang++

# One-shot clean rebuild (RediSearch)
make clean || true && rm -rf bin build CMakeFiles CMakeCache.txt && \
cmake . -DCOORD_TYPE=oss -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_SHARED_LIBRARY_SUFFIX=.so \
-UCMAKE_TOOLCHAIN_FILE -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DSVS_SHARED_LIB=OFF && \
make -j$(sysctl -n hw.logicalcpu)

# Start Redis (foreground)
src/redis-server
# Start Redis with RediSearch
src/redis-server --loadmodule modules/redisearch/src/bin/macos-aarch64-release/search-community/redisearch.so

# Check server
src/redis-cli ping
```

---

**End of working notes.** Save this file to your repo or personal notes for reuse.

