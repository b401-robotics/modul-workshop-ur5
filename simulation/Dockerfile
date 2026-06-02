# Use the official ROS 2 Humble desktop base (includes robot_state_publisher)
FROM ros:humble

# Set environment variables to prevent interactive prompts during installation
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies, UR Driver, and Foxglove Bridge
RUN apt-get update && apt-get install -y \
    ros-humble-ur-robot-driver \
    ros-humble-foxglove-bridge \
    ros-humble-ur-simulation-gz \
    && rm -rf /var/lib/apt/lists/*

# Source the ROS 2 setup bash script automatically for every bash session
RUN echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc

# Set up the working directory
WORKDIR /ros2_ws

# Command to run by default if none is provided
CMD ["bash"]