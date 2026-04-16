<div align="center">
    <h1 align="center">Go2 RL Deploy</h1>
</div>
This project deploys Reinforcement Learning (RL) policies on the Unitree Go2 Edu robot (equipped with Orin NX). The code is modified from [unitree_rl_lab](https://github.com/unitreerobotics/unitree_rl_lab) and supports deployment through Ethernet or the onboard computer.

## Directory Structure
```
go2_deploy/
├── go2/                # Main program, CMakeLists, and config for Go2 robot          
├── include/            # Common headers (FSM, Isaac Lab interfaces, etc.)
├── policy/             # Directory for trained RL policy models
├── resources/          # Robot assets such as urdf and related files
├── thirdparty/         # Third-party libraries (onnxruntime, json)
└── README.md
```

## Dependencies
Before compiling and running, ensure the following dependencies are installed in the development environment (Orin NX):
```bash
sudo apt install libboost-program-options-dev libyaml-cpp-dev libeigen3-dev libfmt-dev libspdlog-dev
```
- **[unitree_sdk2](https://github.com/unitreerobotics/unitree_sdk2)**: SDK for Unitree robot development.
- **Boost**: For program options parsing (`program_options`).
- **yaml-cpp**: For reading configuration files.
- **Eigen3**: Matrix operation library.
- **fmt**: Formatting library.
- **spdlog**: Logging library.
- **onnxruntime**:
   - x64 Linux: Download [onnxruntime-linux-x64-1.23.2.tgz](https://github.com/microsoft/onnxruntime/releases/download/v1.23.2/onnxruntime-linux-x64-1.23.2.tgz) and extract it to the `deploy/thirdparty/` folder.
   - Orin NX: Download [onnxruntime-linux-aarch64-gpu-1.16.0.tar.bz2](https://github.com/csukuangfj/onnxruntime-libs/releases/download/v1.16.0/onnxruntime-linux-aarch64-gpu-1.16.0.tar.bz2) and extract it to the `deploy/thirdparty/` folder. Modify the ONNX link path in [{ROBOT}/CMakeLists.txt](deploy/robots/go2/CMakeLists.txt) by uncommenting the corresponding lines.

> If you deploy on the Orin NX onboard computer, install the aarch64 version of onnxruntime. If you deploy from an x64 Linux computer through Ethernet, install the x64 version and adjust the ONNX link path accordingly.

## Compilation Steps
1. Enter the Go2 deployment directory:
   ```bash
   cd go2
   ```
2. Create a build directory:
    ```bash
   mkdir build && cd build
   ```
3. Run CMake and compile:
    ```bash
   cmake .. && make -j8
   ```
## Running Guide
After compilation, run the generated `go2_ctrl` executable in the `build` directory.

#### Command Line Arguments

- `-h, --help`: Show help message.
- `-v, --version`: Show version information.
- `--log`: Enable logging (logs will be saved in `deploy/robots/go2/log/`).
- `-n, --network <interface>`: Specify the network interface name for DDS communication (e.g., `eth0`, `wlan0`). If not specified, the default interface will be used.

#### Real Robot Launch
Start the Go2 robot. After it enters the standing state, press **[L2 + A]** twice to make the robot lie down. Connect with the mobile app, then go to Settings -> Service Status, disable `mcf/*` services, and close the official control program to avoid control conflicts.

**Example:**

you can run `ifconfig` to check your network connection, for example, the lower-level network interface is `eth0`:
```bash
./go2_ctrl -n eth0
```

### Operation Flow
1. After startup, the console displays "Waiting for connection to robot..."
2. Ensure the robot is powered on and connected. After a successful connection, the console displays "Connected to robot."
3. **Enter Stand Mode**: Press **[L2 + A]** on the gamepad. The robot will enter `FixStand` mode and stand up.
4. **Start RL Control**: Press **[Start + Up/Down/Left/Right]** on the gamepad. The robot switches to the corresponding `Velocity_[UP/DOWN/LEFT/RIGHT].policy_dir` model and starts executing the RL policy. The default configuration uses the Up/Down/Left policy.
5. **Switch Model**: During operation, you can switch to different RL models at any time by pressing **[Start + Direction Key]**.
6. **Fixed Command Execution**: Press **[L2 + Y]** on the gamepad. The robot will start executing preset fixed commands (as set in the config file). Press the combination again to stop fixed command execution.
7. **Enter Damping Mode**: Press **[L2 + B]** on the gamepad. The robot will enter damping mode and stop RL control.

## Configuration
The configuration file is located at [go2/config/config.yaml](./go2/config/config.yaml). It includes:
1. **Multi-model selection**: Configure `Velocity/policy_dir_up/down/left/right` to specify paths for four models. Switch models using `Start + Direction Key`.

### Changing Policy Models
To change the RL policy, modify the `Velocity_[UP/DOWN/LEFT/RIGHT].policy_dir` fields in `config.yaml`:
```yaml
Velocity_Up:
  # Policy model path (relative to the project root or an absolute path)
  policy_dir: ../policy/example
```
The specified directory structure should contain:
- `exported/policy.onnx`: Exported ONNX policy model.
- `params/deploy.yaml`: Corresponding deployment parameters.

### Modifying Control Parameters

You can also adjust PD parameters (`kp`, `kd`) and target joint angles (`qs`) for `FixStand` mode in `config.yaml`.

### Modifying Unexpected Termination Parameters

The control program terminates if the angle between the `base_link` Z axis and gravity exceeds the threshold. The default value is 2 rad.

Find `bad_orientation` in [State_RLBase.cpp](go2/src/State_RLBase.cpp) and modify the second parameter to change the radian threshold.