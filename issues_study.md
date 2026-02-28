### Library Issues on container start after system update
  
- NVIDIA driver/library stack was updated to 580.119.02 (you can see that from nvidia-smi and from /usr/lib64/libEGL_nvidia.so.580.119.02).
  
- Podman GPU integration (via NVIDIA CDI / container runtime device config) still had hard-coded paths to the previous driver libs, specifically /usr/lib64/libEGL_nvidia.so.580.105.08.
  
    - When podman starts, crun tried to mount that exact file into the container. Since that filename no longer existed on the host, crun failed with: “cannot stat … No such file”.


** What the fix did: **

```
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml 
```
- Regenerated the CDI spec from your currently installed driver, so it now points to the 580.119.02 libraries that actually exist.
- After that, Podman/crun could mount the right NVIDIA libraries/devices, and the container started normally.
- After an NVIDIA driver update, if containers suddenly fail with “cannot stat lib…nvidia.so.<oldversion>”, regenerate CDI (or reinstall/reconfigure the NVIDIA container toolkit hooks).



 ❯ podman start nim-gpt-oss-20b 
```
Error: unable to start container "807d1896b124e78c4cc4f6ab8b9d5ce8df8c98049615a323da7643de0d3a1eda": crun: cannot stat `/usr/lib64/libEGL_nvidia.so.580.105.08`: No such file or directory: OCI runtime attempted to invoke a command that was not found
```
    
❯ nvidia-smi
```
Sat Feb 28 21:23:52 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.119.02             Driver Version: 580.119.02     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:01:00.0  On |                  Off |
|  0%   40C    P8             12W /  450W |     908MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A           14855      G   /usr/bin/gnome-shell                    392MiB |
|    0   N/A  N/A           15633    C+G   /usr/bin/nautilus                        54MiB |
|    0   N/A  N/A           17745      G   /usr/bin/Xwayland                         9MiB |
|    0   N/A  N/A           18665    C+G   /usr/bin/gnome-text-editor               84MiB |
|    0   N/A  N/A           18740    C+G   /usr/bin/ptyxis                          60MiB |
|    0   N/A  N/A           26475      G   /usr/lib64/firefox/firefox              216MiB |
+-----------------------------------------------------------------------------------------+
```
  
 ❯ ls -lah /usr/lib64/libEGL_nvidia.so.*
```
lrwxrwxrwx. root root  27 B  Fri Dec 12 01:00:00 2025  /usr/lib64/libEGL_nvidia.so.0 ⇒ libEGL_nvidia.so.580.119.02
.rwxr-xr-x. root root 1.2 MB Mon Dec  8 08:40:15 2025  /usr/lib64/libEGL_nvidia.so.580.119.02
```

❯ sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```
INFO[0000] Using /usr/lib64/libnvidia-ml.so.580.119.02  
INFO[0000] Using /usr/lib64/libnvidia-sandboxutils.so.580.119.02 
INFO[0000] Auto-detected mode as 'nvml'                 
INFO[0000] Using driver version 580.119.02              
INFO[0000] Selecting /dev/nvidia-modeset as /dev/nvidia-modeset 
INFO[0000] Selecting /dev/nvidia-uvm-tools as /dev/nvidia-uvm-tools 
INFO[0000] Selecting /dev/nvidia-uvm as /dev/nvidia-uvm 
INFO[0000] Selecting /dev/nvidiactl as /dev/nvidiactl   
INFO[0000] Selecting /usr/lib64/libnvidia-egl-gbm.so.1.1.3 as /usr/lib64/libnvidia-egl-gbm.so.1.1.3 
INFO[0000] Selecting /usr/lib64/libnvidia-egl-wayland.so.1.1.21 as /usr/lib64/libnvidia-egl-wayland.so.1.1.21 
INFO[0000] Selecting /usr/lib64/libnvidia-allocator.so.580.119.02 as /usr/lib64/libnvidia-allocator.so.580.119.02 
WARN[0000] Could not locate libnvidia-vulkan-producer.so.580.119.02: pattern libnvidia-vulkan-producer.so.580.119.02 not found
libnvidia-vulkan-producer.so.580.119.02: not found 
WARN[0000] Could not locate nvidia_drv.so: pattern nvidia_drv.so not found 
WARN[0000] Could not locate libglxserver_nvidia.so.580.119.02: pattern libglxserver_nvidia.so.580.119.02 not found 
INFO[0000] Selecting /usr/share/glvnd/egl_vendor.d/10_nvidia.json as /usr/share/glvnd/egl_vendor.d/10_nvidia.json 
INFO[0000] Selecting /usr/share/egl/egl_external_platform.d/15_nvidia_gbm.json as /usr/share/egl/egl_external_platform.d/15_nvidia_gbm.json 
INFO[0000] Selecting /usr/share/egl/egl_external_platform.d/10_nvidia_wayland.json as /usr/share/egl/egl_external_platform.d/10_nvidia_wayland.json 
INFO[0000] Selecting /usr/share/nvidia/nvoptix.bin as /usr/share/nvidia/nvoptix.bin 
WARN[0000] Could not locate X11/xorg.conf.d/10-nvidia.conf: pattern X11/xorg.conf.d/10-nvidia.conf not found 
WARN[0000] Could not locate X11/xorg.conf.d/nvidia-drm-outputclass.conf: pattern X11/xorg.conf.d/nvidia-drm-outputclass.conf not found 
WARN[0000] Could not locate vulkan/icd.d/nvidia_icd.json: pattern vulkan/icd.d/nvidia_icd.json not found
pattern vulkan/icd.d/nvidia_icd.json not found 
WARN[0000] Could not locate vulkan/icd.d/nvidia_layers.json: pattern vulkan/icd.d/nvidia_layers.json not found
pattern vulkan/icd.d/nvidia_layers.json not found 
INFO[0000] Selecting /usr/share/vulkan/implicit_layer.d/nvidia_layers.json as /etc/vulkan/implicit_layer.d/nvidia_layers.json 
INFO[0000] Selecting /usr/share/vulkan/icd.d/nvidia_icd.x86_64.json as /etc/vulkan/icd.d/nvidia_icd.x86_64.json 
INFO[0000] Selecting /usr/lib64/libEGL_nvidia.so.580.119.02 as /usr/lib64/libEGL_nvidia.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libGLESv1_CM_nvidia.so.580.119.02 as /usr/lib64/libGLESv1_CM_nvidia.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libGLESv2_nvidia.so.580.119.02 as /usr/lib64/libGLESv2_nvidia.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libGLX_nvidia.so.580.119.02 as /usr/lib64/libGLX_nvidia.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libcuda.so.580.119.02 as /usr/lib64/libcuda.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libcudadebugger.so.580.119.02 as /usr/lib64/libcudadebugger.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvcuvid.so.580.119.02 as /usr/lib64/libnvcuvid.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-allocator.so.580.119.02 as /usr/lib64/libnvidia-allocator.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-cfg.so.580.119.02 as /usr/lib64/libnvidia-cfg.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-eglcore.so.580.119.02 as /usr/lib64/libnvidia-eglcore.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-encode.so.580.119.02 as /usr/lib64/libnvidia-encode.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-fbc.so.580.119.02 as /usr/lib64/libnvidia-fbc.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-glcore.so.580.119.02 as /usr/lib64/libnvidia-glcore.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-glsi.so.580.119.02 as /usr/lib64/libnvidia-glsi.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-glvkspirv.so.580.119.02 as /usr/lib64/libnvidia-glvkspirv.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-gpucomp.so.580.119.02 as /usr/lib64/libnvidia-gpucomp.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-gtk3.so.580.119.02 as /usr/lib64/libnvidia-gtk3.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-ml.so.580.119.02 as /usr/lib64/libnvidia-ml.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-ngx.so.580.119.02 as /usr/lib64/libnvidia-ngx.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-nvvm.so.580.119.02 as /usr/lib64/libnvidia-nvvm.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-opencl.so.580.119.02 as /usr/lib64/libnvidia-opencl.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-opticalflow.so.580.119.02 as /usr/lib64/libnvidia-opticalflow.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-pkcs11-openssl3.so.580.119.02 as /usr/lib64/libnvidia-pkcs11-openssl3.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-present.so.580.119.02 as /usr/lib64/libnvidia-present.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-ptxjitcompiler.so.580.119.02 as /usr/lib64/libnvidia-ptxjitcompiler.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-rtcore.so.580.119.02 as /usr/lib64/libnvidia-rtcore.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-sandboxutils.so.580.119.02 as /usr/lib64/libnvidia-sandboxutils.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-tls.so.580.119.02 as /usr/lib64/libnvidia-tls.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-vksc-core.so.580.119.02 as /usr/lib64/libnvidia-vksc-core.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvidia-wayland-client.so.580.119.02 as /usr/lib64/libnvidia-wayland-client.so.580.119.02 
INFO[0000] Selecting /usr/lib64/libnvoptix.so.580.119.02 as /usr/lib64/libnvoptix.so.580.119.02 
INFO[0000] Selecting /usr/lib64/vdpau/libvdpau_nvidia.so.580.119.02 as /usr/lib64/vdpau/libvdpau_nvidia.so.580.119.02 
WARN[0000] Could not locate /nvidia-persistenced/socket: pattern /nvidia-persistenced/socket not found 
WARN[0000] Could not locate /nvidia-fabricmanager/socket: pattern /nvidia-fabricmanager/socket not found 
WARN[0000] Could not locate /tmp/nvidia-mps: pattern /tmp/nvidia-mps not found 
INFO[0000] Selecting /lib/firmware/nvidia/580.119.02/gsp_ga10x.bin as /lib/firmware/nvidia/580.119.02/gsp_ga10x.bin 
INFO[0000] Selecting /lib/firmware/nvidia/580.119.02/gsp_tu10x.bin as /lib/firmware/nvidia/580.119.02/gsp_tu10x.bin 
INFO[0000] Selecting /usr/sbin/nvidia-smi as /usr/sbin/nvidia-smi 
INFO[0000] Selecting /usr/sbin/nvidia-debugdump as /usr/sbin/nvidia-debugdump 
INFO[0000] Selecting /usr/sbin/nvidia-persistenced as /usr/sbin/nvidia-persistenced 
INFO[0000] Selecting /usr/sbin/nvidia-cuda-mps-control as /usr/sbin/nvidia-cuda-mps-control 
INFO[0000] Selecting /usr/sbin/nvidia-cuda-mps-server as /usr/sbin/nvidia-cuda-mps-server 
WARN[0000] Could not locate nvidia-imex: pattern nvidia-imex not found 
WARN[0000] Could not locate nvidia-imex-ctl: pattern nvidia-imex-ctl not found 
WARN[0000] Could not locate nvidia_drv.so: pattern nvidia_drv.so not found 
WARN[0000] Could not locate libglxserver_nvidia.so.580.119.02: pattern libglxserver_nvidia.so.580.119.02 not found 
INFO[0000] Generated CDI spec with version 1.1.0        
```
 

 ❯ sudo grep -R "580.105.08" -n /etc/cdi/nvidia.yaml || echo "OK: old version not referenced"
```
OK: old version not referenced
```


 ❯ podman start nim-gpt-oss-20b
```
nim-gpt-oss-20b
```


 ❯ podman ps
```
CONTAINER ID  IMAGE                                  COMMAND     CREATED      STATUS        PORTS                                       NAMES
807d1896b124  nvcr.io/nim/openai/gpt-oss-20b:latest              4 weeks ago  Up 4 seconds  0.0.0.0:8000->8000/tcp, 6006/tcp, 8888/tcp  nim-gpt-oss-20b
```

