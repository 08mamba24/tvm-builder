# TVM Builder Project - Session Context

## Current Goal
Continue debugging and fixing the GitHub Actions build workflow for Apache TVM on openEuler 22.03-lts-sp4 ARM64.

## Latest Fix (Commit <待提交>)
**Date**: 2026-02-12
**Issue**: Explicit llvm-config dependency checks were redundant and could cause false negatives.

**Solution**:
- **Completely removed** llvm-config dependency verification
- Simplified LLVM installation step (no post-install verification)
- Philosophy: "Install and let CMake verify" - let actual build process catch issues
- Reduced workflow complexity by eliminating redundant checks

**Reference**: TVM Official Docs - https://tvm.apache.org/docs/install/from_source
```bash
set(USE_LLVM "llvm-config --ignore-libllvm --link-static")
```

## Build Workflow Status
**Container**: `openeuler/openeuler:22.03-lts-sp4` (ARM64, GLIBC 2.34)
**Python Version**: 3.11
**TVM Version**: v0.22.0

## Key Workflow Steps (build-openeuler-arm64 job)
1. Install dependencies (gcc, cmake3, ninja-build, python3, etc.)
2. **Verify critical dependencies only** (checks gcc, g++, make, cmake3, ninja-build, python3, pip3, patchelf, zstd)
3. Install LLVM (llvm-devel package, no verification)
4. Build zstd (PIC static library for auditwheel)
5. Install Python build tools
6. Create CMake config (build/config.cmake)
7. Build Python wheels
8. Repair wheels with auditwheel (manylinux_2_34_aarch64)
9. Test TVM
10. Upload artifacts

## Previous Issues and Fixes

### Issue #1: Missing dependencies
- **Fixed**: Added comprehensive dependency installation and verification
- Added `elfutils` package (provides `patchelf` for auditwheel)
- Added fail-fast dependency check before build

### Issue #2: Python pip command
- **Fixed**: Changed `python3-pip` to `pip3` in dependency verification
- openEuler uses `pip3` command directly

### Issue #3: LLVM configuration - llvm-config-18
- **Fixed**: Use `llvm-config` instead of `llvm-config-18`
- TVM official docs specify `llvm-config` without version suffix
- System LLVM version doesn't need to be exactly 18 (>= 15 is sufficient)

### Issue #4: Redundant llvm-config verification
- **Fixed**: Removed all explicit llvm-config checks
- Rationale: yum install failures + CMake verification are sufficient
- Philosophy: "Let build process catch issues"

## CMake Configuration (build/config.cmake)
```cmake
set(CMAKE_BUILD_TYPE RelWithDebInfo)
set(USE_LLVM "llvm-config --ignore-libllvm --link-static")
set(HIDE_PRIVATE_SYMBOLS ON)
set(USE_CUDA OFF)
set(USE_METAL OFF)
set(USE_VULKAN OFF)
set(USE_OPENCL OFF)
set(USE_CUBLAS OFF)
set(USE_CUDNN OFF)
set(USE_CUTLASS OFF)
```

## Git Commands for Testing
```bash
# Check workflow status
gh workflow view build

# Trigger a new run
gh workflow run build.yml

# Watch the latest run
gh run watch

# View logs for specific job
gh run list --workflow=build.yml
gh run view <run-id> --log-failed
```

## Next Steps to Monitor
After pushing latest commit, monitor GitHub Actions workflow to see if:
1. Dependency verification passes (no llvm-config check)
2. LLVM installation succeeds (yum install)
3. TVM build completes successfully (CMake verifies llvm-config automatically)
4. Wheel repair with auditwheel works
5. TVM tests pass

## Monitoring Strategy
- **Initial phase**: Check frequently until builds start passing consistently
- **Stable phase**: Once builds pass multiple times in a row, reduce check frequency
- **Trigger failure**: Immediately investigate if workflow fails the GitHub Actions workflow to see if:
1. Dependency verification passes (llvm-config found)
2. LLVM installation succeeds
3. TVM build completes successfully
4. Wheel repair with auditwheel works
5. TVM tests pass

## Resources
- TVM Official Docs: https://tvm.apache.org/docs/install/from_source
- openEuler 22.03 Documentation: https://docs.openeuler.org/en/docs/22.03_LTS_SP4/
- TVM GitHub Issues: https://github.com/apache/tvm/issues
- auditwheel Documentation: https://github.com/pypa/auditwheel
