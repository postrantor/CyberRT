# Apollo(v9.0.0) CyberRT

## .bashrc

```bash
export MACHINE=cyber

export http_proxy=http://192.168.31.201:7891
export https_proxy=http://192.168.31.201:7891

export PATH=.:${HOME}/.local/bin:$PATH

# set workdir
export CURRENT_WORKDIR=${HOME}/project/cyber_ws/
export CYBER_WORKDIR=${CURRENT_WORKDIR}
source ${CURRENT_WORKDIR}/.env/bashrc
source ${CURRENT_WORKDIR}/.env/custom
```

## download fast-rtps

copy from apollo [dockerfile](apollo/docker/build/installers)

```bash
wget -t 10  https://apollo-system.cdn.bcebos.com/archive/6.0/fast-rtps-1.5.0-1.prebuilt.x86_64.tar.gz -P ./
wget -t 10  https://apollo-system.cdn.bcebos.com/archive/6.0/fast-rtps-1.5.0-1.prebuilt.aarch64.tar.gz -P ./
```

```shell
sudo apt update
sudo apt install -y uuid-dev libncurses5-dev python3-dev python3-pip
python3 -m pip install protobuf==3.14.0
```

```shell
sudo python3 install.py
# sudo python3 install.py --platform <your-platform-machine> --install_prefix <your-install-path>
```

```shell
source install/setup.bash
```

## Examples

1. pub/sub

```shell
source setup.bash
./cyber/examples/cyber_example_talker
```

```shell
source setup.bash
./cyber/examples/cyber_example_listener
```

2. component

```shell
source setup.bash
cyber_launch start share/examples/common_component_example/common.launch
./cyber/examples/common_component_example/channel_prediction_writer
./cyber/examples/common_component_example/channel_test_writer
```

3. log directory(optional)

**The Cyber log storage path is similar to `ROS` and is saved in `~/.cyber/log`**

If you want to modify the log storage path, you can change the `GLOG_log_dir` environment variable as follows:

```shell
# export GLOG_log_dir=/path/to/cyber/log
```

## Tools

1. channel

```shell
source setup.bash
cyber_channel list

# The number of channels is:  1
# /apollo/test
```

```shell
source setup.bash
cyber_channel echo /apollo/test
```

![example](docs/cyber_echo.png)

```shell
Commands:
	cyber_channel list	list active channels
	cyber_channel info	print information about active channel
	cyber_channel echo	print messages to screen
	cyber_channel hz	display publishing rate of channel
	cyber_channel bw	display bandwidth used by channel
	cyber_channel type	print channel type
```

2. node

```shell
Commands:
	cyber_node list 	List active nodes.
	cyber_node info 	Print node info.
```

3. service

```shell
Commands:
	cyber_service list	list active services
	cyber_service info	print information about active service
```

4. launch

```shell
cyber_launch start share/examples/common_component_example/common.launch
```

5. monitor

```shell
cyber_monitor
```

6. recorder

```shell
Commands:
  	cyber_recorder info	Show information of an exist record.
	cyber_recorder play	Play an exist record.
	cyber_recorder record	Record same topic.
	cyber_recorder split	Split an exist record.
	cyber_recorder recover	Recover an exist record.
```

## Packages

```shell
cmake -DCMAKE_INSTALL_PREFIX=/you/install/path ..
make -j$(nproc)
make package
sudo dpkg -i package/*.deb
```

```
# CMakeLists.txt
find_package(PkgConfig REQUIRED)
pkg_check_modules(Cyber REQUIRED cyber)
include_directories(
  ${Cyber_INCLUDE_DIRS}
)
link_directories(${Cyber_LIB_DIRS})
target_link_libraries(${TARGET_NAME}
  ${Cyber_LIBRARIES}
)
```
