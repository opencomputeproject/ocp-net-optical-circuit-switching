# Expected model usage and conventions

## Optical Switch (OCS) Modelling

The global characteristics and capabilities of an Optical Circuit Switch (OCS) shall be modelled under the `optical-switch` container. Configuration parameters shall be placed under `optical-switch/config`, while device capabilities and operational information shall be reported under `optical-switch/state`.

### `ocp-ocs-optical-switch:optical-switch/config`

The `optical-switch/config` container contains configurable global behavior:

`bidirectional` may be configured when the device supports selectable unidirectional or bidirectional operation.
`cold-boot-recovery` selects the required handling of connections during a cold boot.

The corresponding leaves under `optical-switch/state` shall report the effective operational values and advertised capabilities. Where a parameter is not configurable, its value may be reported only under state.

### `ocp-ocs-optical-switch:optical-switch/state`

The state container shall advertise the capabilities of the switch. These capabilities may be used by a controller to determine whether a requested connection or operating mode is supported before attempting to configure it.

### `ocp-ocs-optical-switch:optical-switch/state/bidirectional`

The `bidirectional` leaf shall indicate whether the switch supports bidirectional connections. In a bidirectional switch, a single connection between two ports permits optical traffic to flow in both directions. In a unidirectional switch, a connection permits traffic to flow only from the input-port-name to the output-port-name specified in the connection. This leaf describes switch capability; it does not indicate the direction of an individual connection.

### `ocp-ocs-optical-switch:optical-switch/state/connection-recovery-capability`

The `connection-recovery-capability` leaf shall identify the cold-boot recovery modes supported by the device. The supported values are:

* `MAINTAIN`: Existing connections persist through a cold boot, and traffic is not interrupted.
* `RESTORE`: Existing connections are interrupted during the cold boot but are automatically re-established afterward.
* `REMOVE`: Existing connections are lost during the cold boot and are not automatically re-established.

### `ocp-ocs-optical-switch:optical-switch/state/cold-boot-recovery`

The `cold-boot-recovery` configuration leaf shall select the behavior used by the device, subject to the capabilities advertised by connection-recovery-capability. A device shall reject a configured mode that it does not support.

### `ocp-ocs-optical-switch:optical-switch/state/port-groups`

The `port-groups` list shall describe the switch-fabric topology and the connection constraints imposed by the OCS architecture. A port group represents a set of ports that share common switching characteristics. The `direction` leaf indicates the permitted connection endpoint role of ports in the group:

* `INPUT-ONLY` ports shall be specified in the `input-port-name` field
* `OUTPUT-ONLY` ports shall be specified in the `output-port-name` field
* `ANY` ports may be specified in either field

The `connection-groups` list identifies the other port groups to which ports in the group may be connected without being blocked by the switch fabric. The `port-group` information describes the capabilities and topology of the switch; it does not describe the current connection state of individual ports.

`Port-group` names are device-defined identifiers and shall be stable for the lifetime of the device configuration. The `connection-groups` entries shall reference valid port groups advertised in the same `optical-switch/state/port-groups` list. A missing `peer-group` reference indicates that connections between the two groups are not supported by the switch fabric.

#### `port-groups` Examples

An OCS that supports any-to-any connectivity may be represented by a single port group. For example:
* Port group GroupA
* Direction: ANY
* Connection group: GroupA

This indicates that any port in GroupA may be connected to any other compatible port in the same group.

An OCS with three groups—A, B, and C—where connections are supported between A and B, A and C, and within C, may be represented as follows:
* Group A: connection groups B and C
* Group B: connection group A
* Group C: connection groups A and C

The connection-group relationships should describe the permitted switching relationships in both directions where applicable. For example, if ports in group A can connect to ports in group B, group A should reference group B, and group B should reference group A, unless the implementation intentionally models a unidirectional relationship.

A device containing multiple independent OCS fabrics may be modelled by defining separate port groups for each fabric. For example, a device containing four independent OCS units, where each unit provides connectivity between groups A and B, may be represented as:
* A1 with connection group B1
* B1 with connection group A1
* A2 with connection group B2
* B2 with connection group A2
* A3 with connection group B3
* B3 with connection group A3
* A4 with connection group B4
* B4 with connection group A4

This representation indicates that connections are permitted within each independent OCS fabric, but not between separate fabrics. For example, a port in A1 may connect to a port in B1, but not to a port in B2.

## Optical Switch Connections Modelling

Optical cross-connections shall be modelled in the `connections/port-connection` list. Each list entry represents one statement of connection intent between two logical OCS ports. The ports shall be referenced using `input-port-name` and `output-port-name`, which shall reference elements in the `oc-platform:components` tree that contain the `ocp-ocs-port` augmentation.

### `ocp-ocs-optical-switch-connections:connections/last-change-time`

The `connections/last-change-time` leaf shall indicate the date and time of the most recent status change affecting a connection in the switch.

### `ocp-ocs-optical-switch-connections:connections/port-connection`

Each connection shall have a controller-selected name that uniquely identifies the connection within the OCS. The name shall remain stable while the connection exists and shall be used to correlate configuration, operational state, alarms, and status changes.

### `ocp-ocs-optical-switch-connections:connections/port-connection/config`

The `config` container shall define the intended connection:

* `name`: Identifier assigned to the connection.
* `input-port-name`: Port to be used as the input endpoint.
* `output-port-name`: Port to be used as the output endpoint.

### `ocp-ocs-optical-switch-connections:connections/port-connection/state`

The `state` container shall report the effective operational values and status of the connection. It shall include the configured connection information together with the following operational data:

* `status`: Current status of the connection.
* `status-reason`: Additional information explaining a `FAILED` or `INVALID` status.
* `last-updated`: Date and time of the most recent connection status transition.

The `port-connection/state` data shall provide the operational status of the connection as reported by the device. The corresponding OCS port state shall identify the current peer port, whether the optical connection is active, and any supported connection-specific optical error counters.

### `ocp-ocs-optical-switch-connections:connections/port-connection/state/status`

The status leaf shall describe the lifecycle and operational condition of the connection:

* `CONNECTED`: The connection has been successfully established and the optical path is active.
* `PENDING_CONNECTION`: The device is in the process of establishing the connection.
* `PENDING_DELETION`: The device is in the process of removing the connection.
* `FAILED`: The connection represents valid intent, but the intended optical path has failed operationally.
* `INVALID`: The connection intent is invalid or it represents valid intent, but cannot currently be established because one or more of its ports are blocked by another active (CONNECTED) connection.

A connection in the `INVALID` state shall remain represented in the configuration and operational data as valid connection intent. The device shall provide additional information in status-reason identifying the condition preventing establishment. For example, a port may be unavailable because it is already assigned to another active connection.

A connection in the `FAILED` state differs from an `INVALID` connection in that the requested connection is not blocked by another active connection, but the device has detected an operational failure preventing the optical path from being established or maintained.

## Port Modelling

Ports shall be modelled as individual elements in the `oc-platform:components` tree.  The `ocp-ocs-port` type shall be used to hold additional OCS specific configuration and state information.  A modelled port shall be a single logical element that can be used as one end point of a cross connect.  In the case of switches with multi-fiber connectors, the mapping of logical ports to physical fibers/connectors will depend on switch capabilities.  A device which forms cross connects at the single fiber level will expose logical ports for all fibers, even if they are physically grouped into multi-fiber connectors.  A device which forms cross connects at the connector level will expose logical ports for all connectors, even if each connector contains multiple fibers.

### Expected usage of existing Open Config component tree

A number of existing Open Config `component` config and state leaves are applicable to OCS ports.

* `Name` - port name defined by the OCS
* `type` - shall be **XXX need to extend oc-platform-types**
* `install-position`
* `install-component`
* `description`
* `parent`

### `ocp-ocs-port` augmentation

### `oc-platform:components/component/ocp-ocs-port`

The `ocp-ocs-port` augmentation shall be applied to the relevant elements in the `oc-platform:components` tree. For each OCS port, the augmentation shall provide:

* Administrative configuration, including whether the port is enabled.
* An optional user-defined alias and description.
* Device-defined port-group membership.
* Device-defined port index.
* Current port status.
* Input and output optical power, where supported.
* The peer port associated with the current connection.
* Whether the port is currently connected.
* Connection-specific and port-level error counters, where supported.

### `oc-platform:components/component/ocp-ocs-port/state`

### `oc-platform:components/component/ocp-ocs-port/state/connection`

The `ocp-ocs-port/state/connection/peer` leaf shall identify the port at the opposite end of the current connection. The `connection/connected` leaf shall indicate whether the optical path through the switch core is currently active. These leaves describe the current operational association and shall not be interpreted as an additional connection-configuration mechanism.

The `ocp-ocs-port/state/status` leaf shall describe the current availability of an individual port:

* `BLOCKED`: The port is not currently part of an active connection, or is disabled.
* `FAILED`: The port has failed and is not currently available for a new connection.
* `TUNING`: The port is undergoing a transient operation and its intended connection is not yet complete.
* `TUNED`: The port is in the operational state associated with its current intended connection.
* `UNKNOWN`: The device cannot determine the current port state.

A device shall maintain consistency between the connection list and the port state. When a connection is active, its endpoint ports should identify each other as peers, and the connection should be reported as `CONNECTED` with the associated ports reported in their corresponding operational states. When a connection is deleted, its ports shall no longer report that connection as their current peer after deletion has completed.

### Port Naming

Guidance on how to name ports (not mandatory)

TBD

### Port Error Counters

Port error counters are intended to provide insight into internal device errors related to establishing or maintaining optical connections.  This functionality is hardware dependant and not all OCS’s may support it.  There are two types of error counts defined and two sets of counters.

`correctable-error-count` is intended to count events where an error occurred but it was later automatically recovered and the connection was established/re-established.
`total-error-count` is intended to count all error events.

The set of counters defined under `ocp-ocs-port/state/connection/counters` are intended to represent errors occurring with the currently provisioned connection.  This set of counters resets to zero whenever the connection related to the port is reprovisioned.
The set of counters defined under `ocp-ocs-port/state/counters` are intended to represent the total number of errors associated with the port across all connections since device boot up.

Not all OCSs may have the hardware capability to either detect errors or associate them with specific ports.  If the OCS has no capability to detect such errors, all counters must report zero.  If the OCS can detect specific connection errors but cannot localize them to a particular port, they must be counted against the ports at both ends of the connection.  If the OCS can detect errors but not localize them to a specific connection, all counters must report zero.

## Amplification

The OCP model does not specifically support managing any embedded amplification within an OCS.  It is recommended that devices use the model defined in the existing `openconfig-optical-amplifier` module to model and control any embedded amplifiers.

## Expected device behavior

### Default settings

Default setting for connection directionality (`ocp-ocs-optical-switch:optical-switch/config/bidirectional`) shall be bi-directional unless the device is not physically capable of bi-directional connections.

Default setting for connection recovery on cold boot (`ocp-ocs-optical-switch:optical-switch/config/cold-boot-recovery`) shall be `REMOVE`, indicating that no connections should be automatically created when a device initially boots.

### Restarts

Devices must support the following restart behaviors.

#### Warm restarts

All configuration settings must be preserved across a warm restart.  All active optical connections must be maintained with no interruption.

#### Cold restarts

All active optical connections must be handled according to the setting for `ocp-ocs-optical-switch:optical-switch/config/cold-boot-recovery` prior to restart.  If set to `MAINTAIN` or `RESTORE`, the `ocp-ocs-optical-switch-connections:connections` tree must be repopulated to match the state prior to restart.
Power cycles must be treated the same as commanded cold restarts.

### Connection configuration updates

The general set of operations that should happen when the connections tree is updated is as follows:

Step 1:  Analyze requested connections for validity

Invalid connections can be due to:
Invalid syntax that doesn’t conform to the data model.
Specifying port names that don’t exist.
Trying to make connections using ports that are already in use in another connection.
Trying to make connections between ports that violate port group connectivity rules.
Trying to use ports belonging to an input port group as output ports within a connection or trying to use ports belonging to an output port group as input ports within a connection.

In general, it is desirable to detect invalid connections at the highest possible SW layer and reject the entire gNMI set operation if any portion of the configuration is invalid.  In practice, this may or may not be possible depending on the specific system implementation.  If the gNMI request is accepted and some portion of the configuration is later determined to be invalid, the entry for the invalid connection shall be created in the `ocp-ocs-optical-connections:connections/port-connection` tree and the connection status shall be reported as `INVALID`.  The `status-reason` should report detail about why the connection is invalid.  It is the controller’s responsibility to delete invalid connections.

Step 2:  Analyze requested connections against existing physical state

Compare the requested connection list against the current physical state of the device to determine any requested connections that already exist, any current connections that are not longer requested and any requested connections that need to be created.

Step 3:  Maintain any requested connections which already exist

No existing connections shall be disrupted due to a provisioning request that does not explicitly delete those connections.

Step 4:  Delete any physical connections which no longer exist in the new configuration

Step 5:  Create any new connections which are not in the existing physical state

## Port state diagram

![Port status state diagram](/port_state_diagram.png)

## Connection state diagram

![Connection status state diagram](/connection_state_diagram.png)

