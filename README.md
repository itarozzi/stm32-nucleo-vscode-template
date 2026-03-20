STM32 Nucleo-F446RE Template for VSCodium and GNU toolchain
===========================================================

This repository provides a generic template for STM32 MCU development with the GNU toolchain, Makefile, and VSCodium/VSCode IDE.

The repository was created for the Nucleo-F446RE demoboard.

It describes the complete workflow for editing, compiling, and debugging an STM32 MCU program. Some concepts can, however, be adapted to other MCU types.

Software and tools used in this workflow:
- GNU Arm toolchain
- [stm32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)
- VSCode/VSCodium
- [OpenOCD](https://openocd.org/)
- [Bear](https://github.com/rizsotto/Bear)

I used this repo on Arch Linux but the same can be applied to other GNU/Linux distro.


> A special thank to my friend Enrico Marchesini for his support. This workflow is the result of his expertise and hard work.


## Prerequisites

- Install the arm toolchain
- Install stm32CubeMX
- VSCodium 
- Install openocd
- Install Bear

> in my Arch system I installed the listed packages from distro repo:
```
extra/arm-none-eabi-binutils 2.43-2 
extra/arm-none-eabi-gcc 14.2.0-2 
extra/arm-none-eabi-gdb 17.1-1 
extra/arm-none-eabi-newlib 4.5.0.20241231-2 
extra/openocd 1:0.12.0-5 
extra/bear 4.0.4-1
```


In VSCodium install the following add-ons:

- clangd
- Cortex-Debug

## Project Creation

Using STM32CubeMX select board/MCU and generate the project code, specifying the **Toolchain/IDE**: `Makefile`.

From the project directory, run `make` to verify that the toolchain is OK. This should compile and generate the `.elf` file.

In  VsCodium, open the project directory → the `.vscode` directory is created.

## VSCodium Configuration

You can use `.vscode/settings.json` file to define some custom variables to use in your tasks or IDE commands (for example to define the name of the application or of the firmware, the build directory...).

```json
{
  "FW_NAME": "nucleo_vscodium",
  // "BUILD_DIR": "./build"
  "BUILD_DIR": "/tmp/stm32_build"
}
```

Then you can refer to these variables (eg. in task file) using the syntax: `${config:FW_NAME}`.

> Note:
>
> Configuration variables are defined only in VSCode scope. For example, you can't use them directly in Makefile or shell commands.
> 
> For this reason you should use VSCodium Tasks to call make or shell script, passing configuration variables as parameters.


## Tasks

In VSCodium, **Tasks** allows you to perform a series of operations using external tools. These can be run in the IDE using menus or the CTRL-SHIFT-B shortcut.

In this case we'll use them to:
- call Bear to generate clangd configuration file
- build the project calling make
- flash the firmware to MCU calling make


Create a `.vscode/tasks.json` file. See the [template](.vscode/tasks.json) in this repository and adapt it to your preferences.

CTRL-SHIFT-B (Terminal → Run Build Task) to select the task to execute.

→ Verify that the make command executes correctly and compiles properly



## Flashing with OpenOCD

Flash the firmware to MCU using STM32 ST-LINK (in this case is embedded in the Nucleo board).

Create a symlink or copy the st_nucleo_f4.cfg file into the project directory 

> the cfg file depends on the Board/MCU

`ln -s /usr/share/openocd/scripts/board/st_nucleo_f4.cfg .`

Add the flash command to the `Makefile` (just to make it easier to compile using a single command and recompile if something has changed in the meantime)

```
flash: all
    openocd -f st_nucleo_f4.cfg -c "program $(BUILD_DIR)/$(TARGET).elf verify exit"
```


If it isn't already present, add a new task to `tasks.json` to call `make flash`.
```
{
"type": "shell",
"label": "Build release and flash",
"command": "make",
"args": ["flash", "-j$(nproc)", "DEBUG=0"],
"group": "build",
"detail": "Build all with arm-none-eabi-gcc and flash firmware ST Programmer"
}
```

Run the task and check that the Board is programmed correctly.

## Debug 

You can use ST-LINK and gdb to debug the firmware, using breakpoint and inspecting registers, memories and variables.

Create the  `.vscode/launch.json` file (see [template](.vscode/launch.json) in this repository).

Adapt it to your requirements:
- check the `executables` field is pointing  to your elf
- enable/disable `preLaunch` field to run a new `make flash` command before entering in debug
- check the `device` field for MCU model
- check the `configFiles` field is pointing to openocd configuration file 
- check the `svdFile` field is pointing to the file for your MCU

> I downloaded svd file from https://github.com/modm-io/cmsis-svd-stm32/tree/main/stm32f4

Press F5 to start the debugger. 

If everything works you should see the debugger stop at the first instruction of the main.


## Clangd and Bear

The VSCode clangd plugin ([clangd](https://github.com/clangd/vscode-clangd)) provide some useful features to help editing C/C++ code.

To allow clangd to index all the files of your project you can use the `bear` tool to generate a `compile_command.json` file.

Add a task to `tasks.json` to call bear (the first time and each time you add new files to your project).

```json
{
	"type": "shell",
	"label": "Bear - Build clangd json file",
	"command": "bear",
	"args": ["--output", "${workspaceFolder}/compile_commands.json",
			"--", "make", "rebuild_all"
			], 
	"group": "build",
	"detail": "Execute Bear utility to create compile_commands.json file"
},
```

Running the task should generate the `compile_commands.json` file and, after a few moments, the `.cache` directory in which clangd indexes everything.

From that moment, autocompletion features, function jumps, etc. should be activated.

> **Warning**: 
> If you run bear with the wrong Makefile target, or if you run it without a `make clean` , you'll end up with an empty JSON file that overwrites the previous one. This stops clangd from working!
>
>It's therefore a good idea to create a specific target in the Makefile for clean and then make, and always check the resulting `compile_commands.json` file.


## Tips and Tricks

When STM32cubeMX generates project directory, it creates a basic Makefile. 

Starting from it, some changes and optimizations have been applied, listed below:
- added `flash` target to flash firmware to MCU
- added `rebuild_all` target to use in conjunction with Bear utility
- added conditional sections to change optimizations based on DEBUG variable value
-  added conditional sections to change build directory based on `settings.json` variables and DEBUG variable value
- added the -p parameter to mkdir commands, to create complete paths




Other customization are:
- build to /tmp directory to reduce storage stress
- use Task to build firmware to debug/release subdirectory



## License and Credits

This projects uses the MIT License. See LICENSE file for details.

A big thank you to Enrico Marchesini who taught me the workflow to permits this repository to exists and supported me in my first steps on STM32 and VSCodium world.

Please feel free to share, improve or report inaccuracies.


