# Threat Model analysis for `MARA` robot

*Developed in collaboration with [Alias Robotics](http://aliasrobotics.com).*

<a href="http://www.acutronicrobotics.com"><img src="https://acutronicrobotics.com/assets/images/AcutronicRobotics_logo.jpg" align="left" hspace="8" vspace="2" width="200"></a>

**A threat model is a structured representation of all the information that affects the security of a robot in a particular  application**. Threat modeling allows capturing, organizing, and analyzing all of this information while enabling informed decision-making about application security risks. In addition to producing a model, threat modeling allows to identify, enumerate and prioritize structural vulnerabilities – all from a hypothetical attacker’s point of view.

When applied to robotics, the ultimate purpose of threat modeling is to provide manufacturers, integrators and end-users with a systematic analysis of the probable attacker’s profile, the most likely attack vectors, and the assets most desired by an attacker. Typically, threat models answers the following questions:
- Where are the high-value assets?
- Where am I most vulnerable to attack?
- What are the most relevant threats? and
- Is there an attack vector that might go unnoticed?

The sections below cover the threat model analysis of our [MARA](https://acutronicrobotics.com/products/mara/) robot.

## Agenda

- [Agenda](#agenda)
	- [System description](#system-description)
	- [Architecture Dataflow diagram](#architecture-dataflow-diagram)
		- [Assets MARA](#assets-mara)
			- [Hardware](#hardware)
			- [Network](#network)
			- [Software processes](#software-processes)
			- [Software dependencies](#software-dependencies)
			- [External Actors](#external-actors)
			- [Robot Data assets](#robot-data-assets)
		- [Use case scenarios](#use-case-scenarios)
		- [Entry points for MARA](#entry-points-for-mara)
		- [Trust Boundaries for MARA in `pick & place` application](#trust-boundaries-for-mara-in-pick-place-application)
	- [Threat Model for MARA](#threat-model-for-mara)
		- [MARA Attack Trees](#mara-attack-trees)
		- [Physical vector attack tree](#physical-vector-attack-tree)
		- [ROS2 API vector attack tree](#ros2-api-vector-attack-tree)
		- [H-ROS API vector attack tree](#h-ros-api-vector-attack-tree)
		- [Code repository compromise vector attack tree](#code-repository-compromise-vector-attack-tree)
	- [Threat Model Validation Strategy for MARA](#threat-model-validation-strategy-for-mara)
	- [MARA Security Assessment preliminary results](#mara-security-assessment-preliminary-results)
		- [Introduction](#introduction)
		- [Results](#results)
			- [Findings](#findings)
- [References](#references)




### System description
The application considered in this section is a **[MARA][mara_robot] modular robot operating on an industrial environment while performing a pick & place activity**. MARA is the first robot to run ROS 2 natively. It is an industrial-grade collaborative robotic arm which runs ROS 2.0 on each joint, end-effector, external sensor or even on its industrial controller. Throughout the H-ROS communication bus, MARA empowers new possibilities and applications in the professional landscape of robotics. It offers millisecond-level distributed bounded latencies or submicrosecond-level synchronization capabilities.

Built out of individual modules that natively run on ROS 2, MARA can be physically extended in a seamless manner. However, this also moves towards more networked robots and production environments which brings new challenges, especially in terms of [security and safety][safe_sec].

The robot considered is a [MARA][mara_datasheet], a 6 Degrees of Freedom (6DoF) modular and collaborative robotic arm with 3 kg of payload and repeatibility below 0.1 mm. The robot can reach angular speeds up to 90º/second and has a reach of 656 mm. The robot contains a variety of sensors on each joint
and can be controlled from any industrial controller that supports ROS 2 and uses the [HRIM][hrim] information model. Each of the modules contains the [H-ROS communication bus][hros] for robots enabled by the [H-ROS SoM][hrossom], which delivers real-time, security and safety capabilities for ROS 2 at the module level.

No information is provided about how ROS 2 nodes are distributed on each module. Each joint offers the following ROS 2 API capabilities as described in their [documentation (MARA joint)][mara_joint_ros2_api]:
* **Topics**
  * `GoalRotaryServo.msg` model allows to control the position, velocity or/and effort (generated from [models/actuator/servo/topics/goal.xml](https://github.com/AcutronicRobotics/HRIM/blob/master/models/actuator/servo/topics/goal.xml), see [HRIM][hrim] for more).
  * `StateRotaryServo.msg` publishes the status of the motor.
   * `Power.msg` publishes the power consumption.
   * `Status.msg`  informs about the resources that are consumed, SpecsCommunication.msg and StateCommunication.msg that are created to inform about communication aspects.
   * `StateCommunication.msg` is a custom message which reports the state of the device network.

* **Services**
  * `ID.srv` publishes the general identity of the component.
  * `Simulation3D.srv` and `SimulationURDF.srv`.msg that sends the device 3D model and related information.
  * `SpecsCommunication.srv` is a custom message which reports the specs of the device network.
  * `SpecsRotaryServo.srv` is a custom message which reports the main features of the device.
  * `EnableDisable.srv` disables or enables the servo motor.
  * `ClearFault.srv` sends a request to clear any fault in the servo motor.
  * `Stop.srv` requests to stop any ongoing movement.
  * `OpenCloseBrake.srv` opens or closes the servo motor brake.

* **Actions**:
  * `GoalJointTrajectory` allows to move the joint using a trajectory msg.

Such API gets translated into the following abstractions:

| Topic | Name                                                |
| ----- | --------------------------------------------------- |
| goal  | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/goal_axis1  |
| goal  | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/goal_axis2  |
| state | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/state_axis1 |
| state | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/state_axis2 |

<br>

| Service       | Name                                                      |
| ------------- | --------------------------------------------------------- |
| specs         | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/specs             |
| enable servo  | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/enable            |
| disable servo | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/disable           |
| clear fault   | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/clear_fault       |
| stop          | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/stop_axis1        |
| stop          | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/stop_axis2        |
| close brake   | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/close_brake_axis1 |
| close brake   | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/close_brake_axis2 |
| open brake    | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/open_brake_axis1  |
| open brake    | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/open_brake_axis2  |

<br>

| Action                | Name                                                     |
| --------------------- | -------------------------------------------------------- |
| goal joint trajectory | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/trajectory_axis1 |
| goal joint trajectory | /hrim_actuator_rotaryservo_XXXXXXXXXXXX/trajectory_axis2 |

<br>

| Parameters            | Name                                                     |
| --------------------- | -------------------------------------------------------- |
| ecat_interface | Ethercat interface |
| reduction_ratio | Factor to calculate the position of the motor |
| position_factor | Factor to calculate the position of the motor |
| torque_factor | Factor to calculate the torque of the motor |
| count_zeros_axis1 | Axis 1 absolute value of the encoder for the zero position |
| count_zeros_axis2 | Axis 2 absolute value of the encoder for the zero position |
| enable_logging| Enable/Disable logging |
| axis1_min_position | Axis 1 minimun position in radians |
| axis1_max_position | Axis 1 maximun position in radians |
| axis1_max_velocity | Axis 1 maximun velocity in radians/s |
| axis1_max_acceleration | Axis 1 maximun acceleration in radians/s^2 |
| axis2_min_position | Axis 2 minimun position in radians |
| axis2_max_position | Axis 2 maximun position in radians |
| axis2_max_velocity | Axis 2 maximun velocity in radians/s |
| axis2_max_acceleration | Axis 2 maximun acceleration in radians/s^2 |

<br>

The controller used in the application is a ROS 2-enabled industrial PC, specifically, an [ORC][orc]. This PC also features the H-ROS communication bus for robots which commands MARA in a deterministic manner. This controller makes direct use of the aforementioned communication abstractions (topics, services and actions).


### Architecture Dataflow diagram

![ROS 2 Application](ros2_threat_model/MARA-ROS2_Application.png)
[Diagram Source (draw.io)](ros2_threat_model/ROS2_Application.xml)


#### Assets MARA
This section aims for describing the components and specifications within the MARA robot environment. Below the different aspects of the robot are listed and detailed in Hardware, Software and Network. The external actors and Data assests are described independently.

##### Hardware

* 1x [**MARA modular robot**][mara_robot] is a modular manipulator, ROS 2 enabled robot for industrial automation purposes.
  * **Robot modules**
    * 3 x [Han's Robot Modular Joints][hans_modular_joint]: 2 DoF electrical motors that include precise encoders and an electro-mechanical breaking system for safety purposes. Available in a variety of different torque and size combinations, going from 2.8 kg to 17 kg weight and from 9.4 Nm to 156 Nm rated torque.
      * Mechanical connector: [H-ROS connector A][hros_connector_A]
      * Power input: 48 Vdc
      * Communication: H-ROS robot communication bus
        * Link layer: 2 x Gigabit (1 Gbps) TSN Ethernet network interface
        * Middleware: Data Distribution Service (DDS)
      * On-board computation: Dual core ARM® Cortex-A9
      * Operating System: Real-Time Linux
      * ROS 2.0 version: Crystal Clemmys
      * Information model: [HRIM][hrim] Coliza
      * Security:
        * DDS crypto, authentication and access control plugins
    * 1 x [Robotiq Modular Grippers][robotiq_modular_gripper]: ROS 2 enabled industrial end-of-arm-tooling.
      * Mechanical connector: [H-ROS connector A][hros_connector_A]
      * Power input: 48 Vdc
      * Communication: H-ROS robot communication bus
        * Link layer: 2 x Gigabit (1 Gbps) TSN Ethernet network interface
        * Middleware: Data Distribution Service (DDS)
      * On-board computation: Dual core ARM® Cortex-A9
      * Operating System: Real-Time Linux
      * ROS 2.0 version: Crystal Clemmys
      * Information model: [HRIM][hrim] Coliza
      * Security:
        * DDS crypto, authentication and access control plugins
* 1 x [**Industrial PC**: ORC][orc] include:
  * CPU: Intel i7 @ 3.70GHz (6 cores)
  * RAM: Physical 16 GB DDR4 2133 MHz
  * Storage: 256 GB M.2 SSD interface PCIExpress 3.0
  * Communication: H-ROS robot communication bus
    * Link layer: 2 x Gigabit (1 Gbps) TSN Ethernet network interface
    * Middleware: Data Distribution Service (DDS)
  * Operating System: Real-Time Linux
  * ROS 2.0 version: Crystal Clemmys
  * Information model: [HRIM][hrim] Coliza
  * Security:
    * DDS crypto, authentication and access control plugins
* 1 x **Update Server: OTA**
  * Virtual Machine running on AWS EC2
    * Operating System: Ubuntu 18.04.2 LTS
    * Software: Mender OTA server

##### Network

1. **Ethernet time sensitive (TSN) internal network**: Interconnection of modular joints in the MARA Robot is performed over a daisy chained Ethernet TSN channel. Each module acts as a switch, forming the internal network of the robot.
2. **Manufacturer (Acutronic Robotics) corporate private network**: A secure corporate wide-area network, that spans
  multiple cities. Only authenticated user with suitable credentials can connect
  to the network, and good security practices like password rotation are in
  place. This network is used to develop and deploy updates in the robotic systems. Managed by the original manufacturer.
1. **End-user corporate private network**: A secure corporate wide-area network, that spans
  multiple cities. Only authenticated user with suitable credentials can connect
  to the network, and good security practices like password rotation are in
  place.
2. **Cloud network**: A VPC network residing in a public cloud platform, containing multiple servers. The network follows good security practices, like implementation of security applicances, user password rotation and multi-factor authentication. Only allowed users can access the network. The OTA service is open to the internet. Uses of the cloud network:
     - Manufacturer (Acutronic Robotics) uploads OTA artifacts from their `Manufacturer corporate private network` to the Cloud Network.
     - Robotic Systems on the `End-user corporate private network` fetch those artifacts from the Cloud Network.
     - Robotic Systems on the  `End-user corporate private network` send telemetry data for predictive maintenance to Cloud Network.



##### Software processes
In this section all the processes running on robotic system in scope are detailed.

* **Onboard Modular Joints**
  * `hros_servomotor_hans_lifecyle_node` this node is in charge of controlling the motors inside the H-ROS SoM. This nodes exposes some services, actions and topics described below. This node publishes joint states and joint velocities, it provides a topic for servoing the modular joint and an action that will be waiting for a trajectory.
  * `hros_servomotor_hans_generic_node` a node which contains several generic services and topics with information about the modular joint, such as `power` measurement readings: voltage and current, `status` about the H-ROS SoM like cpu load or network stats, specifications about `communication` and `cpu`, the `URDF` or even the mesh file of the modular joint.
  * Mender (OTA client) runs inside the H-ROS SoM. When an update is launched, the client downloads this version and it gets installed and available when the device is rebooted.
* **Onboard Modular Gripper**: *Undisclosed*.
* **Onboard Industrial PC**:
    - MoveIt! motion planning framework.
    - Manufacturing process control applications.
    - Robot teleoperatiopn utilities.
    - **ROS1/ROS2 bridges**: These bridges are needed to be able to run MoveIT! which is not yet ported to ROS 2.0. Right now there is an [effort in the community](https://acutronicrobotics.com/news/ros-2-moveit-robotic-motion-planning/) to port this tool to ROS 2.0.

##### Software dependencies

In this section all the relevant third party software dependencies used within the different components among the scope of this threat model are listed.

* Linux OS / Kernel
* ROS 2 core libraries
* H-ROS core libraries and packages
* ROS 2 system dependencies as defined by rosdep:
  * RMW implementation: In the H-ROS API there is the chance for selecting the DDS implementation.
  * The threat model describes attacks with the security enabled and disabled. If the security is enabled, the security plugins are assumed to be configured and enabled.

See [Mara ROS 2 Tutorials](https://acutronicrobotics.com/docs/products/robots/mara/tutorials) to find more details about software dependencies.

##### External Actors
All the actors interacting with the robotic system are here gathered.
* End users
  * Robotics user: Interacts with the robot for performing work labour tasks.
  * Robotics operator: Performs maintainance tasks on the robot. Integrates the robot with the industrial network.
  * Robotics researcher: Develops new applications and algorithms for the robot.
* Manufacturer
  * Robotics engineer: Develops new updates and features for the robot itself. Performs in-site robot setup and maintainance tasks.

##### Robot Data assets
In this section all the assets storing information within the system are displayed.
* ROS 2 network (topic, actions, services information)
  * Private Data
    * Logging messages
      * Restricted Data
    * Robot Speed and Orientation. Some understanding of the current robot task may be reconstructed from those messages.
* H-ROS API
  * Configuration and status data of hardware.
* Modules (joints and gripper)
  * ROS logs on each module. Private data, metrics and configuration information that could lead to GDPR issues or disclosure of current robot tasks.
  * Module embedded software (drivers, operating system, configuration, etc.)
    * Secret data. Intellectual Property (IP) theft is a critical issue here.
  * Credentials. CI/CD, enterprise assets, SROS certificates.
* ORC Industrial PC
  * Public information. Motion planning algorithms for driving the robot (MoveIt! motion planning framework).
  * Configuration data. Private information with safety implications. Includes configuration for managing the robot.
  * Programmed routines. Private information with safety implications.
  * Credentials. CI/CD, enterprise assets, SROS certificates.
* CI/CD and OTA subsystem
  * Configuration data.
  * Credentials
  * Firmware files and source code. Intellectual property, both end-user and manufacturer.
*  Robot Data
    * System and ROS logs are stored on each joint module's filesystem.
    * Robot system is subject to physical attacks (physically removing the
      disk from the robot to read its data).
  * Cloud Data
    * Different versions of the software are stored on the OTA server.
  * Secret Management
    * DDS / ROS Topics
      * If SROS is enabled, private keys are stored on the local filesystem of each module.
    * Mender OTA
      * Mender OTA client certificates are stored on the robot file system.


#### Use case scenarios

As described above, the application considered is a MARA modular robot operating on an industrial environment while performing a `pick & place` activity.For this use case all possible external actor have been included. The actions they can perform on the robotic system have been limited to the following ones:

* **End-Users**: From the End-user perspective, the functions considered are the ones needed for a factory line to work. Among this functions, there are the ones keeping the robot working and being productive.

  On an industrial environment, this group of external actors are the ones making use of the robot on their facilities.
  * **Robotics Researcher**: Development, Testing and Validation
    * A research engineer develops a pick and place task with the robot. They may:
      * Restart the robot
      * Restart the ROS graph
      * Physically interact with the robot
      * Receive software updates from OTA
      * Check for updates
      * Configure ORC control funtionalities

  * **Robotic User**: Collaborative tasks
     * Start the robot
     * Control the robot
     * Work alongside the robot

  * **Robotics Operator**: Automation of industrial tasks
    * An industrial operator uses the robot in a factory. They may:
      * Start the robot
      * Control the robot
      * Configure the robot
      * Include the robot into the industrial network
      * Check for updates
      * Configure and interacts with the ORC

* **Manufacturer**: From the manufacturer perspective, the application considered is the development of the MARA robotic platform, using an automated system for deployment of updates, following a CI/CD system.

  These are the actors who create and maintain the robotic system itself.

  * **Robotics Engineer**: Development, Testing and Validation
    * Development of new functionality and improvements for the MARA robot.
      * Develop new software for the H-ROS SoM
      * Update OS and system libraries
      * Update ROS2 subsystem and control nodes
      * Deployment of new updates to the robots and management of the fleet
      * In-Place robot maintenance

#### Entry points for MARA

The following section outlines the possible entry points an attacker could use as attack vectors to render the MARA vulnerable.

* **Physical Channels**
  * Exposed debug ports.
  * Internal field bus.
  * Hidden development test points.
* **Communication Channels**
  * DDS / ROS 2 Topics
    * Topics can be listened or written to by any actor:
      1. Connected to a network where DDS packets are routed to.
      2. Have necessary permissions (read / write) if SROS is enabled.
    * When SROS is enabled, attackers may try to compromise the CA authority
      or the private keys to generate or intercept private keys as well as
      emitting malicious certificates to allow spoofing.
  * H-ROS API
    * H-ROS API access is possible to anyone on the same LAN or WAN (if port forwarding is enabled).
    * When authentication (in the H-ROS API) is enabled, attackers may try to vulnerate it.
  * Mender OTA
    * Updates are pushed from the server.
    * Updates could be intercepted and modified before reaching the robot.
* **Deployed Software**
  * ROS nodes running on hardware are compiled by the manufacturer and deployed directly. An attacker may tamper the software running in the hardware by compromising the OTA services.
  * An attacker compromising the developer workstation could
    introduce a vulnerability in a binary which would then be deployed to the
    robot.
  * MoveIt! motion planning library may contain exploitable vulnerabilities.


#### Trust Boundaries for MARA in `pick & place` application

The following content will apply the threat model to an industrial robot on its environment. The objective on this threat analysis is to identify the attack vectors for the MARA robotic platform. Those attack vectors will be identified and clasified depending on the risk and services implied. On the next sections MARA's components and risks will be detailed thoroughly using the threat model above, based on STRIDE and DREAD.

The diagram below illustrates MARA's application with different trust zones
(trust boundaries showed with dashed green lines). The number and scope of trust
zones is depending on the infrastructure behind.

![Robot System Threat Model](ros2_threat_model/threat_model_mara.png)
[Diagram Source](ros2_threat_model/threat_model_mara.xml)
(edited with [Draw.io](draw_io))

The trust zones ilustrated above are the following:

* **Firmware Updates:** This zone is where the manufacturer develops the different firmware versions for each robot.
* **OTA System:** This zone is where the firmwares are stored for the robots to download.
* **MARA Robot:** All the robots' componentents and internal comunications are gathered in this zone.
* **Industrial PC (ORC):** The industrial PC itself is been considered a zone itself because it manages the software the end-user develops and sends it to the robot.
* **Software control:** This zone is where the end user develops software for the robot where the tasks to be performed are defined.

### Threat Model for MARA

Each generic threat described in the main threat table can be instantiated on the MARA.

This table indicates which of MARA's particular assets and entry points are
impacted by each threat. A check sign (✓) means impacted while a
cross sign (✘) means not impacted.
The "SROS Enabled?" column explicitly states out whether using SROS would
mitigate the threat or not. A check sign (✓) means that the threat could be
exploited while SROS is enabled while a cross sign (✘) means that the threat
requires SROS to be disabled to be applicable.

<div class="table" markdown="1">
<table class="table">
  <tr>
    <th rowspan="2" style="width: 20em">Threat</th>
    <th colspan="2" style="width: 6em">MARA Assets</th>
    <th colspan="6" style="width: 15em">Entry Points</th>
    <th rowspan="2" style="width: 3.25em; transform: rotate(-90deg) translateX(-2.5em) translateY(5.5em)">SROS Enabled?</th>
    <th rowspan="2" style="width: 30em">Attack</th>
    <th rowspan="2" style="width: 30em">Mitigation</th>
    <th rowspan="2" style="width: 30em">Mitigation Result (redesign / transfer
/ avoid / accept)</th>
    <th rowspan="2" style="width: 30em">Additional Notes / Open Questions</th>
  </tr>

  <tr style="height: 10em; white-space: nowrap;">
    <th style="transform: rotate(-90deg) translateX(-3.5em)
translateY(3em)">Human Assets</th>
    <th style="transform: rotate(-90deg) translateX(-3.5em)
translateY(3em)">Robot App.</th>
    <th style="transform: rotate(-90deg) translateX(-4em) translateY(4em)">ROS 2 API (DDS)</th>
    <th style="transform: rotate(-90deg) translateX(-4em) translateY(4em)">Manufacturer CI/CD</th>
    <th style="transform: rotate(-90deg) translateX(-4em) translateY(4em)">End-user CI/CD</th>
    <th style="transform: rotate(-90deg) translateX(-4em)translateY(4em)">H-ROS API</th>
    <th style="transform: rotate(-90deg) translateX(-4em)translateY(4em)">OTA</th>
    <th style="transform: rotate(-90deg) translateX(-4em)translateY(4em)">Physical</th>
  </tr>

  <tr><th colspan="13">Embedded / Software / Communication / Inter-Component
  Communication</th></tr>

  <tr>
    <td rowspan="3">An attacker spoofs a software component identity.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Without SROS any node may have any name so spoofing is trivial.</td>
    <td class="success">
      <ul>
        <li>Enable SROS / DDS Security Extension to authenticate and encrypt DDS communications.</li>
      </ul>
    </td>
    <td>Mitigating risk requires implementation of SROS on MARA.</td>
    <td>No verification of components. An attacker could connect a fake joint directly. Direct access to the system is granted. (No NAC)</td>
  </tr>

  <tr>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker deploys a malicious node which is not enabling DDS Security Extension and spoofs the joy_node forcing the robot to stop.</td>
    <td>
      <ul>
        <li>DDS Security Governance document must set allow_unauthenticated_participants to False to avoid non-authenticated participants to be allowed to communicate with authenticated nodes.</li>
        <li>DDS Security Governance document must set enable_join_access_control to True to explicitly whitelist node-to-node-communication. permissions.xml should be as restricted as possible."</li>
      </ul>
    </td>
    <td class="success">Risk is mitigated</td>
    <td></td>
  </tr>

  <tr>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td>An attacker steals node credentials and spoofs the joint node forcing the robot to stop.</td>
    <td>
      <ul>
        <li>Store node credentials in a secure location (secure enclave, RoT)
        to reduce the probability of having a private key leaked.</li>
        <li>Run nodes in isolated sandboxes to ensure one node cannot access
        another node data (including credentials)</li>
        <li>Permissions CA should digitally sign nodes binaries to prevent
        running tampered binaries.</li>
        <li>Permissions CA should be able to revoke certificates in case
        credentials get stolen.</li>
      </ul>
    </td>
    <td class="danger">Mitigation risk requires additional work.</td>
    <td>
      <ul>
        <li>AWS Robotics and Automation is currently evaluating the feasibility
        of storing DDS-Security credentials in a TPM.</li>
        <li>Complete mitigation would require isolation using e.g. Snap or
        Docker.</li>
        <li><span markdown="1">Deploying an application with proper isolation
        would require us to revive discussions around
        [ROS 2 launch system][ros2_launch_design_pr]</span></li>
        <li>Yocto / OpenEmbedded / Snap support should be considered</li>
      </ul>
    </td>
  </tr>

  <tr>
    <td>An attacker intercepts and alters a message.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Without SROS an attacker can modify `/goal_axis` or `trajectory_axis` messages sent through a network connection to e.g. stop the robot.
</td>
    <td>
    <ul>
        <li>Enable SROS / DDS Security Extension to authenticate and encrypt DDS communications. Message tampering is mitigated by DDS security as message authenticity is verified by default (with preshared HMACs / digital signatures)</li>
      </ul>
    </td>
    <td class="success">Risk is reduced if SROS is used.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker writes to a communication channel without
authorization.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Without SROS, any node can publish to any topic.</td>
    <td>
      <ul>
        <li>Enable SROS / DDS Security Extension to authenticate and encrypt DDS communications.</li>
      </ul>
    </td>
    <td class="success"></td>
    <td> </td>
  </tr>

  <tr>
    <td rowspan="2">An attacker listens to a communication channel without
authorization.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Without SROS: any node can listen to any topic.</td>
    <td>
    <ul>
        <li>Enable SROS / DDS Security Extension to authenticate and encrypt DDS communications.</li>
      </ul>
    </td>
    <td class="success">Risk is reduced if SROS is used.</td>
    <td></td>
  </tr>

<tr>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td><span markdown="1">DDS participants are enumerated and [fingerprinted][aztarna] to look for potential vulnerabilities.</span></td>
    <td>
      <ul>
        <li>DDS Security Governance document must set metadata_protection_kind to ENCRYPT to prevent malicious actors from observing communications.</li>
        <li>DDS Security Governance document mus set enable_discovery_protection to True to prevent malicious actors from enumerating and fingerprinting DDS participants.</li>
        <li>DDS Security Governance document must enable_liveliness_protection to True</li>
      </ul>
    </td>
    <td class="danger">Risk is mitigated if DDS-Security is configured appropriately.</td>
    <td>
    </td>
  </tr>

  <tr>
    <td rowspan="2">An attacker prevents a communication channel from being
usable.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Without SROS: any node can ""spam"" any other component.</td>
    <td>
      <ul>
        <li>Enable SROS to use the DDS Security Extension. This does not
prevent nodes from being flooded but it ensures that only communication from
allowed participants are processed.</li>
      </ul>
    </td>
    <td class="warning">Risk may be reduced when using SROS.</td>
    <td> </td>
  </tr>

  <tr>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>A node can ""spam"" another node it is allowed to communicate with.</td>
    <td>
      <ul>
        <li>Implement rate limitation on topics</li>
        <li>Define a method for topics to declare their required bandwidth /
rate.</li>
      </ul>
    </td>
    <td class="danger">Mitigating risk requires additional work.</td>
    <td>How to enforce when nodes are malicious? Observe and kill?</td>
  </tr>
  <tr><th colspan="13">Embedded / Software / Communication / Remote Application Interface</th></tr>
    <tr>
    <td>An attacker gains unauthenticated access to the remote application interface.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker connects to the H-ROS API in an unauthenticated way. Reads robot configuration and alters configuration values.</td>
    <td>
      <ul>
        <li>Add authentication mechanisms to the H-ROS API. </li>
        <li>Enable RBAC to limit user interaction with the API.</li>
      </ul>
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>
  <tr>
    <td>An attacker could eavesdrop communications to the Robot’s remote application interface.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker executes a MitM attack, eavesdropping all unencrypted communications and commands sent to the API.</td>
    <td>
        Encrypt the communications through the usage of HTTPS.
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>
  <tr>
    <td>An attacker could alter data sent to the Robot’s remote application interface.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker could execute a MitM attack and alter commands being sent to the API.</td>
    <td>
      Encrypt the communications through the usage of HTTPS.
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>
  <tr><th colspan="13">Embedded / Software / OS & Kernel</th></tr>

  <tr>
    <td>An attacker compromises the real-time clock to disrupt the kernel RT
scheduling guarantees.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>A malicious actor attempts to write a compromised kernel to /boot</td>
    <td>
      <ul>
        <li>Enable verified boot on Uboot to prevent booting altered kernels.</li>
        <li>Use built in TPM to store firmware public keys and define an RoT.</li>
      </ul>
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker compromises the OS or kernel to alter robot data.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>A malicious actor attempts to write a compromised kernel to /boot</td>
    <td>
      <ul>
        <li>Enable verified boot on Uboot to prevent booting altered kernels.</li>
        <li>Use built in TPM to store firmware public keys and define an RoT.</li>
      </ul>
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker compromises the OS or kernel to eavesdrop on robot
data.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>A malicious actor attempts to write a compromised kernel to /boot</td>
    <td>
      <ul>
        <li>Enable verified boot on Uboot to prevent booting altered kernels.</li>
        <li>Use built in TPM to store firmware public keys and define an RoT.</li>
      </ul>
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>
<tr><th colspan="13">Embedded / Software / Component-Oriented
Architecture</th></tr>

  <tr>
    <td rowspan="2">A node accidentally writes incorrect data to a communication
channel.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>A node writes random or invalid values to the /goal_axis topics.</td>
    <td>
      <ul>
        <li>Expand DDS IDL to allow users to embed validity criteria to automate input sanitization (i.e. validate ranges, etc.)</li>
        <li>Expand RMW to define per-topic strategies for invalid messages (drop, throw, abort, etc.).</li>
      </ul>
    </td>
    <td class="danger">Need to expand DDS IDL or RMW for mitigating the risk.	</td>
    <td></td>
  </tr>
  <tr>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>A node could to write out of physical bounds values to the `/goal_axis` or `/trajectory_axis` topics, causing damage to the robot.</td>
    <td><ul>
        <li>Define physical limitations of the different joints on the motor driver, limiting the possible movement to a safe range.</li><li>Enable signature verification of executables to reduce the risks of inserting a malicious node.</li><li>Limit control of actuators to only the required nodes. Enable AppArmor policies to isolate nodes.</li>
      </ul>
    </td>
    <td class="success">Risk is mitigated when applying limits on actuator drivers.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker deploys a malicious node on the robot.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>An attacker deploys a malicious node to the robot. This node performs dangerous movements that compromise safety. The node attempts to perform physical or logical damage to the modules.</td>
    <td>
      <ul>
        <li>Run each node in an isolated environment with limited privileges(sandboxing).</li>
        <li>Enable signing and verification of executables.</li>
      </ul>
    </td>
    <td class="warning"><ul>
        <li>Running the component in a Ubuntu Core sandbox environment could limit the consequences of the attack.</li>
        <li>Enabling signature verification of executables would reduce the risks of inserting a malicious node.</li>
        <li>Limiting control of actuators to only the required nodes would reduce risk in case of a node compromise. Enable AppArmor policies to isolate nodes.</li>
      </ul></td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker can prevent a component running on the robot from executing
normally.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>A malicious node running on the robot starts sending kill requests to other nodes in the system, disrupting the normal behaviour of the robot.
</td>
    <td>Having the abiliy to shutdown/kill nodes through API request supposes a problem on the ROS implementation. Deprecation of the function should be considered. Node restarting policie should be applied.
    </td>
    <td class="danger">Deprecation of the shutdown API call needs to be considered.</td>

  </tr>
  <tr><th colspan="13">Embedded / Software / Configuration Management</th></tr>

  <tr>
    <td>An attacker modifies configuration values without authorization.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Node parameters are freely modifiable by any DDS domain participant.</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker accesses configuration values without authorization.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td>Node parameters values can be read by any DDS domain participant.</td>
    <td></td>
    <td></td>
    <td></td>
    <td> </td>
  </tr>

  <tr>
    <td>A user accidentally misconfigures the robot.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>Node parameters values can be modified by anyone to any value.</td>
    <td></td>
    <td></td>
    <td></td>
    <td> </td>
  </tr>

  <tr><th colspan="13">Embedded / Software / Data Storage (File
System)</th></tr>

  <tr>
    <td>An attacker modifies the robot file system by physically acessing
it.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>An attacker modifies the filesystem data within the robot</td>
    <td>Enable filesystem encryption with LUKS or dm-crypt, storing keys on TPM device.</td>
    <td class="success">Risk is mitigated.</td>
    <td> </td>
  </tr>

  <tr>
    <td>An attacker eavesdrops on the robot file system by physically accessing
it.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>An attacker physically accesses the memory chip to eavesdrop credentials, logs or sensitive data.</td>
    <td>Enable filesystem encryption with LUKS or dm-crypt, storing keys on TPM device.</td>
    <td class="success">Risk is mitigated.</td>
    <td> </td>
  </tr>

  <tr>
    <td>An attacker saturates the robot disk with data.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td>A malicious node writes random data that fills the robot disk.
</td>
    <td>Enable disk quotas on the system. Enable sandboxing of the processes. Separate disk into multiple partitions, sending non trivial data to temporary directories.</td>
    <td class="warning">Risk is partially mitigated. Disk cleanup routines and log rotation should also be implemented.</td>
    <td></td>
  </tr>

  <tr><th colspan="13">Embedded / Software / Logs</th></tr>

  <tr>
    <td>An attacker exfiltrates log data to a remote server.</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker compromising the OTA server could request device log data and eavesdrop sensitive information.</td>
    <td>Enable RBAC on the OTA server, limit access to sensitive functions.
    </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr><th colspan="13">Embedded / Hardware / Sensors</th></tr>

  <tr>
    <td>An attacker spoofs a robot sensor (by e.g. replacing the sensor itself
or manipulating the bus).</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="warning">✘/✓</td>
    <td>An attacker could physically tamper the readings from the sensors.
</td>
    <td>Add noise or out-of-bounds reading detection mechanism on the robot, causing to discard the readings or raise an alert to the user. Add detection of sensor disconnections.
</td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr><th colspan="13">Embedded / Hardware / Actuators</th></tr>

  <tr>
    <td>An attacker spoofs a robot actuator.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker could insert a counterfeit modular joint in the robot, compromising the whole system (e.g. a modified gripper).</td>
    <td><ul><li>Implement network access control systems, performing a verification of the part before granting access to the system.</li><li>Implement certificate based, 802.1x authentication for the communication with the nodes, discarding any new modules that do not authenticate on the system.</li></ul>
  </td>
    <td class="success">Risk is mitigated.</td>
    <td>Additional evaluation should be performed. Authenticating nodes via certificates would require shipping the nodes with client certificates, and the validated manufacturers would require a subordinate CA to sign their modules.(Kinda DRM-ish)</td>
  </tr>

  <tr>
    <td>An attacker modifies the command sent to the robot actuators.
(intercept & retransmit)</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker intercepts the communication channel traffic. The command is altered an retransmitted to the target joint.</td>
    <td><ul><li>Implement network access control systems, performing a verification of the part before granting access to the system.</li><li>Implement certificate based, 802.1x authentication for the communication with the nodes, discarding any new modules that do not authenticate on the system.</li></ul></td>
    <td class="success">Risk is mitigated.</td>
    <td> </td>
  </tr>
<tr><th colspan="13">Embedded / Hardware / Communications</th></tr>

  <tr>
    <td>An attacker connects to an exposed debug port.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker could connect to an exposed debug port and gain control over the robot through the execution of arbitrary commands.</td>
    <td>
      <ul>
      <li>Limit access or remove exposed debug ports.</li>
      <li>Disable local debug terminals and functionality from the ports.</li>
      <li>Add authentication mechanisms to limit access to the ports only to authenticated devices and users.</li>
    </ul>
  </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker connects to an internal communication bus.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker could connect to an internal communication bus to send arbitrary data or eavesdrop communication between different components of the robot.</td>
    <td>
      <ul>
      <li>Limit access or remove unused communication ports.</li>
      <li>Physically limit access to the internal robot components and communication buses.</li>
      <li>Add physical tamper detection sensors to detect physical intrussions to the robot.</li>
    </ul>
  </td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>



  <tr><th colspan="13">Remote / Software Deployment</th></tr>

  <tr>
    <td>An attacker spoofs the deployment service.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker spoofs the update deployment server and serves malicious content to the devices.</td>
    <td><ul><li>Validate the deployment server through Public Key Infrastructure.</li> <li>Prevent insecure connections to the server from the devices through HTTPS and HSTS policies.</li><li>Certificate pinning on devices.</li></ul>
</td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker modifies the binaries sent by the deployment service.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker intercepts the communication to the deployment server and serves malicious content to the devices.
</td>
    <td><ul><li>Validate the deployment server through Public Key Infrastructure.</li> <li>Prevent insecure connections to the server from the devices through HTTPS and HSTS policies.</li>
<li>Digitally sign the binaries sent to the devices.</li></ul>
</td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker intercepts the binaries sent by the depoyment service.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker intercepts the update communication and stores the binary sent to the devices, gaining access to intellectual property.
</td>
    <td><ul><li>Make use of secure, encrypted communication channels.</li> <li>Verify client devices through client certificates.</li>
<li>Sign and Encrypt update files.</li></ul>
</td>
    <td class="success">Risk is mitigated.</td>
    <td></td>
  </tr>

  <tr>
    <td>An attacker prevents the robot  and the deployment service from
communicating.</td>
    <td class="danger">✓</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="success">✘</td>
    <td class="danger">✓</td>
    <td class="success">✘</td>
    <td class="warning">✘/✓</td>
    <td>An attacker blocks the robots update process.</td>
    <td>
      <ul>
        <li>Deploy multiple update server endpoints.</li>
        <li>Deploy a distributed update system.</li>
      </ul>
    </td>
    <td class="warning">Risk is partially mitigated.</td>
    <td></td>
  </tr>

</table>
</div>

#### MARA Attack Trees

In an atempt to analyze the different possible attacks before even happening, attack trees are created. This diagrams speculate about the possible attacks against the system in order to be able to counter them. Threats can be organized in the following [attack trees][wikipedia_attack_tree]. Attacks are ordered starting from the physical attack vector, continuing with network based attacks, and finishing with the support infrastructure for the robot.



#### Physical vector attack tree

The following attack tree describes the possible paths to be followed by an attacker for compromising the system.


![Physical vector attack tree](ros2_threat_model/attack_tree_physical.png)


[Diagram Source (draw.io)](ros2_threat_model/attack_tree_physical.xml)

The next diagram shows the infrastructure affected on a possible attack based on a compromise of a physical communication port.


![Physical vector attack architecture](ros2_threat_model/attack_tree_physical_arch.png)






#### ROS2 API vector attack tree


The following attack tree describes the possible paths to be followed by an attacker for  physically compromising the system.



![ROS2 API vector attack tree](ros2_threat_model/attack_tree_ros2.png)

[Diagram Source (draw.io)](ros2_threat_model/attack_tree_ros2.xml)

The following diagram shows the infrastructure affected on a possible attack based on exploitation of the ROS2 API.

![ROS2 API vector attack architecture](ros2_threat_model/attack_tree_ros2_arch.png)






#### H-ROS API vector attack tree

The following attack tree describes the possible paths to be followed by an attacker for compromising the system.


![H-ROS API vector attack tree](ros2_threat_model/attack_tree_api.png)

[Diagram Source (draw.io)](ros2_threat_model/attack_tree_api.xml)

The diagram below shows the infrastructure affected on a possible attack against MARA's H-ROS API.

![H-ROS API vector attack architecture](ros2_threat_model/attack_tree_api_arch.png)





#### Code repository compromise vector attack tree

The following attack tree describes the possible paths to be followed by an attacker for compromising the system.



![Code repository compromise attack tree](ros2_threat_model/attack_tree_repo.png)

[Diagram Source (draw.io)](ros2_threat_model/attack_tree_repo.xml)

The following diagram shows the infrastructure affected on a possible attack against the manufacturer code repository, showing its potential implications and consequences.


![Code repository attack architecture](ros2_threat_model/attack_tree_repo_arch.png)






### Threat Model Validation Strategy for MARA

<div class="alert alert-info" markdown="1">
Validating this threat model end-to-end is a long-term effort meant to be
distributed among the whole ROS 2 community. The threats list is made to
evolve with this document and additional reference may be introduced in
the future.
</div>

1. Setup a MARA with the exact software described in the Components MARA section
   of this document.
2. Penetration Testing
     * Attacks described in the spreadsheet should be implemented. For instance,
       a malicious <code>hros_actuator_servomotor_XXXXXXXXXXXX</code> could be implemented to try to
       disrupt the robot operations.
     * Once the vulnerability has been exploited, the exploit should be released
       to the community so that the results can be reproduced.
     * Whether the attack has been successful or not, this document should be
       updated accordingly.
     * If the attack was successful, a mitigation strategy should be implemented.
       It can either be done through improving ROS 2 core packages or it can be
       a platform-specific solution. In the second case, the mitigation will serve
       as an example to publish best practices for the development of secure
       robotic applications.
      * For trainning purposes, an online playground ([RCTF][rctf]) exists to challenge roboticists to learn and discover robot vulnerabilities.
      * For an overall evaluation of the robots' security measures, the Robot Security Framework ([RSF][rsf]) will be used. This validation has to be don e after the assessment is completed in order to have a realistic results.




### MARA Security Assessment preliminary results

Due to the iterative nature of this process, the results shown on this section may differ depending on the threat model's version.

The results here presented are a summary of the discoveries made during the Acutronic Robotics' MARA assessment. Prior to the assessment, in order to be time-effective, a threat analysis was done over the system and the most likely attack vectors were identified.

#### Introduction

Based on the threat model performed over MARA, an assessment has been performed in order to discover the vulnerabilities within. The content below is directly related to that threat model using all the attack vectors identified as starting points for the exercise.

This Security Assessment Report for Acutronic Robotics’ MARA components has been performed by Alias Robotics S.L. The targeted system for the assessment is the H-ROS powered MARA. H-ROS, the Hardware Robot Operating System, is a collection of hardware and software specifications and implementations that allows creating modular and distributed robot parts. H-ROS heavily relies on the use of ROS 2 as a robotic development framework.

This document presents an overview of the results obtained in the assessment. In it, we introduce a report focused on identifying security issues or vulnerabilities detected during the exercise.

#### Results

This assessment has evaluated the security measures present in the H-ROS SoM components. The evaluation has been done through a vulnerability and risk assessment, and the following sections are a summary of the findings.

The vulnerabilities are rated on a severity scale according to the guidelines stated in the Robot Vulnerability Scoring System ([RVSS][rvss]), resulting in a score between 0 and 10, according to the severity of the vulnerability. The RVSS follows the same principles as the Common Vulnerability Scoring System (CVSS), yet adding aspects related to robotics systems that are required to capture the complexity of robot vulnerabilities.  Every evaluated aspect is defined in a vector, having its weight at the final score.

All assessment findings from the MARA are classified by technical severity. The following list explains how Alias Robotics rates vulnerabilities by their severity.

* **Critical**: Scores between 9 and 10
* **High**: Scores between 7 and 8.9
* **Medium**: Scores between 4 and 6.9
* **Low**: Scores between 0.1 and 3.9
* **Informational**: Scores 0.0

![Signing Service Mitigation](ros2_threat_model/vulns.png)

In the course of a first funded vulnerability assessment, <a style="color:black;">**2 critical**</a>, <a style="color:red;">**6 high**</a> , <a style="color:orange;">**13 medium**</a> and <a style="color:green;">**6 low**</a> vulnerabilities have been found. The ones fixed by Acutronic Robotics are disclosed below using the following format:

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>Name of the vulnerability</td>
  </tr>
    <tr>
    <th>ID</th>
    <td>Vulnerability identifier</td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>Rating assigned according to the Robot Vulnerability Scoring System (RVSS), based on the severity of the vulnerability</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>Vector showing the different aspects that have defined the final score</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>Information about the nature of the vulnerability</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td>Status of the remediation, with evidences and date (Active, Fixed)</td>
  </tr>
</table>
</div>

<br/>
<br/>

##### Findings

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>H-ROS API vulnerable to DoS attacks</td>
  </tr>
    <tr>
    <th>ID</th>
    <td><a href="https://github.com/aliasrobotics/RVDP/issues/97">Vuln-07</a></td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>7.5</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>RVSS:1.0/AV:RN/AC:L/PR:N/UI:N/Y:Z/S:U/C:N/I:N/A:H/H:N</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>The H-ROS API does not use any mechanism for limiting the requests that a user is able to perform in a determined set of time. This can lead to DoS attacks or premature wearing of the device.</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td class="success">✓</td>
  </tr>
</table>
</div>

<br/>
<br/>

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>Insufficient limitation of hardware boundaries.</td>
  </tr>
    <tr>
    <th>ID</th>
    <td><a href="https://github.com/aliasrobotics/RVDP/issues/98">Vuln-02</a></td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>6.8</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>RVSS:1.0/AV:IN/AC:L/PR:L/UI:N/Y:Z/S:U/C:N/I:L/A:H/H:E</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>Actuator drivers do not correctly limit the movement boundaries, being able to execute commands above the physical capabilities of the actuator.</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td class="success">✓</td>
  </tr>
</table>
</div>

<br/>
<br/>

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>ROS 2 Goal topic vulnerable to DoS attacks.</td>
  </tr>
    <tr>
    <th>ID</th>
    <td><a href="https://github.com/aliasrobotics/RVDP/issues/99">Vuln-13</a></td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>5.5</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>RVSS:1.0/AV:IN/AC:L/PR:N/UI:N/Y:Z/S:U/C:N/I:N/A:H/H:U</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>The ROS 2 nodes that control the motor fail when a big number of messages are sent in a small span of time. The application crashes and is not able to recover from failure, causing a DoS. This is probably caused by memory leaks, bugs or log storage.</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td class="success">✓</td>
  </tr>
</table>
</div>

<br/>
<br/>

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>ROS 2 Trajectory topic vulnerable to DoS</td>
  </tr>
    <tr>
    <th>ID</th>
    <td><a href="https://github.com/aliasrobotics/RVDP/issues/100">Vuln-14</a></td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>5.5</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>RVSS:1.0/AV:IN/AC:L/PR:N/UI:N/Y:Z/S:U/C:N/I:N/A:H/H:N</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>Trajectory messages sent to the `/trajectory` topic of the H-ROS SoM cause an unrecoverable failure on the ROS node that can only be fixed by a system restart.</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td class="success">✓</td>
  </tr>
</table>
</div>

<br/>
<br/>

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>OTA OpenSSH Linux distribution version disclosure</td>
  </tr>
    <tr>
    <th>ID</th>
    <td><a href="https://github.com/aliasrobotics/RVDP/issues/101">Vuln-20</a></td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>5.3</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>RVSS:1.0/AV:RN/AC:L/PR:N/UI:N/Y:Z/S:U/C:L/I:N/A:N/H:N</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>The OpenSSH server discloses the distribution name (Ubuntu) being used in the server in the connection headers, providing additional information to attackers.</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td class="success">✓</td>
  </tr>
</table>
</div>

<br/>
<br/>

<div class="table">
<table class="table">
  <tr>
    <th>Name</th>
    <td>OTA OpenSSH version vulnerable to user enumeration attacks</td>
  </tr>
  <tr>
    <th>ID</th>
    <td><a href="https://github.com/aliasrobotics/RVDP/issues/102">Vuln-21</a></td>
  </tr>
  <tr>
    <th>RVSS Score</th>
    <td>5.3</td>
  </tr>
  <tr>
    <th>Scoring Vector</th>
    <td>RVSS:1.0/AV:RN/AC:L/PR:N/UI:N/Y:Z/S:U/C:L/I:N/A:N/H:N</td>
  </tr>
    <tr>
    <th>Description</th>
    <td>TThe OpenSSH server version 7.6p1 is vulnerable to user enumeration attacks by timing.</td>
  </tr>
    <tr>
    <th>Remediation Status</th>
    <td class="success">✓</td>
  </tr>
</table>
</div>

## References

1. Abera, Tigist, N. Asokan, Lucas Davi, Jan-Erik Ekberg, Thomas Nyman, Andrew
   Paverd, Ahmad-Reza Sadeghi, and Gene Tsudik. “C-FLAT: Control-FLow
   ATtestation for Embedded Systems Software.”
   ArXiv:1605.07763 [Cs], May 25, 2016. <http://arxiv.org/abs/1605.07763>.
2. Ahmad Yousef, Khalil, Anas AlMajali, Salah Ghalyon, Waleed Dweik, and Bassam
   Mohd. “Analyzing Cyber-Physical Threats on Robotic Platforms.” Sensors 18,
   no. 5 (May 21, 2018): 1643. <https://doi.org/10.3390/s18051643>.
3. Bonaci, Tamara, Jeffrey Herron, Tariq Yusuf, Junjie Yan, Tadayoshi Kohno, and
   Howard Jay Chizeck. “To Make a Robot Secure: An Experimental Analysis of
   Cyber Security Threats Against Teleoperated Surgical Robots.”
   ArXiv:1504.04339 [Cs], April 16, 2015. <http://arxiv.org/abs/1504.04339>.
4. Checkoway, Stephen, Damon McCoy, Brian Kantor, Danny Anderson, Hovav Shacham,
   Stefan Savage, Karl Koscher, Alexei Czeskis, Franziska Roesner, and Tadayoshi
   Kohno. “Comprehensive Experimental Analyses of Automotive Attack Surfaces.”
   In Proceedings of the 20th USENIX Conference on Security, 6–6. SEC’11.
   Berkeley, CA, USA: USENIX Association, 2011.
   <http://dl.acm.org/citation.cfm?id=2028067.2028073>.
5. Clark, George W., Michael V. Doran, and Todd R. Andel. “Cybersecurity Issues
   in Robotics.” In 2017 IEEE Conference on Cognitive and Computational Aspects
   of Situation Management (CogSIMA), 1–5. Savannah, GA, USA: IEEE, 2017.
   <https://doi.org/10.1109/COGSIMA.2017.7929597>.
6. Denning, Tamara, Cynthia Matuszek, Karl Koscher, Joshua R. Smith, and
   Tadayoshi Kohno. “A Spotlight on Security and Privacy Risks with Future
   Household Robots: Attacks and Lessons.” In Proceedings of the 11th
   International Conference on Ubiquitous Computing - Ubicomp ’09, 105. Orlando,
   Florida, USA: ACM Press, 2009. <https://doi.org/10.1145/1620545.1620564>.
7. Dessiatnikoff, Anthony, Yves Deswarte, Eric Alata, and Vincent Nicomette.
   “Potential Attacks on Onboard Aerospace Systems.” IEEE Security & Privacy 10,
   no. 4 (July 2012): 71–74. <https://doi.org/10.1109/MSP.2012.104>.
8. Dzung, D., M. Naedele, T.P. Von Hoff, and M. Crevatin. “Security for
   Industrial Communication Systems.” Proceedings of the IEEE 93, no. 6 (June
   2005): 1152–77. <https://doi.org/10.1109/JPROC.2005.849714>.
9. Elmiligi, Haytham, Fayez Gebali, and M. Watheq El-Kharashi.
   “Multi-Dimensional Analysis of Embedded Systems Security.” Microprocessors
   and Microsystems 41 (March 2016): 29–36. <https://doi.org/10.1016/j.micpro.2015.12.005>.
10. Groza, Bogdan, and Toma-Leonida Dragomir. “Using a Cryptographic
   Authentication Protocol for the Secure Control of a Robot over TCP/IP.” In
   2008 IEEE International Conference on Automation, Quality and Testing,
   Robotics, 184–89. Cluj-Napoca, Romania: IEEE, 2008.<https://doi.org/10.1109/AQTR.2008.4588731>.
11. Javaid, Ahmad Y., Weiqing Sun, Vijay K. Devabhaktuni, and Mansoor Alam.
   “Cyber Security Threat Analysis and Modeling of an Unmanned Aerial Vehicle
   System.” In 2012 IEEE Conference on Technologies for Homeland Security (HST),
   585–90. Waltham, MA, USA: IEEE, 2012.<https://doi.org/10.1109/THS.2012.6459914>.
12. Kleidermacher, David, and Mike Kleidermacher. Embedded Systems Security:
   Practical Methods for Safe and Secure Software and Systems Development.
   Amsterdam: Elsevier/Newnes, 2012.
13. Klein, Gerwin, June Andronick, Matthew Fernandez, Ihor Kuz, Toby Murray, and
   Gernot Heiser. “Formally Verified Software in the Real World.” Communications
   of the ACM 61, no. 10 (September 26, 2018): 68–77.<https://doi.org/10.1145/3230627>.
14. Lee, Gregory S., and Bhavani Thuraisingham. “Cyberphysical Systems Security
   Applied to Telesurgical Robotics.” Computer Standards & Interfaces 34, no. 1
   (January 2012): 225–29. <https://doi.org/10.1016/j.csi.2011.09.001>.
15. Lera, Francisco J. Rodríguez, Camino Fernández Llamas, Ángel Manuel Guerrero,
   and Vicente Matellán Olivera. “Cybersecurity of Robotics and Autonomous
   Systems: Privacy and Safety.” In Robotics - Legal, Ethical and Socioeconomic
   Impacts, edited by George Dekoulis. InTech, 2017.<https://doi.org/10.5772/intechopen.69796>.
16. McClean, Jarrod, Christopher Stull, Charles Farrar, and David Mascareñas. “A
   Preliminary Cyber-Physical Security Assessment of the Robot Operating System
   (ROS).” edited by Robert E. Karlsen, Douglas W. Gage, Charles M. Shoemaker,
   and Grant R. Gerhart, 874110. Baltimore, Maryland, USA, 2013.<https://doi.org/10.1117/12.2016189>.
17. Morante, Santiago, Juan G. Victores, and Carlos Balaguer. “Cryptobotics: Why
   Robots Need Cyber Safety.” Frontiers in Robotics and AI 2 (September 29,
   2015). <https://doi.org/10.3389/frobt.2015.00023>.
18. Papp, Dorottya, Zhendong Ma, and Levente Buttyan. “Embedded Systems Security:
   Threats, Vulnerabilities, and Attack Taxonomy.” In 2015 13th Annual
   Conference on Privacy, Security and Trust (PST), 145–52. Izmir,
   Turkey: IEEE, 2015. <https://doi.org/10.1109/PST.2015.7232966>.
19. Pike, Lee, Pat Hickey, Trevor Elliott, Eric Mertens, and Aaron Tomb.
   “TrackOS: A Security-Aware Real-Time Operating System.” In Runtime
   Verification, edited by Yliès Falcone and César Sánchez, 10012:302–17. Cham:
   Springer International Publishing, 2016.<https://doi.org/10.1007/978-3-319-46982-9_19>.
20. Ravi, Srivaths, Paul Kocher, Ruby Lee, Gary McGraw, and Anand Raghunathan.
   “Security as a New Dimension in Embedded System Design.” In Proceedings of
   the 41st Annual Conference on Design Automation  - DAC ’04, 753. San Diego,
   CA, USA: ACM Press, 2004. <https://doi.org/10.1145/996566.996771>.
21. Serpanos, Dimitrios N., and Artemios G. Voyiatzis. “Security Challenges in
   Embedded Systems.” ACM Transactions on Embedded Computing Systems 12, no. 1s
   (March 29, 2013): 1–10. <https://doi.org/10.1145/2435227.2435262>.
22. Vilches, Víctor Mayoral, Laura Alzola Kirschgens, Asier Bilbao Calvo,
   Alejandro Hernández Cordero, Rodrigo Izquierdo Pisón, David Mayoral Vilches,
   Aday Muñiz Rosas, et al. “Introducing the Robot Security Framework (RSF), a
   Standardized Methodology to Perform Security Assessments in Robotics.”
   ArXiv:1806.04042 [Cs], June 11, 2018. <http://arxiv.org/abs/1806.04042>.
23. Zubairi, Junaid Ahmed, and Athar Mahboob, eds. Cyber Security Standards,
   Practices and Industrial Applications: Systems and Methodologies.
   IGI Global, 2012. <https://doi.org/10.4018/978-1-60960-851-4>.
24. Akhtar, Naveed, and Ajmal Mian. “Threat of Adversarial Attacks on Deep
   Learning in Computer Vision: A Survey.” ArXiv:1801.00553 [Cs],
   January 2, 2018. <http://arxiv.org/abs/1801.00553>.
25. V. Mayoral Vilches, E. Gil-Uriarte, I. Zamalloa Ugarte, G. Olalde Mendia, R. Izquierdo Pisón, L. Alzola Kirschgens, A. Bilbao Calvo, A. Hernández     Cordero, L. Apa, and C. Cerrudo, “Towards an open standard for assessing the severity of robot security vulnerabilities, the Robot Vulnerability Scoring System (RVSS),” ArXiv:807.10357 [Cs], Jul. 2018. <https://arxiv.org/abs/1807.10357>
26. G. Olalde Mendia, L. Usategui San Juan, X. Perez Bascaran, A. Bilbao Calvo, A. Hernández Cordero, I. Zamalloa Ugarte, A. Muñiz Rosas, D. Mayoral Vilches, U. Ayucar Carbajo, L. Alzola Kirschgens, V. Mayoral Vilches, and E. Gil-Uriarte, “Robotics CTF (RCTF), a playground for robot hacking,” ArXiv:810.02690 [Cs], Oct. 2018. <https://arxiv.org/abs/1810.02690>
27. L. Alzola Kirschgens, I. Zamalloa Ugarte, E. Gil Uriarte, A. Muñiz Rosas, and V. Mayoral Vilches, “Robot hazards: from safety to security,” ArXiv:1806.06681 [Cs], Jun. 2018. <https://arxiv.org/abs/1806.06681>
28. Víctor Mayoral Vilches, Gorka Olalde Mendia, Xabier Perez Baskaran, Alejandro Hernández Cordero, Lander Usategui San Juan, Endika Gil-Uriarte, Odei Olalde Saez de Urabain, Laura Alzola Kirschgens, "Aztarna, a footprinting tool for robots," ArXiv:1806.06681 [Cs], December 22, 2018. <https://arxiv.org/abs/1812.09490>

[adbd_linux]: https://github.com/tonyho/adbd-linux
[akhtar_threat_2018]: http://arxiv.org/abs/1801.00553
[aws_code_signing]: https://docs.aws.amazon.com/signer/latest/developerguide/Welcome.html
[cw_sample_app]: https://github.com/aws-robotics/aws-robomaker-sample-application-cloudwatch
[fastrtps_security]: https://eprosima-fast-rtps.readthedocs.io/en/latest/security.html
[joy_node]: https://github.com/ros2/joystick_drivers/blob/ros2/joy/src/joy_node_linux.cpp
[opencr_1_0]: https://github.com/ROBOTIS-GIT/OpenCR
[raspicam2_node]: https://github.com/christianrauch/raspicam2_node
[rmw_fastrtps]: https://github.com/ros2/rmw_fastrtps
[ros2_launch_design_pr]: https://github.com/ros2/design/pull/163
[ros_wiki_tb]: https://wiki.ros.org/Robots/TurtleBot
[tb3_burger]: http://emanual.robotis.com/docs/en/platform/turtlebot3/specifications/#hardware-specifications
[tb3_ros2_setup]:(http://emanual.robotis.com/docs/en/platform/turtlebot3/ros2/#setup
[tb3_teleop]: https://github.com/ROBOTIS-GIT/turtlebot3/tree/master/turtlebot3_teleop
[teleop_twist_joy]: https://github.com/ros2/teleop_twist_joy
[threat_dragon]: https://threatdragon.org/
[turtlebot3_node]: https://github.com/ROBOTIS-GIT/turtlebot3/tree/ros2/turtlebot3_node
[wikipedia_attack_tree]: https://en.wikipedia.org/wiki/Attack_tree
[wikipedia_bastion_hosts]: https://en.wikipedia.org/wiki/Bastion_host
[wikipedia_code_signing]: https://en.wikipedia.org/wiki/Code_signing
[wikipedia_dread]: https://en.wikipedia.org/wiki/DREAD_(risk_assessment_model)
[wikipedia_federated_identity]: https://en.wikipedia.org/wiki/Federated_identity
[wikipedia_public_key]: https://en.wikipedia.org/wiki/Public-key_cryptography
[wikipedia_stride]: https://en.wikipedia.org/wiki/STRIDE_(security)
[mara_robot]: https://acutronicrobotics.com/products/mara/
[mara_datasheet]: https://acutronicrobotics.com/products/mara/files/Robotic_arm_MARA_datasheet_v1.2.pdf
[hrim]: https://acutronicrobotics.com/technology/hrim/
[hros]: https://acutronicrobotics.com/technology
[hrossom]: https://acutronicrobotics.com/technology/som/
[mara_joint_ros2_api]: https://acutronicrobotics.com/docs/products/actuators/modular_motors/hans/ros2_api
[orc]: https://acutronicrobotics.com/products/orc/
[hans_modular_joint]: https://acutronicrobotics.com/products/modular-joints/
[hros_connector_A]: https://acutronicrobotics.com/products/modular-joints/files/HROS_Connector_A_%20Robot_Assembly_datasheet_v1.0.pdf
[robotiq_modular_gripper]: https://acutronicrobotics.com/products/modular-grippers/
[rvss]: https://arxiv.org/abs/1807.10357
[rctf]: https://arxiv.org/abs/1810.02690
[aztarna]: https://arxiv.org/abs/1812.09490
[rsf]: http://arxiv.org/abs/1806.04042
[safe_sec]: https://arxiv.org/abs/1806.06681
