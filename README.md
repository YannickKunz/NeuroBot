# NeuroBot 🧠

This project uses a Muse 2 headset and BrainFlow to capture EEG data, train machine learning models, and perform real-time classification of 5 psychological states (Calm, Focused, Happy, Stressed), which then drive the physical behavior of a TurtleBot3, a generative audio soundscape (Reaper), and an ambient lighting system (ESP32) for the scope of a theatre play.

## 📂 Project Structure

- **`muse/`**: The Brain. Contains the BCI pipeline, data recording, cleaning, visualization, and Machine Learning models (`new_train.py`, `eegnet.py`).
- **`ros_ws/`**: The Body. Contains the Robot Operating System workspace.
  - `src/neuro_rover/scripts/`: Contains the main control logic (`neuro_controller.py`).
  - `src/neuro_rover/launch/`: Simulation launch files.
  - `src/neuro_rover/worlds/`: Custom Gazebo simulation worlds.
- **`IO/`**: The Senses.
  - `arduino/`: C++ code for the ESP32 LED controller.
  - `reaper/`: Reaper DAW project files (.RPP) for generative audio.
- **`misc/`**
---

## 🔌 Setup & Installation

### 1. Python Environment (BCI)
Navigate to the root directory where `requirements.txt` is located and create and activate a virtual environment:

```bash
python -m venv venv
# Linux/Mac
source venv/bin/activate
# Windows
.\venv\Scripts\Activate.ps1
```
Then, install the required dependencies:
```bash
pip install -r requirements.txt
```

---

### 2. 🤖 ROS Workspace (Robot)

The robot behavior is handled by the `neuro_rover` package inside the `ros_ws` directory.

#### 1. Build the Workspace
Before running the robot, you must compile the ROS package.

```bash
cd ros_ws
catkin_make
source devel/setup.bash
```
*Note: You must run `source devel/setup.bash` in every new terminal you open.*

#### 2. Launch the Simulation (Gazebo + Controller)
This command launches the 3D world, spawns the robot, and starts the **Neuro-Empathic Controller**.

**⚠️ Graphics Fix (For WSL/Linux):**
If you see a black screen in Gazebo, run this command *before* launching:
```bash
export LIBGL_ALWAYS_SOFTWARE=1
```

**Launch Command:**
```bash
roslaunch neuro_rover neuro_behavior.launch
```
* **Default Behavior:** The robot will enter "Neutral Orbit" (circling the center point at 1m radius).
* **Safety:** The robot will automatically stop/turn if it detects an obstacle < 0.5m.

#### 3. Visualization (RViz)
If Gazebo is black or you want to see the LIDAR data:

1.  Open a new terminal.
2.  Run:
    ```bash
    roslaunch turtlebot3_bringup turtlebot3_model.launch
    ```
3.  **Configuration:**
    * Set **Fixed Frame** to `odom`.
    * Add **RobotModel**.
    * Add **LaserScan** (Topic: `/scan`, Reliability: Best Effort, Size: 0.1).

#### 4. Controlling States (Testing)
You can manually trigger emotional states to test the choreography.

**Method A: Graphical Interface (RQT)**
1.  Run `rqt` in a terminal.
2.  Go to **Plugins** -> **Topics** -> **Message Publisher**.
3.  Select topic `/neuro_state`.
4.  Set expression to `'happy'`, `'focused'`, `'stressed'`, or `'neutral'`.
5.  Check the box to start publishing.

**Method B: Command Line**
```bash
# Trigger Happy (Wiggle/Figure-8)
rostopic pub /neuro_state std_msgs/String "data: 'happy'" -1

# Trigger Stressed (Erratic)
rostopic pub /neuro_state std_msgs/String "data: 'stressed'" -1

# Trigger Calm (Slow Orbit)
rostopic pub /neuro_state std_msgs/String "data: 'calm'" -1

# Trigger Focused (Face Center)
rostopic pub /neuro_state std_msgs/String "data: 'focused'" -1

# Reset Center Point (Set current location as new orbit center)
rostopic pub /reset_center std_msgs/String "data: 'now'" -1

# Trigger Neutral (Standard Orbit)
rostopic pub /neuro_state std_msgs/String "data: 'neutral'" -1
```

**Method C: Command Line**

Open the node-red flow and inject the states manually.

#### 5. Manual Driving
To take over control manually (WASD keys):
```bash
roslaunch turtlebot3_teleop turtlebot3_teleop_key.launch
```

### 3. 🔀 Middleware (Node-RED) ###

Node-RED bridges the Python BCI script (MQTT), the Robot (ROS), and the Audio Engine (OSC).

1. Installation & Setup
```bash
npm install -g --unsafe-perm node-red
npm install node-red-dashboard
npm install node-red-contrib-ros
npm install node-red-contrib-osc
 ```
2. Launching
   In a new terminal on the laptop:
```bash
   # 1. Start the ROS Bridge (Required for Node-RED to talk to ROS)
roslaunch rosbridge_server rosbridge_websocket.launch

# 2. Start Node-RED
node-red
```
3. Configuration
   
* Open http://localhost:1880.
Import the flow file located at muse/node-red.json.
* Verify Configuration:
    - MQTT Nodes: Ensure the Server is set to localhost (127.0.0.1) port 1883.
    - ROS Nodes: Ensure the Server is set to ws://localhost:9090.
    - UDP/OSC Nodes: Ensure the destination is 127.0.0.1 port 8000.
---

## 🧠 BCI Pipeline (The Brain): EEGNet (PyTorch)

This is the primary BCI workflow using the EEGNet deep learning model. All BCI logic is located in the muse/ directory. The main entry point is new_train.py.

### 1. Record Training Data
Specify a **user ID** (e.g., `yk`) and emotion to capture EEG data for specific states to train your user-specific model.


```bash
# Record 30 seconds of 'calm'
python ml_emotion_class.py record yk calm --duration 30 --count 5
```
Repeat for `focused`, `happy`, etc. Saves the data under .csv format.

### 2. Train the Model
Train an EEGNet (Deep Learning) on your recorded data.
```bash
python eegnet.py train yk
```
Saves model to `models/yk/emotion_model_eegnet.pth`.

### 3. Run Live Prediction

```bash
python new_train.py predict <user_id> --model_type eegnet --mqtt_ip 127.0.0.1
```
This streams EEG data from the headband, predicts the state, and publishes it via MQTT to the robot and audio systems.

---

## 🧩 Alternate Workflow: Classic ML

Uses RandomForest or SVM on extracted features.

1.  **Record:** Same as above (`ml_emotion_class.py record ...`).
2.  **Train:**
    ```bash
    python ml_emotion_class.py train yk --model_type rf
    ```
3.  **Predict:**
    ```bash
    python ml_emotion_class.py predict yk --model_type rf
    ```

---

## 📊 Utilities

**Live Signal Check:**
View raw EEG stream without prediction.
```bash
python live_fft_muse.py
```

**Data Visualization:**
To view recorded sessions (change user initials in script line 14 first):
```bash
python data_visualiser.py
```

## 🤖 Robot Control (The Body)

When moving from the Gazebo simulation to the physical TurtleBot3 (e.g., at the university), you must reconfigure the network settings to ensure the Laptop ("Brain") and Robot ("Body") can communicate.

### 1. Network Prerequisites
* **Wi-Fi:** Ensure both the Laptop and the Robot are connected to the same network (e.g., `ProFab`).
* **Firewalls:** Disable firewalls on your Laptop (Windows/WSL), or ROS messages will be blocked.

### 2. IP Configuration 
Every time your Laptop connects to a new network, its IP address changes. You must update **both** devices.

**A. On the Laptop:**
1.  Find your IP: `ifconfig` (look for `inet` under `wlan0` or `eth0`).
2.  Edit `.bashrc`: `nano ~/.bashrc`
3.  Update these lines with your **Laptop's IP**:
    ```bash
    export ROS_MASTER_URI=http://<YOUR_LAPTOP_IP>:11311
    export ROS_HOSTNAME=<YOUR_LAPTOP_IP>
    ```
4.  Apply changes: `source ~/.bashrc`

**B. On the Robot:**
1.  SSH into the robot: `ssh ubuntu@<ROBOT_IP>` (Check label on robot for its IP).
2.  Edit `.bashrc`: `nano ~/.bashrc`
3.  Update **ROS_MASTER_URI** to point to your **Laptop's IP**:
    ```bash
    export ROS_MASTER_URI=http://<YOUR_LAPTOP_IP>:11311
    # Leave ROS_HOSTNAME as the robot's own IP
    ```
4.  Apply changes: `source ~/.bashrc`

### 3. Launch Sequence

#### A. Running in Simulation (Gazebo) ####
```bash
cd ros_ws
source devel/setup.bash
# Launches Gazebo world + Neuro Controller
roslaunch neuro_rover neuro_behavior.launch
```

#### B. Running on Physical Robot ####
The startup order is strict. Run these in separate terminals:

1.  **Laptop (Terminal 1):** Start the Master.
    ```bash
    roscore
    ```
2.  **Robot (SSH Terminal):** Start the robot drivers.
    ```bash
    roslaunch turtlebot3_bringup turtlebot3_robot.launch
    ```
    *(Wait until you see "Calibration End" or connection success).*
3.  **Laptop (Terminal 2):** Start the Controller.
    ```bash
    rosrun neuro_rover neuro_controller.py
    ```
    **⚠️ Important:** Do **not** use `roslaunch neuro_behavior.launch` here. That file starts Gazebo, which conflicts with the real robot. Use `rosrun` to start only the Python script.
---
## Model Training ##

### Phase 0: Preparation (Every Session)
1.  **Environment:** Sit comfortably. Use noise-canceling headphones.
2.  **Signal Check:** Run `python live_fft_muse.py`.
    * *Test:* Close your eyes and relax.
    * *Pass Condition:* You must see a clear **Alpha peak (8-12 Hz)** rise on the graph (especially TP9/TP10).
3.  **Hydration:** If the signal is noisy ("rails"), wipe your forehead and mastoids (behind ears) with a damp cloth.



### Phase 1: Calm
* **Target:** High Alpha/Theta waves. Low Arousal.
* **Command:** `python ml_emotion_class.py record <user> calm --duration 60`
* **Eyes:** **CLOSED**.
* **Task:** Box breathing (In 4s, Hold 4s, Out 4s). Focus entirely on the sensation of your breath.
* **Physiology:** Relax your jaw. Let your shoulders drop.



### Phase 2: Focused
* **Target:** High Beta waves. High Arousal.
* **Command:** `python ml_emotion_class.py record <user> focused --duration 60`
* **Eyes:** **OPEN**.
* **Task:** Solve complex math problems in your head (e.g., "1000 minus 7, minus 7...") or read a dense academic paper.
* **Physiology:** **Do not frown.** Keep your gaze steady on one point; avoid wild scanning or rapid blinking.



### Phase 3: Happy
* **Target:** Frontal Alpha Asymmetry (Right > Left). Positive Valence.
* **Command:** `python ml_emotion_class.py record <user> happy --duration 60`
* **Eyes:** **OPEN** (watching video).
* **Task:** Watch a compilation of stand-up comedy or funny clips.
* **Physiology:** **Internalize the laugh.** Do not laugh out loud or grin widely (this creates muscle noise). Aim for a "Duchenne smile" (smiling with eyes) while keeping the jaw relaxed.



### Phase 4: Stressed
* **Target:** High Beta/Gamma. Negative Valence.
* **Command:** `python ml_emotion_class.py record <user> stressed --duration 60`
* **Eyes:** **OPEN**.
* **Task:** Perform a **Stroop Test** under time pressure, or listen to an annoying sound (e.g., alarm clock) while trying to ignore it.
* **Physiology:** **Watch your jaw.** Stress causes natural teeth clenching. Keep your mouth slightly open or tongue on the roof of your mouth to prevent EMG artifacts.


## 🎨 IO Systems (Lights & Sound) ##

### Lights (ESP32) ###
* Location: IO/arduino/
* Setup: Flash the code to the ESP32. It connects via Wi-Fi and listens to MQTT topics sent by the BCI script/Node-RED.
* States: Happy (Green Chase), Calm (Blue Pulse), Focused (Purple Pulse), Stressed (Red Strobe).

### Sound (Reaper) ###
* Location: IO/reaper/
* Setup: Open the .RPP project file on the host laptop.
* Logic: Receives OSC messages derived from the EEG stream to modulate VST plugin parameters (Filter cutoff, Distortion drive, Delay time) in real-time.
  


## 📐 System Pipeline ##

1. **Sensing**:  
   Muse S — *(Bluetooth)* → Python Script (`new_train.py`)

2. **Processing**:  
   Python Script — *(MQTT)* → Node-RED

3. **Actuation (Robot)**:  
   Node-RED — *(WebSockets)* → ROS Master — *(Wi-Fi)* → TurtleBot3 (`neuro_controller.py`)

4. **Actuation (Audio)**:  
   Node-RED — *(OSC / UDP)* → REAPER — *(Bluetooth)* → JBL Speaker

5. **Actuation (Light)**:  
   Node-RED — *(MQTT)* → ESP32
