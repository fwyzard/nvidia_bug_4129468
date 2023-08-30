This repository contains the preprocessed files to reproduce the NVIDIA bug #4129468 (https://developer.nvidia.com/nvidia_bug/4129468).
The problem has been fixed in CUDA 12.2.1.

## Description

While compiling one of the files from https://github.com/cms-sw/cmssw/ with CUDA 12.1.1 we observe a segmentation fault within `cicc`:
```bash
/usr/local/cuda-12.1/bin/nvcc -x cu -dc -DGNU_GCC -D_GNU_SOURCE -DEIGEN_DONT_PARALLELIZE -DTBB_USE_GLIBCXX_VERSION=110201 -DTBB_SUPPRESS_DEPRECATED_MESSAGES -DTBB_PREVIEW_RESUMABLE_TASKS=1 -DTBB_PREVIEW_TASK_GROUP_EXTENSIONS=1 -DBOOST_SPIRIT_THREADSAFE -DPHOENIX_THREADSAFE -DBOOST_MATH_DISABLE_STD_FPCLASSIFY -DBOOST_UUID_RANDOM_PROVIDER_FORCE_POSIX -DDD4HEP_USE_GEANT4_UNITS=1 -DCMSSW_GIT_HASH='CMSSW_13_0_1' -DPROJECT_NAME='CMSSW' -DPROJECT_VERSION='CMSSW_13_0_1' -I/data/user/fwyzard/CMSSW_13_0_1/src -I/data/cmssw/el8_amd64_gcc11/cms/cmssw/CMSSW_13_0_1/src -I/data/cmssw/el8_amd64_gcc11/external/boost/1.80.0-bff81045eb6d0806da34e08d781c05db/include -I/usr/local/cuda-12.1/include -I/data/cmssw/el8_amd64_gcc11/external/tbb/v2021.8.0-bffa28ed2b6c238943a7fc1c9ad8f6f2/include -I/data/cmssw/el8_amd64_gcc11/external/eigen/82dd3710dac619448f50331c1d6a35da673f764a-b0dc243d10365424e66e60c822ab818e/include/eigen3 --diag-suppress 20014 -std=c++17 -O3 --generate-line-info --display-error-number --expt-relaxed-constexpr --extended-lambda -gencode arch=compute_75,code=[sm_75,compute_75] -Wno-deprecated-gpu-targets -Xcudafe --diag_suppress=esa_on_defaulted_function_ignored --cudart shared --compiler-options '-O2 -pthread -pipe -ftree-vectorize -fvisibility-inlines-hidden -fno-math-errno --param vect-max-version-for-alias-checks=50 -fuse-ld=bfd -msse3 -felide-constructors -fmessage-length=0 -fdiagnostics-show-option -DBOOST_DISABLE_ASSERTS -std=c++17 -fPIC ' /data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu -o tmp/el8_amd64_gcc11/src/RecoLocalTracker/SiPixelRecHits/plugins/RecoLocalTrackerSiPixelRecHitsPlugins/PixelRecHitGPUKernel.cu.o --source-in-ptx
```
results in:
```
nvcc error : 'cicc' died due to signal 11 (Invalid memory reference)
nvcc error : 'cicc' core dumped
```

Compiling the same file with CUDA 11.5.2, 11.8.0 and 12.0.1 works without problems.

Compiling the same file with CUDA 12.1.1 without `--source-in-ptx` also works.


## Reproducer

To reproduce:
```bash
/usr/local/cuda-12.1/nvvm/bin/cicc --c++17 --gnu_version=110201 --diag_suppress 20014 --display_error_number --orig_src_file_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --orig_src_path_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --allow_managed --extended-lambda --relaxed_constexpr  --device-c  --diag_suppress=esa_on_defaulted_function_ignored  -arch compute_75 -show-src -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name PixelRecHitGPUKernel.fatbin.c -generate-line-info -tused --module_id_file_name PixelRecHitGPUKernel.module_id --gen_c_file_name PixelRecHitGPUKernel.cudafe1.c --stub_file_name PixelRecHitGPUKernel.cudafe1.stub.c --gen_device_file_name PixelRecHitGPUKernel.cudafe1.gpu PixelRecHitGPUKernel.cpp1.ii -o PixelRecHitGPUKernel.ptx
```
should result in
```
Segmentation fault (core dumped)
```

Compiling without `-show-src` seems to work:
```bash
/usr/local/cuda-12.1/nvvm/bin/cicc --c++17 --gnu_version=110201 --diag_suppress 20014 --display_error_number --orig_src_file_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --orig_src_path_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --allow_managed --extended-lambda --relaxed_constexpr  --device-c  --diag_suppress=esa_on_defaulted_function_ignored  -arch compute_75 -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name PixelRecHitGPUKernel.fatbin.c -generate-line-info -tused --module_id_file_name PixelRecHitGPUKernel.module_id --gen_c_file_name PixelRecHitGPUKernel.cudafe1.c --stub_file_name PixelRecHitGPUKernel.cudafe1.stub.c --gen_device_file_name PixelRecHitGPUKernel.cudafe1.gpu PixelRecHitGPUKernel.cpp1.ii -o PixelRecHitGPUKernel.ptx
```

As does compiling the same preprocessed files with `cicc` from CUDA 12.0.1:
```bash
/usr/local/cuda-12.0/nvvm/bin/cicc --c++17 --gnu_version=110201 --diag_suppress 20014 --display_error_number --orig_src_file_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --orig_src_path_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --allow_managed --extended-lambda --relaxed_constexpr  --device-c  --diag_suppress=esa_on_defaulted_function_ignored  -arch compute_75 -show-src -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name PixelRecHitGPUKernel.fatbin.c -generate-line-info -tused --module_id_file_name PixelRecHitGPUKernel.module_id --gen_c_file_name PixelRecHitGPUKernel.cudafe1.c --stub_file_name PixelRecHitGPUKernel.cudafe1.stub.c --gen_device_file_name PixelRecHitGPUKernel.cudafe1.gpu PixelRecHitGPUKernel.cpp1.ii -o PixelRecHitGPUKernel.ptx
```


## Update for CUDA 12.2.0

The problem seems to be still present in CUDA 12.2.0:
```bash
/usr/local/cuda-12.2/bin/nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Tue_Jun_13_19:16:58_PDT_2023
Cuda compilation tools, release 12.2, V12.2.91
Build cuda_12.2.r12.2/compiler.32965470_0

/usr/local/cuda-12.2/nvvm/bin/cicc --c++17 --gnu_version=110201 --diag_suppress 20014 --display_error_number --orig_src_file_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --orig_src_path_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --allow_managed --extended-lambda --relaxed_constexpr  --device-c  --diag_suppress=esa_on_defaulted_function_ignored  -arch compute_75 -show-src -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name PixelRecHitGPUKernel.fatbin.c -generate-line-info -tused --module_id_file_name PixelRecHitGPUKernel.module_id --gen_c_file_name PixelRecHitGPUKernel.cudafe1.c --stub_file_name PixelRecHitGPUKernel.cudafe1.stub.c --gen_device_file_name PixelRecHitGPUKernel.cudafe1.gpu PixelRecHitGPUKernel.cpp1.ii -o PixelRecHitGPUKernel.ptx
```
results in
```
Segmentation fault (core dumped)
```

## Update for CUDA 12.2.1

The problem has been fixed in CUDA 12.2.1:
```bash
/usr/local/cuda-12.2/bin/nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Tue_Jul_11_02:20:44_PDT_2023
Cuda compilation tools, release 12.2, V12.2.128
Build cuda_12.2.r12.2/compiler.33053471_0

/usr/local/cuda-12.2/nvvm/bin/cicc --c++17 --gnu_version=110201 --diag_suppress 20014 --display_error_number --orig_src_file_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --orig_src_path_name "/data/user/fwyzard/CMSSW_13_0_1/src/RecoLocalTracker/SiPixelRecHits/plugins/PixelRecHitGPUKernel.cu" --allow_managed --extended-lambda --relaxed_constexpr  --device-c  --diag_suppress=esa_on_defaulted_function_ignored  -arch compute_75 -show-src -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name PixelRecHitGPUKernel.fatbin.c -generate-line-info -tused --module_id_file_name PixelRecHitGPUKernel.module_id --gen_c_file_name PixelRecHitGPUKernel.cudafe1.c --stub_file_name PixelRecHitGPUKernel.cudafe1.stub.c --gen_device_file_name PixelRecHitGPUKernel.cudafe1.gpu PixelRecHitGPUKernel.cpp1.ii -o PixelRecHitGPUKernel.ptx
```
builds correctly.
