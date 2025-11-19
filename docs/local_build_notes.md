# Local build notes (Developer saved)

This file documents the local toolchain and setup used on this machine. The goal is to record exact steps, paths, and quick fixes so we don't need to troubleshoot these again.

## Quick system checks (copy/paste)
- Compiler and CUDA:
  - g++ 13: `g++-13 --version`
  - nvcc: `which nvcc && nvcc --version`
  - Check CUDA location: `ls -l /usr/local/cuda` and `ls -l /usr/local/cuda/bin/nvcc`

- CMake: `cmake --version` (requires >= 3.25 for local build)
- Doxygen: `doxygen --version` (project requires >= 1.10 if you want docs)
- nlohmann json: `ls -l /usr/include/nlohmann/json.hpp || ls -l ~/.local/include/nlohmann/json.hpp` (or use the local `build/_deps/json-src` copy)

## Known paths found on this machine
- NVCC (from repo): `/home/wewantgreatness/Unity/UnityProjects/TEST/MoonQuestVR/Sunshine/build/cuda/bin/nvcc`
- NVIDIA Path symlink: `/usr/local/cuda -> /home/.../Sunshine/build/cuda/` (this repo provided a local CUDA toolchain)
- Installed sunshine binary (older): `/usr/bin/sunshine`
- User config: `~/.config/sunshine/sunshine.conf` (controls `output_name`)
- User logs: `~/.config/sunshine/sunshine.log` (search here for "desktop" or "Selected entire virtual desktop")
- nlohmann fallback copy in the project: `build/_deps/json-src/include/nlohmann/json.hpp` (CMake FetchContent falls back here if system not found)

## Local workarounds & reproducible steps
1) If CMake complains about `nlohmann_json` not being found, we used a local CMake config to expose local headers:

```bash
# Create ~/.local/nlohmann_jsonConfig.cmake pointing to the repo copy
JSON_INCLUDE="/home/wewantgreatness/Unity/UnityProjects/TEST/MoonQuestVR/Sunshine/build/_deps/json-src/include"
mkdir -p ~/.local
cat > ~/.local/nlohmann_jsonConfig.cmake <<EOF
# Auto-generated local config
if (NOT TARGET nlohmann_json::nlohmann_json)
  add_library(nlohmann_json::nlohmann_json INTERFACE IMPORTED)
  set_target_properties(nlohmann_json::nlohmann_json PROPERTIES
    INTERFACE_INCLUDE_DIRECTORIES "${JSON_INCLUDE}"
  )
endif()
set(nlohmann_json_FOUND TRUE)
EOF
cp ~/.local/nlohmann_jsonConfig.cmake ~/.local/nlohmann_json-config.cmake
```

2) If `CMake` complains about `CUDA` not found, check the path of the `nvcc` tool (example):

```bash
# typical checks
which nvcc || command -v nvcc
nvcc --version
# if nvcc is inside the repo build/cuda, export PATH or point CMake explicitly
export PATH=/home/wewantgreatness/Unity/UnityProjects/TEST/MoonQuestVR/Sunshine/build/cuda/bin:$PATH
# or pass to CMake
-DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc
```

3) If `CMake` complains about Doxygen (< 1.10 required) and you don't need docs locally, disable docs:

```bash
cmake -S . -B build-local -G Ninja \
  -DCMAKE_C_COMPILER=/usr/bin/gcc-13 \
  -DCMAKE_CXX_COMPILER=/usr/bin/g++-13 \
  -DSUNSHINE_ENABLE_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc \
  -DBUILD_DOCS=OFF
```

4) If `CMake` shows root-owned build files (error on `rm -rf build`), either use `build-local` or fix permissions:

```bash
# Option 1: use a separate build dir (recommended)
rm -rf build-local
# Option 2: change owner (careful, requires sudo environment)
sudo chown -R $(id -u):$(id -g) build
```

## CLI Build commands (copy/paste) — one at a time
- Configure (CUDA ON):
```bash
rm -rf build-local
cmake -S . -B build-local -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=/usr/bin/gcc-13 \
  -DCMAKE_CXX_COMPILER=/usr/bin/g++-13 \
  -DSUNSHINE_ENABLE_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc \
  -DCMAKE_CUDA_HOST_COMPILER=/usr/bin/gcc-13 \
  -DBUILD_DOCS=OFF -Wno-dev
```
- Build:
```bash
cmake --build build-local --target sunshine -- -j$(nproc)
```
- Run:
```bash
./build-local/sunshine 2>&1 | tee ~/sunshine_new.log
grep -i "Selected entire virtual desktop for streaming" ~/sunshine_new.log || true
grep -i "desktop" ~/sunshine_new.log || true
```

## How to test `desktop` (quick steps)
- Update config:
```bash
sed -i 's/^output_name = .*/output_name = desktop/' ~/.config/sunshine/sunshine.conf
```
- Restart the user service to pick it up (or run the local binary manually):
```bash
systemctl --user restart sunshine.service
```
- Tail logs:
```bash
journalctl --user -u sunshine.service -n 200 --no-hostname
# or 
less +F ~/.config/sunshine/sunshine.log
```

## Notes
- If you want NVFBC-specific behavior (`desktop` via NVidia), you must build with CUDA support enabled and nvcc available. The local repo has a prepackaged CUDA toolchain in `build/cuda` that can be used.
- The installed system package `/usr/bin/sunshine` is separate from the local build; it shows the previously built version (useful to test a previously compiled version).
- This file is meant to be kept in repo `docs/` and updated when you make local environment changes so you won't need to re-discover the same steps.

---
Documented by developer (copied to `docs/local_build_notes.md`)
