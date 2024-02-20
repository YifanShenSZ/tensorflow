# Build TensorFlow on Rosetta (February 2024)
The x86 TensorFlow wheels available on pypi and conda all use avx and later instruction sets. However, as a virtual machine intended for backward compatibility, rosetta has only old instruction set (presumably sse or older). As a result, to use TensorFlow on rosetta, we will have to build ourselves

Due to Bazel issue, the latest TensorFlow we can build is 2.7. See Appendix 2 for details

Subsequently, given TensorFlow 2.7, the latest python we can support is 3.9, since starting from 3.10 `MutableSequence` is moved from `collections` to `collections.abc`

Reference
> 1. TensorFlow [build-from-source doc](https://www.tensorflow.org/install/source#macos)
> 2. Apple Silicon [rosetta doc](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment)


## Steps

### Prepare Environment
Create a rosetta environment (taking python 3.9 as an example)
```
CONDA_SUBDIR=osx-64 conda create -p envs/tf2.7-py3.9 -y
conda activate envs/tf2.7-py3.9
conda config --env --set subdir osx-64
conda install python=3.9 cmake -y
```

Install common packages according to coremltools taste
```
pip install -r coremltools-test.pip
```

Install remaining TensorFlow build dependencies
```
pip install opt_einsum
pip install keras_preprocessing --no-deps
conda install -c conda-forge bazel=3.7.2 -y
```

Specify clang as the compiler (although clang is actually required, somehow gcc is the default)
```
export CC=clang
```

### Configure
Run TensorFlow configure script
```
python configure.py
```
When prompted `Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -Wno-sign-compare]:`, enter
```
-march=x86-64
```

### Build
Build package builder
```
bazel build --config=opt //tensorflow/tools/pip_package:build_pip_package
```

Build package
```
./bazel-bin/tensorflow/tools/pip_package/build_pip_package wheels
```

The built wheel has suffix "macosx_xx_yy", where "xx_yy" is our current MacOS version. However, again, due to backward compatibility purpose, rosetta python targets older MacOS. This can be simply resolved by rename the suffix, though, since our real arm MacOS is current. To know which suffix can cheat pip (taking python 3.9 as an example)
```
pip debug --verbose |grep cp39-cp39-macosx |grep x86_64
```


## Appendix 1: Modified Source
Branch r2.7 uses an old `boringssl` that has undefined variables, which would fail in C++11 and higher. To resolve this issue, we have updated the `boringssl` in `tensorflow/workspace2.bzl` to
```
    tf_http_archive(
        name = "boringssl",
        sha256 = "9dc53f851107eaf87b391136d13b815df97ec8f76dadb487b58b2fc45e624d2c",
        strip_prefix = "boringssl-c00d7ca810e93780bd0c8ee4eea28f4f2ea4bcdc",
        system_build_file = "//third_party/systemlibs:boringssl.BUILD",
        urls = [
            "https://storage.googleapis.com/mirror.tensorflow.org/github.com/google/boringssl/archive/c00d7ca810e93780bd0c8ee4eea28f4f2ea4bcdc.tar.gz",
            "https://github.com/google/boringssl/archive/c00d7ca810e93780bd0c8ee4eea28f4f2ea4bcdc.tar.gz",
        ],
    )
```

Reference: https://github.com/tensorflow/tensorflow/issues/60191#issuecomment-1498895246


## Appendix 2: Bazel Issues
TensorFlow uses Bazel to build

### TensorFlow 2.8 (Bazel 4.2.1)
We encountered
```
(tf2-py310) nninfci@nninfcis-MacBook-Pro-3 TensorFlow-2.8 % bazel build //tensorflow/tools/pip_package:build_pip_package
~/Debug/TensorFlow-2.8/envs/tf2-py310/bin ~/Debug/TensorFlow-2.8
~/Debug/TensorFlow-2.8
dyld[95908]: Symbol not found: __ZN4absl12lts_2021110219str_format_internal13FormatArgImpl8DispatchINS0_11string_viewEEEbNS2_4DataENS1_24FormatConversionSpecImplEPv
  Referenced from: <3984A6B7-EDAB-3871-A95C-98BEFE86F1B6> /Users/nninfci/Debug/TensorFlow-2.8/envs/tf2-py310/lib/libgpr.24.0.0.dylib
  Expected in:     <8DCB7680-E9A4-3B3A-B5C8-1D8765946EC7> /Users/nninfci/Debug/TensorFlow-2.8/envs/tf2-py310/lib/libabsl_str_format_internal.2111.0.0.dylib
/Users/nninfci/Debug/TensorFlow-2.8/envs/tf2-py310/bin/bazel: line 20: 95908 Abort trap: 6           $PREFIX_DIR/bin/bazel-real --output_user_root ${PREFIX_DIR}/share/bazel $*
```
The culprit seems to be "Bazel 4.2.1 does not properly link to rosetta dynamic libraries"

### TensorFlow 2.9+ (Bazel 5+)
We would first encounter
```
(tf2-py310) nninfci@nninfcis-MacBook-Pro-3 TensorFlow-2.11 % bazel build //tensorflow/tools/pip_package:build_pip_package
FATAL: corrupt installation: file '/Users/nninfci/Debug/TensorFlow-2.11/envs/tf2-py310/share/bazel/install/d77d71e07e81dfb36661713e8e56be33/A-server.jar' is missing or modified.  Please remove '/Users/nninfci/Debug/TensorFlow-2.11/envs/tf2-py310/share/bazel/install/d77d71e07e81dfb36661713e8e56be33' and try again.
```
If we follow instruction "remove and try again", then we would further encounter
```
Cuda Configuration Error: Invalid cpu_value: 
ERROR: /Users/nninfci/Debug/TensorFlow-2.11/WORKSPACE:15:14: fetching cuda_configure rule //external:local_config_cuda: Traceback (most recent call last):
	File "/Users/nninfci/Debug/TensorFlow-2.11/third_party/gpus/cuda_configure.bzl", line 1394, column 33, in _cuda_autoconf_impl
		_create_dummy_repository(repository_ctx)
	File "/Users/nninfci/Debug/TensorFlow-2.11/third_party/gpus/cuda_configure.bzl", line 786, column 43, in _create_dummy_repository
		"%{cuda_driver_lib}": lib_name("cuda", cpu_value),
	File "/Users/nninfci/Debug/TensorFlow-2.11/third_party/gpus/cuda_configure.bzl", line 477, column 28, in lib_name
		auto_configure_fail("Invalid cpu_value: %s" % cpu_value)
	File "/Users/nninfci/Debug/TensorFlow-2.11/third_party/gpus/cuda_configure.bzl", line 329, column 9, in auto_configure_fail
		fail("\n%sCuda Configuration Error:%s %s\n" % (red, no_color, msg))
Error in fail:
```
The culprit seems to be "Bazel 5+ does not install properly on rosetta, and the re-try somehow ignores our cuda=N configuration"

## Appendix 3: Some Details
Bazel works as if creating a local server
1. Start by `bazel build ...`
2. Shutdown by `bazel shutdown` then clean by `bazel clean`

The local server stores file in
```
/private/var/tmp/_bazel_<user-name>/<some-hash>/
```
