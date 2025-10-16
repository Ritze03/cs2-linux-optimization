# Introduction (quick overview)
This guide is aimed at players who want the smoothest and most responsible CS2 experience on Linux with AMD GPUs. I will walk you through every step and explain what and why we are doing each step. 

___
⚠️ **Note:** You’ll need some basic experience using the terminal to follow this guide.  
I highly recommend reading it all the way through first to make sure you understand and feel confident about each step before executing any commands.

⚠️ **Disclaimer:** This guide is intended for AMD GPUs on Linux. Following it on other setups may cause instability or reduce performance. Proceed at your own risk. Always back up important data before making system changes. 
I added disclaimers for every critical step! Read them thoroughly!

💾 **Backup info:** This guide includes instructions to **backup critical configuration files** (like `~/.bash_profile` and `~/.config/environment.d/99-radv.conf`) before making changes. If anything goes wrong, you can **restore the original files** from these backups to undo all edits and return your system to its previous state.

Always **back up important data** before making system changes and read all disclaimers carefully before proceeding.
___

This guide covers:
- Ensuring that your system uses the ACO shader compiler.
- Installing and configuring the correct Mesa Vulkan driver (`vulkan-radeon`)
- Forcing your system to use ACO by default for Vulkan games.
- Setting up optimal shader cache paths for faster compilation.
- Optionally setting up AMD user queues for lower latency
- Configuring Gamescope for Wayland/X11 to achieve smoother frame pacing and minimal input lag.
- Fine-tuning launch options, CPU thread affinity, and engine variables for best performance.
- Fixing potential VAC errors caused by Gamescopes “nice” capabilities.

Each step is designed to remove every bottleneck possible and make CS2 perform at its absolute best on Linux by eliminating shader stutter and ensuring consistent frametimes.
___
# 1. My Setup
CPU: Intel Core i9-10900K @ 5.1 GHz

GPU: AMD Radeon RX 6700 XT

RAM: 32GB @ 3333 MHz

OS: EndeavourOS (Arch)

Kernel: 6.16.12-lqx

Dekstop Environment: KDE Plasma 6

Display Server: Wayland

*I can only guarantee for similar OS/Kernel setups! If your setup is vastly different, proceed with caution.* 
___
# 2. Is this guide even worth your time? (Performance comparison)

Map used for benchmarking: [https://steamcommunity.com/sharedfiles/filedetails/?id=3240880604](https://steamcommunity.com/sharedfiles/filedetails/?id=3240880604)

|                                    | AVG FPS     | P1 FPS      |
| ---------------------------------- | ----------- | ----------- |
| **LLPC, no optimizations**         | 264         | 121         |
| **LLPC 2nd run, no optimizations** | 162         | 94          |
| **ACO + optimizations**            | 395 (+49%)  | 192 (+58%)  |
| **ACO 2nd run  + optimizations**   | 407 (+151%) | 190 (+102%) |

*And yes, this is higher than I get in Windows 10/11. Input latency is effectively eliminated, making the game feel extremely responsive!*

*On ACO I’m peaking at 1000+ FPS in the last scene, where the camera is pointed towards the sky! (not representative of actual gameplay, but still cool)*

Before switching to ACO I had massive performance degradation problems over time. My FPS kept getting worse and worse. Compiling shaders took like 20 minutes after every update. Switching to ACO fixed my performance degradation problems. Compiling shaders takes a maximum of 2 minutes.
___
# 3. Checking if you are using the ACO shader compiler
*I decided to include this, since I encountered this problem twice now. It SHOULD be configured this way by default.*

**What even is ACO?**
*The mesa ACO shader compiler is an open source shader compiler created and developed by Valve Corporation. It is the default shader compiler used since mesa version 20.2.
Some systems somehow default to the LLPC/LLVM shader compiler by default for some reason. This sometimes happens even with mesa 25.2.4-2 installed*


In order to check if you are using ACO, run the following command:
```bash
vulkaninfo | grep driver
```


We mainly care about the `driverName` and `driverInfo`.

**LLPC Example Output:**
```bash
VK_LUNARG_direct_driver_loading : extension revision 1 
driverVersion = 2.0.349 (8388957) 
driverUUID = 414d442d-4c49-4e55-582d-445256000000 
driverID = DRIVER_ID_AMD_OPEN_SOURCE 
driverName = AMD open-source driver            ← AMD open-source (LLPC)
driverInfo = 2025.Q2.1 (LLPC)                  ← Dead giveaway for LLPC
VK_KHR_driver_properties : extension revision 1
```
*Perfect! We most likely found the issue for your performance problems! Keep reading!*


**ACO Example Output:**
```bash
VK_LUNARG_direct_driver_loading : extension revision 1
driverVersion = 25.2.4 (104865796)
driverUUID = 414d442d-4d45-5341-2d44-525600000000
driverID = DRIVER_ID_MESA_RADV
driverName = radv                             ← radv Driver (ACO ready!)
driverInfo = Mesa 25.2.4-arch1.2              ← mesa >20.2 ACO by default!
VK_KHR_driver_properties : extension revision 1
```

*You are already running ACO. Skip ahead to step X.X for further improvements!*
___
# 4. Installing the necessary packages for ACO
Run the following command:
```bash
sudo pacman -S vulkan-radeon lib32-vulkan-radeon
```

*Reboot and repeat step 3. If that didn’t do the trick, continue with step 5. Otherwise skip to step 6 for more optimizations!*
___
# 5. Forcing all applications to default to ACO
## 5.0 breakdown of what we are going to do
*You can skip reading this if you don’t care to understand, what we are actually going to do.*

#### We are going to modify the following variables:
**VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json**
*By default the Driver is selected in alphabetical order. The default amdgpu driver comes first. We are going to override it with the radeon_icd driver.*

**RADV_PERFTEST=aco**
*This explicitly tells the GPU driver to use ACO for shader compilation.*

**MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders**
*This forces shaders to be on your main drive. This is useful, since it is most likely the fastest drive in your system.*

## 5.1 Forcing ACO for programs launched from Terminal
⚠️ **Disclaimer:** First backup your current `~/.bash_profile` using the following command:
```bash
cp ~/.bash_profile ~/.bash_profile.BACKUP
```

If anything goes wrong, you can restore it with the following command:
```bash
cp ~/.bash_profile.BACKUP ~/.bash_profile
```

**Restoring from backup will undo any edits you made to this file, including changes from this guide.**

**Reboot your system afterwards to apply the changes.**
___

Run the following command to edit your default bash profile:
```bash
nano ~/.bash_profile
```

Insert the following lines at the end of the config file:
```bash
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json
export RADV_PERFTEST=aco
export MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
```

Press `Ctrl + X` → `Y` → `Enter` to save the file.

## 5.2 Forcing ACO for programs launched from GUI
⚠️ **Disclaimer:** First backup your current `~/.config/environment.d/99-radv.conf` using the following command:
```bash
cp ~/.config/environment.d/99-radv.conf ~/.config/environment.d/99-radv.conf.BACKUP
```
*If you get a “No such file or directory”-error, that’s because the file doesn’t exist, so there is nothing to backup. This is just for the case, that you already created and modified it.*

If anything goes wrong, you can restore it with the following command:
```bash
cp ~/.config/environment.d/99-radv.conf.BACKUP ~/.config/environment.d/99-radv.conf
```

If the file never existed, you can just delete the one that we have created using the following command:
```bash
rm -i ~/.config/environment.d/99-radv.conf
```

**Restoring from backup will undo any edits you made to this file, including changes from this guide.**

**Reboot your system afterwards to apply the changes.**
___

*This approach works for X11 and Wayland!*

Run the following command to create a systemd environment for your user apps:
```bash
mkdir -p ~/.config/environment.d
nano ~/.config/environment.d/99-radv.conf
```

Add the following content to it: 
```bash
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json
RADV_PERFTEST=aco
MESA_SHADER_CACHE_DIR=/home/<your_username>/.cache/mesa-shaders
```
**Don’t forget to change the username! $HOME and $USERNAME does not work in this case, since it doesn’t auto expand in this file! 
Use `echo $USERNAME` if you don’t know your username!**:

### 5.2.1 (Experimental) Enabling AMD User Queue
Setting `AMD_USERQ=1` enables individual command submission queues for the GPU.
This can reduce latency and improve FPS by bypassing the global kernel queue.
Add it to the end of the file if you want to give it a try. 

⚠️ **It is experimental. Just remove it if you encounter any issues with it turned on.**

Disclaimer:
- This is only recommended for Kernel versions **6.2** and above! 
- This tweak will increase VRAM usage slightly and may behave weird on older GPUs!
- It works on my GPU and thus should work if your GPU is newer!


Press `Ctrl + X` → `Y` → `Enter` to save the file when you are done.

## 5.3 Confirm that our changes took effect
Reboot your system and repeat step **3**. It should default to ACO now!

## 5.4 Deleting the old compiled shaders
⚠️ **Warning:** Always double-check the full path when running `rm -rf`.  
This command **recursively deletes everything** in the specified folder without asking for confirmation.  
**Accidentally deleting the wrong folder can cause serious system or data loss!  
Make sure to copy the WHOLE command to prevent this!**

Make sure that Steam is not running before continuing.
Run the following command to delete the old shaders compiled by LLPC/LLVM:
```bash
rm -rf ~/.steam/steam/steamapps/shadercache/730
rm -rf ~/.local/share/Steam/steamapps/shadercache/730
```
⚠️ **Warning:** This will delete **all compiled shaders for CS2**. Double-check the paths before running this command. The shaders will be compiled again when you launch the game.

## 5.5 Try running the game
Now you can start Steam again. When you launch CS2 it should start compiling the shaders. Don’t skip this, just wait!

Your game should now already run significantly faster! But we can squeeze even more performance out of it!
___
# 6. CS2 launch arguments
Im going to structure this in a way, where we “build” our launch arguments step by step. If you don’t want/need certain arguments, you can just skip them. Im going to provide the progress on how it should look, if you apply all my tweaks.

Open a notepad and add the launch arguments to it step by step. In the end, you can copy paste it into steam.

## 6.1 Fixing stuttering after 30-45 minutes on some systems
**LD_PRELOAD=""** 
This basically clears all the loaded libraries for your game (including the steam overlay). The steam overlay is known to cause stuttering after a certain time of playing the game, thats why we disable it.

**Example:**
```
LD_PRELOAD=""
```

## 6.2 Redundant but lets be on the safe side
**RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1**
Yes, we already set those variables system wide, but we can also set them here. Im going to include them, just as a “backup”.

**Example:**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
```

## 6.3 gamescope
***Gamescope** is a **micro-compositor** made by **Valve** (for Steam Deck & Linux gaming).  
Think of it as a **tiny, optimized version of a desktop compositor (like Wayland or X11)**, built **specifically for games**. (Thanks ChatGPT)*

These are the most important gamescope arguments:
```
-W 1920     : output window width (your native monitor width)
-H 1080     : output window height (your native monitor height)
-w 1440     : nested window width (this is the nested window width)
-h 1080     : nested window height (this is the nested window height)
-r 280      : nested refresh rate. Set this to your monitors refresh rate.
-S stretch  : This defines the scaler. Use stretch for stretched.
--immediate-flips    : Reduces latency by allowing tearing.
--adaptive-sync      : Add if you want to use Freesync (VRR)
--force-grab-cursor  : Use this, if your mouse isn't limited to the window

-- %command%         : THIS ENDS THE GAMESCOPE ARGUMENTS AND IS NEEDED!!
```

Example for 16:9 native on wayland:
**gamescope -W 1920 -H 1080 -w 1920 -h 1080 -r 280 -f --immediate-flips --force-grab-cursor -- %command%**

Example for 4:3 → 16:9 stretched on wayland:
**gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280  -S stretch -f --immediate-flips --force-grab-cursor -- %command%**

**Example:**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280  -S stretch -f --immediate-flips --force-grab-cursor -- %command%
```

## 6.4 Limiting the amount of cores used
This can improve P1 FPS for CPUs with high single core performance. 
Higher P1 FPS → smoother gameplay.

The following command forces CS2 to run only on 4 physical cores (this improves P1 FPS and reduces micro stutters).
`taskset -c 2,4,6,8`

If you are using gamescope, then insert this between the ending `--`and `%command%`

**Example (gamescope):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280  -S stretch -f --immediate-flips --force-grab-cursor --
taskset -c 2,4,6,8
%command%
```


**Example (without gamescope):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
taskset -c 2,4,6,8 %command%
```

## 6.5 Actual CS2 launch arguments
These are pretty much a no-brainer. Make sure to set the refresh rate to your monitors refresh rate.
``` 
-refresh 280 
+engine_low_latency_sleep_after_client_tick true 
+fps_max 0 
-nojoy 
-high 
+mat_disable_fancy_blending 1 
-forcenovsync 
+r_dynamic 0 
+mat_queue_mode 2 
+engine_no_focus_sleep 0 
-softparticlesdefaultoff
```

Make sure to add the following, if you added step **6.4**
```
-threads 4
```

If you didn’t add step **6.4** set the threads to the amount of threads, that your CPU has. You can check this by running the following command:
```bash
lscpu | grep 'CPU(s):'
```

In my case, I get `20`. So I set it to 20 threads.
``` 
-threads 20
```


**Example (with gamescope, with taskset):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280 -S stretch -f --immediate-flips --force-grab-cursor --
taskset -c 2,4,6,8
%command%
-refresh 280 
+engine_low_latency_sleep_after_client_tick true 
+fps_max 0 
-nojoy 
-high 
+mat_disable_fancy_blending 1 
-forcenovsync 
+r_dynamic 0 
+mat_queue_mode 2 
+engine_no_focus_sleep 0 
-softparticlesdefaultoff
-threads 4
```


**Example (without gamescope, without taskset):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
%command%
-refresh 280 
+engine_low_latency_sleep_after_client_tick true 
+fps_max 0 
-nojoy 
-high 
+mat_disable_fancy_blending 1 
-forcenovsync 
+r_dynamic 0 
+mat_queue_mode 2 
+engine_no_focus_sleep 0 
-softparticlesdefaultoff 
-threads 20
```

## 6.5 Autoexec
Of course, don’t forget to add your autoexec at the end, if you have one.
```
+exec autoexec
```

**Example (with gamescope, with taskset):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280 -S stretch -f --immediate-flips --force-grab-cursor --
taskset -c 2,4,6,8
%command%
-refresh 280 
+engine_low_latency_sleep_after_client_tick true 
+fps_max 0 
-nojoy 
-high 
+mat_disable_fancy_blending 1 
-forcenovsync 
+r_dynamic 0 
+mat_queue_mode 2 
+engine_no_focus_sleep 0 
-softparticlesdefaultoff
-threads 4
+exec autoexec
```


**Example (without gamescope, without taskset):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
%command%
-refresh 280 
+engine_low_latency_sleep_after_client_tick true 
+fps_max 0 
-nojoy 
-high 
+mat_disable_fancy_blending 1 
-forcenovsync 
+r_dynamic 0 
+mat_queue_mode 2 
+engine_no_focus_sleep 0 
-softparticlesdefaultoff 
-threads 20
+exec autoexec
```

## 6.6 Putting it all together!
We are almost done! Now make sure that every line ends with a space. Remove all the line breaks and you are done!

**Example (with gamescope, with taskset):**
```
LD_PRELOAD=""
RADV_PERFTEST=aco 
MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders
AMD_USERQ=1
gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280 -S stretch -f --immediate-flips --force-grab-cursor --
taskset -c 2,4,6,8
%command%
-refresh 280 
+engine_low_latency_sleep_after_client_tick true 
+fps_max 0 
-nojoy 
-high 
+mat_disable_fancy_blending 1 
-forcenovsync 
+r_dynamic 0 
+mat_queue_mode 2 
+engine_no_focus_sleep 0 
-softparticlesdefaultoff
-threads 4
+exec autoexec
```

Turns into:

**Example (with gamescope, with taskset):**
```
LD_PRELOAD="" RADV_PERFTEST=aco MESA_SHADER_CACHE_DIR=/home/$USER/.cache/mesa-shaders AMD_USERQ=1 gamescope -W 1920 -H 1080 -w 1440 -h 1080 -r 280 -S stretch -f --immediate-flips --force-grab-cursor -- taskset -c 2,4,6,8 %command% -refresh 280  +engine_low_latency_sleep_after_client_tick true +fps_max 0 -nojoy -high +mat_disable_fancy_blending 1 -forcenovsync +r_dynamic 0 +mat_queue_mode 2 +engine_no_focus_sleep 0 -softparticlesdefaultoff -threads 4 +exec autoexec
```

Save it somewhere, so you don’t lose it!

Paste it into Steam `Right Click on CS2` → `Properties` → `General` → and paste it into `Launch Options`.

Launch the game and enjoy the most optimal experience possible!
___
# Fix: gamescope causing VAC errors
*Clarification: The VAC error occurs because `gamescope` uses capabilities to adjust its process priority (nice values). Removing these capabilities prevents the error.*

## 1.1 Removing all nice capabilities for `gamescope`
Run the following command in a terminal:
```bash
sudo sh -c 'for f in $(whereis -b gamescope | cut -d" " -f2-); do [ -x "$f" ] && setcap -r "$f"; done'
```
*Note: This command removes all capabilities that allow `gamescope` to adjust its nice value for process priority.*

## 1.2 Confirm that the capabilities are removed
Run the following command in a terminal:
```bash
getcap /usr/bin/gamescope /usr/local/bin/gamescope
```
*Note: No output means capabilities have been removed successfully*

## 1.3 Reboot
**Reboot your system, for changes to take effect.**
___
# Wrapping it up
You’ve successfully:
- Ensured your system uses Mesa with ACO, not AMDVLK with LLPC/LLVM.
- Optimized shader caching for faster compilation and less stutter when loading them on the fly.
- Optionally enabled AMD_USERQ for lower latency.
- Set up Gamescope for clean frame pacing, lower latency, and fullscreen scaling.
- Optimized CPU core usage via taskset to prioritize physical cores for lower frame times and higher 1% lows.
- Fixed the VAC error caused by Gamescopes “nice” process capabilities.

Your game should now:
- Compile shaders ~10x faster.
- Maintain consistent FPS even in long playing sessions and intense scenarios.
- Feel really snappy and even more responsive than on Windows.

Enjoy gaming and I hope you learned something new along the way! 
Writing this guide took roughly 4 hours, so I truly appreciate your time reading it. If your game runs smoother and feels better after following these steps, don’t forget to **rate this guide** to help others find it!
