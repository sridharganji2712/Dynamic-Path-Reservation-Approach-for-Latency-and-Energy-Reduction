# Dynamic-Path-Reservation-Approach-for-Latency-and-Energy-Reduction

Path Reservation / OCS 

Core Idea of Path Reservation:
Path Reservation (PR), also referred to as OCS bypass in this codebase, is an enhancement on top of standard wormhole routing. In classic wormhole mode, every flit must go through buffer write, VC allocator, and switch allocator at each hop. This creates unnecessary overhead when a flow is stable and repeatedly selects the same output direction.PR introduces an adaptive, local reservation mechanism. Each router watches the output direction chosen for a given flow (VC/port). If the flow consistently travels in the same direction, and buffer/credit conditions are safe, the router activates a temporary fast-path. On this fast-path, the flow bypasses arbitration and proceeds directly to the output port each cycle, behaving like a lightweight, short-lived circuit.

The reservation automatically dissolves when the flow becomes unstable, when it idles, or when a competitor appears. No global control packets are used; everything is local and topology-agnostic. The effect is reduced latency, reduced arbitration work, and higher throughput for flows that exhibit directional continuity.








----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

List of Modified Files :



=============================================================================
Router.h and Router.cpp

These files contain the full Path Reservation implementation.

Key fields and parameters used here:

GlobalParams::OCS_ENABLE
GlobalParams::OCS_DEBUG
GlobalParams::OCS_PRED_MIN
GlobalParams::OCS_MIN_BURST
GlobalParams::OCS_CREDIT_REQ
GlobalParams::OCS_IDLE_RELEASE

These parameters control when a reservation is created, sustained, or released.

Important router-level structures and variables:

reservation_active
Tracks whether a specific (input VC → output port) mapping is currently reserved.

reservation_idle_counter
Counts idle cycles for the reserved path and is used to release the reservation when it exceeds OCS_IDLE_RELEASE.

direction_history
Tracks recent output-port decisions for each flow to compute prediction stability.

prediction_score
Quantifies directional continuity and is compared against OCS_PRED_MIN.

Important functions and why they are used:

getNextHops(RouteData rd)
Retrieves possible next-hop output ports. PR uses these outputs to determine if the predicted direction is stable.

nextDeltaHops(RouteData rd)
Computes deterministic next-hop directions. PR uses this function when comparing the chosen direction with the direction history.

evaluateOCSActivation(int in_port, int vc)
Checks if prediction_score ≥ OCS_PRED_MIN, burst length ≥ OCS_MIN_BURST, and downstream credits ≥ OCS_CREDIT_REQ. If all conditions hold, sets reservation_active.

bypassSwitchAllocator(int in_port, int vc)
Executes fast-path traversal when reservation_active is true. This function bypasses arbitration entirely.

releaseOCSReservation(int in_port, int vc, int reason)
Releases the reservation when idle cycles exceed OCS_IDLE_RELEASE, direction changes, or credit constraints fail.

These functions form the complete activation, maintenance, and bypass logic for PR.




=================================================================================================================
GlobalParams.h and GlobalParams.cpp

These files define and initialize the parameters required by PR.

Parameter names:

OCS_ENABLE
Controls whether PR logic is active.

OCS_DEBUG
Enables or disables reservation activity logging.

OCS_PRED_MIN
Minimum prediction score needed to activate a reservation.

OCS_MIN_BURST
Minimum run-length (number of consecutive flits) needed before activation.

OCS_CREDIT_REQ
Minimum downstream credits required for safe bypass activation.

OCS_IDLE_RELEASE
Maximum allowed idle cycles before a reservation is released.

These fields are referenced directly inside Router.cpp to evaluate reservation conditions and enforce release logic.



========================================================================================================================
ConfigurationManager.cpp

This file maps YAML configuration parameters into GlobalParams variables.

Function and parameter names:

readParam<bool>(config, "ocs_enable") → GlobalParams::OCS_ENABLE
readParam<bool>(config, "ocs_debug") → GlobalParams::OCS_DEBUG
readParam<double>(config, "ocs_pred_min") → GlobalParams::OCS_PRED_MIN
readParam<int>(config, "ocs_min_burst") → GlobalParams::OCS_MIN_BURST
readParam<int>(config, "ocs_credit_req") → GlobalParams::OCS_CREDIT_REQ
readParam<int>(config, "ocs_idle_release") → GlobalParams::OCS_IDLE_RELEASE

These functions ensure that simulation parameters in ocs.yaml are consistently loaded into the global PR configuration.


========================================================================================================================
ReservationTable.h and ReservationTable.cpp

These files implement the reservation mechanisms for router ports and virtual channels. PR uses these tables as the underlying structure for associating an input VC with an output port during a reservation.

Important functions:

isAvailable(int output_port)
Checks whether a port is free for reservation.

reserve(int input_port, int vc, int output_port)
Creates the input-to-output mapping used for fast-path traversal.

release(int input_port, int vc)
Removes the mapping when PR logic determines that the reservation should end.

These functions allow PR to enforce exclusive fast-path control during bypass mode.


========================================================================================================================
LocalRoutingTable.h and LocalRoutingTable.cpp

These files calculate next-hop directions for each router and are referenced by the router during PR decision-making.

Key function:

getOutputPort(int src_id, int dst_id)
Determines the next-hop direction based on topology and routing strategy. PR uses this output to evaluate directional continuity and update prediction_score.


========================================================================================================================
GlobalRoutingTable.h and GlobalRoutingTable.cpp

These files provide global routing information necessary for correct next-hop computation. Router functions such as getNextHops and nextDeltaHops rely on this routing information when evaluating flow stability and predicting direction.


========================================================================================================================
GlobalStats.h and GlobalStats.cpp

These files track statistics related to PR behavior.

Typical counters:

ocs_reservations_created
Number of times a reservation was activated.

ocs_reservations_released
Number of times reservations were released.

ocs_bypassed_flits
Number of flits that used the fast-path bypass.

These counters allow evaluation of PR performance.


















---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

YAML Configuration Terms (ocs.yaml)

mesh_dim_x, mesh_dim_y
Dimensions of the NoC mesh. PR works on any size but shows more benefits on larger meshes.

min_packet_size, max_packet_size
Packet length in flits. Larger packet sizes give PR more room to identify stable flows.

packet_injection_rate
Average load per node. Higher PIR stresses the network and helps evaluate PR benefits.

probability_of_retransmission
Used if retransmission is enabled; unrelated to PR.

traffic_distribution
Specifies the traffic pattern (random, transpose, hotspot, bit reversal, etc.). PR tends to benefit from patterns with directional stability.

ocs_enable
true enables Path Reservation / OCS bypass mode.
false disables it and uses plain wormhole routing.

ocs_debug
true prints reservation creation and release events.
Useful for debugging.

ocs_pred_min
Minimum prediction score required to activate a reservation. This is a measure of flow stability.

ocs_min_burst
Minimum expected number of flits needed before allowing a reservation. Prevents reserving paths for short bursts.

ocs_credit_req
Minimum number of downstream buffer credits required before forming a reservation. Prevents reserving into congested routers.

ocs_idle_release
Number of idle cycles after which a reservation is released. Ensures the reservation does not persist unnecessarily.














---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step-by-Step Guide to Run Path Reservation

Step 1: Build the simulator
Run your make or cmake build process according to your project’s build system.

Step 2: Prepare the configuration
Edit ocs.yaml.
Set ocs_enable to false for baseline runs.
Set ocs_enable to true to enable Path Reservation.
Tune ocs_pred_min, ocs_min_burst, ocs_credit_req, and ocs_idle_release as needed.

Step 3: Running the YAML file
Open the terminal shell
2. Go to the noxim/bin directory
3. type:
./noxim -config ../config_examples/ocs.yaml
