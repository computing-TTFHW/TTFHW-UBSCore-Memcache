# memcache-build

Build, incremental build with ccache, and unit testing for the memcache project inside a manlinux Docker container on a remote Linux host.

## Prerequisites

- SSH access to the remote Linux host (e.g., `root@<REMOTE_IP>:22`)
- Docker installed on the remote host
- Network access to pull `swr.cn-north-4.myhuaweicloud.com/memfabric-hybrid/memfabric-hybrid_x86:v20`

## Steps

### 1. Connect to Remote Host and Prepare Environment

```bash
# SSH to remote host (disable strict host key checking for automation)
ssh -o StrictHostKeyChecking=no -p 22 root@<REMOTE_IP>

# Pull the manlinux Docker image
docker pull swr.cn-north-4.myhuaweicloud.com/memfabric-hybrid/memfabric-hybrid_x86:v20

# Start container (run as root, detached, with sleep to keep alive)
docker rm -f memcache_build 2>/dev/null
docker run -d --name memcache_build --user root -w /root \
  swr.cn-north-4.myhuaweicloud.com/memfabric-hybrid/memfabric-hybrid_x86:v20 sleep 3600
```

### 2. Clone Repository and Initialize Submodules

```bash
# Clone the memcache repository inside the container
docker exec memcache_build bash -c 'git clone https://gitcode.com/Ascend/memcache.git'

# Initialize git submodules (required for spdlog, memfabric_hybrid, googletest, etc.)
docker exec memcache_build bash -c 'cd /root/memcache && git submodule update --init --recursive'
```

### 3. Install Dependencies Inside Container

```bash
# Install build tools
docker exec memcache_build bash -c 'dnf install -y ninja-build ccache dos2unix lcov 2>&1'

# Install Python dependencies
docker exec memcache_build bash -c 'python3 -m ensurepip 2>&1 && \
  python3 -m pip install pybind11 wheel auditwheel 2>&1'
```

### 4. Set Environment Variables

All builds must run with `IS_MANYLINUX=True` to produce manylinux-compatible wheels:

```bash
export IS_MANYLINUX=True
export MMC_BUILD_JOBS=32
export PATH="/opt/rh/gcc-toolset-14/root/usr/bin:$PATH"
```

### 5. Full Build (build_and_pack_run.sh)

The project provides `script/build_and_pack_run.sh` which orchestrates:
1. Version info writing
2. CMake configuration (RELEASE mode, Ninja generator)
3. Ninja compile + install
4. Copy libraries and Python wrapper
5. Build Python wheel (`setup.py bdist_wheel`)
6. Package `.run` installer

**Key CMake options:**
- `-DCMAKE_BUILD_TYPE=RELEASE`
- `-DBUILD_TESTS=OFF`
- `-DBUILD_PYTHON=ON`
- `-DENABLE_PTRACER=ON`

**Output artifacts:**
- Wheel: `output/memcache/wheel/memcache_hybrid-*.whl`
- Run package: `output/memcache_hybrid-*-linux_x86_64.run`

### 6. Incremental Build with ccache

Install ccache and configure it to speed up incremental builds:

```bash
# Install ccache
dnf install -y ccache

# Create symlink-based compiler wrappers (preferred over CC="ccache gcc")
mkdir -p /root/ccache_links
ln -sf /usr/bin/ccache /root/ccache_links/gcc
ln -sf /usr/bin/ccache /root/ccache_links/g++
ln -sf /usr/bin/ccache /root/ccache_links/cc
ln -sf /usr/bin/ccache /root/ccache_links/c++
export PATH="/root/ccache_links:$PATH"
```

**Procedure:**
1. **First full build** with ccache launcher to populate the cache:
   ```bash
   cmake -G Ninja -DCMAKE_BUILD_TYPE=RELEASE \
     -DCMAKE_C_COMPILER_LAUNCHER=ccache \
     -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
     -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ \
     -S . -B build/
   ninja install -j32 -C build/
   ```

2. **Clear stats**, then modify source files (e.g., add a comment to 5 `.cpp` files in `src/memcache/csrc/`)

3. **Incremental rebuild** (only modified files recompile, rest served from ccache):
   ```bash
   cmake -G Ninja ... -S . -B build/   # incremental reconfigure
   ninja install -j32 -C build/         # incremental compile
   ```

4. **Check cache hit rate:**
   ```bash
   ccache -s   # look for "cache hit rate" line
   ```

Typical ccache hit rate: **~85%** when modifying a small subset of files.

### 7. Unit Testing (run_ut.sh)

The project provides `script/run_ut.sh` which:
1. Converts DOS line endings (`dos2unix`) for mockcpp sources and patches
2. CMake configures with tests enabled (`-DBUILD_TESTS=ON -DCMAKE_BUILD_TYPE=DEBUG`)
3. Builds test binaries
4. Runs `test_mmc_test` with gtest XML output
5. Generates code coverage with lcov

**Key CMake options for tests:**
- `-DCMAKE_BUILD_TYPE=DEBUG`
- `-DBUILD_TESTS=ON`
- `-DBUILD_GIT_COMMIT=OFF -DBUILD_GIT_COMMIT_GEN_FILE=OFF`
- `-DBUILD_OPEN_ABI=ON`

**Required environment for test execution:**
```bash
export LD_LIBRARY_PATH="$MEMCACHE_LIB_PATH:$SMEM_LIB_PATH:$HYBM_LIB_PATH:$MOCK_CANN:$MOCK_DFC:$MOCK_CANN_DRIVER"
export ASCEND_HOME_PATH="$MOCK_CANN"
export ASAN_OPTIONS="detect_stack_use_after_return=1:allow_user_poisoning=1:detect_leaks=0"
```

**Test coverage thresholds:**
- Line coverage: >= 70%
- Branch coverage: >= 40%

### 8. Timing Each Phase

To measure build/test times, wrap each phase with Python's `time.time()`:

```python
import time, json

TIMES = {}
def run(name, cmd):
    start = time.time()
    subprocess.run(cmd, shell=True, ...)
    TIMES[name] = round((time.time() - start) * 1000)
```

### 9. Collect Build Artifacts

```bash
# Copy JSON results and packages from container to remote host
docker cp memcache_build:/root/build_result.json /root/
docker cp memcache_build:/root/memcache/output/memcache/wheel/*.whl /root/
docker cp memcache_build:/root/memcache/output/*.run /root/

# Copy from remote host to local Windows machine
scp root@<REMOTE_IP>:/root/build_result.json ./build/
scp root@<REMOTE_IP>:/root/*.whl ./build/
scp root@<REMOTE_IP>:/root/*.run ./build/
```

## JSON Output Format

The unified `build_result.json` structure:

```json
{
  "build": {
    "total_duration_ms": 83839,
    "copy_version_ms": 6,
    "cmake_configure_ms": 9307,
    "ninja_compile_ms": 71999,
    "copy_libs_ms": 11,
    "copy_python_wrapper_ms": 6,
    "build_wheel_ms": 2422,
    "copy_wheel_ms": 9,
    "make_run_package_ms": 78,
    "_comment": "total_duration_ms=总耗时, copy_version_ms=版本信息写入, cmake_configure_ms=CMake配置, ninja_compile_ms=Ninja编译, copy_libs_ms=拷贝动态库, copy_python_wrapper_ms=拷贝Python绑定库, build_wheel_ms=构建Wheel包, copy_wheel_ms=拷贝Wheel包到输出目录, make_run_package_ms=制作.run安装包"
  },
  "Incremental build": {
    "incremental_cmake_ms": 450,
    "incremental_ninja_ms": 5609,
    "total_duration_ms": 6059,
    "modified_files_count": 5,
    "ccache": {
      "cache_hit_direct": 132,
      "cache_hit_preprocessed": 0,
      "cache_miss": 22,
      "cache_total": 154,
      "cache_hit_rate_percent": 85.71,
      "_comment": "cache_hit_direct=直接缓存命中, cache_hit_preprocessed=预处理缓存命中, cache_miss=缓存未命中, cache_total=缓存查询总数, cache_hit_rate_percent=缓存命中率(%)"
    },
    "_comment": "安装ccache后修改5个源文件进行增量编译; incremental_cmake_ms=CMake配置, incremental_ninja_ms=Ninja编译, total_duration_ms=增量编译总耗时, modified_files_count=修改的源文件数量"
  },
  "UT": {
    "total_duration_ms": 223039,
    "dos2unix_conversion_ms": 6,
    "cmake_configure_test_ms": 7873,
    "build_tests_ms": 92047,
    "run_ut_tests_ms": 123110,
    "_comment": "total_duration_ms=UT总耗时, dos2unix_conversion_ms=dos2unix格式转换, cmake_configure_test_ms=测试CMake配置, build_tests_ms=构建测试程序, run_ut_tests_ms=运行148个测试用例(全部通过)"
  }
}
```

## Common Issues

| Issue | Solution |
|-------|----------|
| `ninja: command not found` | Install with `dnf install -y ninja-build` |
| `pybind11 not found` | Install with `python3 -m pip install pybind11` |
| `wheel module not found` | Install with `python3 -m pip install wheel` |
| Submodule files missing (spdlog, etc.) | Run `git submodule update --init --recursive` |
| GCC not found at default path | Add `/opt/rh/gcc-toolset-14/root/usr/bin` to PATH |
| ccache fails with "missing equal sign" | Don't set `CC=ccache gcc`; use symlinks or `-DCMAKE_C_COMPILER_LAUNCHER=ccache` |
| Docker copy fails (file not found) | Verify the file path exists inside the container with `docker exec ... ls` |

## Field Naming Convention

| Chinese | JSON Key |
|---------|----------|
| 编译 | `build` |
| 增量编译 | `Incremental build` |
| UT测试 | `UT` |
