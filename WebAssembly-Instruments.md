## WebAssembly

MSFS supports running WebAssembly for aircraft gauages.

WebAssembly is a sandboxed, deterministic and cross-platform bytecode format that is used as a compile target for many programming languages including C, C++, Rust, Go, Kotlin and more.

The only official toolchain supported by Microsoft / Asobo is using C++, Visual Studio 2019 and Clang. Support for other toolchains / languages has been discussed by community members and seems to be the interest of some developers.

### C++ example

```c++
#include <MSFS/MSFS.h>
#include <MSFS/MSFS_Render.h>
#include <MSFS/Legacy/gauges.h>
#include <SimConnect.h>

enum GROUP_ID
{
	ELEVATOR_GROUP,
	AILERON_GROUP,
	RUDDER_GROUP
};

enum EVENT_ID
{
	// Elevator Group
	AXIS_ELEVATOR_SET_EVENT,
	// Aileron Group
	AXIS_AILERONS_SET_EVENT,
	CENTER_AILERONS_RUDDER_EVENT,
	// Rudder group
	AXIS_RUDDER_SET_EVENT,
	RUDDER_SET_EVENT,
	RUDDER_CENTER_EVENT
};

enum DEFINITION_ID
{
	CONTROL_SURFACES_DEFINITION
};

/*
 * Variables used for querying the state of the aircraft
 */
struct SIM_VARS
{
	// TODO: Place SimVars here that need to be queried
} sim_vars;

/*
 * Describes the current position of the control surfaces (yoke + rudder) 
 */
struct CONTROL_SURFACES
{
	double elevator; // -1 is full down, and +1 is full up
	double aileron; // -1 is full left, and +1 is full right
	double rudder; // -1 is full left, and +1 is full right
} control_surfaces;

/*
 * Describes the current user input of the control surfaces (yoke + rudder)
 */
struct USER_INPUT
{
	double yoke_y; // -1 is full down, and +1 is full up
	double yoke_x; // -1 is full left, and +1 is full right
	double rudder; // -1 is full left, and +1 is full right
} user_input;

HANDLE hSimConnect;

void RegisterSimVars()
{
	// TODO: Put data in the SIM_VARS structure as necessary
	// Simulation variables reference: http://www.prepar3d.com/SDKv3/LearningCenter/utilities/variables/simulation_variables.html
	SimConnect_AddToDataDefinition(hSimConnect, CONTROL_SURFACES_DEFINITION, "ELEVATOR POSITION", "Position");
	SimConnect_AddToDataDefinition(hSimConnect, CONTROL_SURFACES_DEFINITION, "AILERON POSITION", "Position");
	SimConnect_AddToDataDefinition(hSimConnect, CONTROL_SURFACES_DEFINITION, "RUDDER POSITION", "Position");
}

void RegisterInputCapture()
{
	// Client events reference: http://www.prepar3d.com/SDKv3/LearningCenter/utilities/variables/event_ids.html
	// Elevator group
	SimConnect_MapClientEventToSimEvent(hSimConnect, AXIS_ELEVATOR_SET_EVENT, "AXIS_ELEVATOR_SET");
	SimConnect_AddClientEventToNotificationGroup(hSimConnect, ELEVATOR_GROUP, AXIS_ELEVATOR_SET_EVENT, TRUE);

	// Aileron group
	SimConnect_MapClientEventToSimEvent(hSimConnect, AXIS_AILERONS_SET_EVENT, "AXIS_AILERONS_SET");
	SimConnect_MapClientEventToSimEvent(hSimConnect, CENTER_AILERONS_RUDDER_EVENT, "CENTER_AILER_RUDDER");
	SimConnect_AddClientEventToNotificationGroup(hSimConnect, AILERON_GROUP, AXIS_AILERONS_SET_EVENT, TRUE);
	SimConnect_AddClientEventToNotificationGroup(hSimConnect, AILERON_GROUP, CENTER_AILERONS_RUDDER_EVENT, TRUE);

	// Rudder group
	SimConnect_MapClientEventToSimEvent(hSimConnect, AXIS_RUDDER_SET_EVENT, "AXIS_RUDDER_SET");
	SimConnect_MapClientEventToSimEvent(hSimConnect, RUDDER_SET_EVENT, "RUDDER_SET");
	SimConnect_MapClientEventToSimEvent(hSimConnect, RUDDER_CENTER_EVENT, "RUDDER_CENTER");
	SimConnect_AddClientEventToNotificationGroup(hSimConnect, RUDDER_GROUP, AXIS_RUDDER_SET_EVENT, TRUE);
	SimConnect_AddClientEventToNotificationGroup(hSimConnect, RUDDER_GROUP, RUDDER_SET_EVENT, TRUE);
	SimConnect_AddClientEventToNotificationGroup(hSimConnect, RUDDER_GROUP, RUDDER_CENTER_EVENT, TRUE);

	// Set maskable notification priorities
	SimConnect_SetNotificationGroupPriority(hSimConnect, ELEVATOR_GROUP, SIMCONNECT_GROUP_PRIORITY_HIGHEST_MASKABLE);
	SimConnect_SetNotificationGroupPriority(hSimConnect, AILERON_GROUP, SIMCONNECT_GROUP_PRIORITY_HIGHEST_MASKABLE);
	SimConnect_SetNotificationGroupPriority(hSimConnect, RUDDER_GROUP, SIMCONNECT_GROUP_PRIORITY_HIGHEST_MASKABLE);
}

void HandleControlSurfaces()
{
	// For now, let's make the users' position the direct position of the elevators/ailerons/rudder (i.e. direct law)
	control_surfaces.elevator = user_input.yoke_y;
	control_surfaces.aileron = user_input.yoke_x;
	control_surfaces.rudder = user_input.rudder;
	SimConnect_SetDataOnSimObject(hSimConnect, CONTROL_SURFACES_DEFINITION, SIMCONNECT_OBJECT_ID_USER, 0, 0, sizeof(control_surfaces), &control_surfaces);
}

void CALLBACK OnSimConnectEvent(SIMCONNECT_RECV* pData, DWORD cbData, void *pContext)
{
	if (pData->dwID == SIMCONNECT_RECV_ID_EVENT)
	{
		auto evt = static_cast<SIMCONNECT_RECV_EVENT*>(pData);
		switch (evt->uEventID)
		{
			case AXIS_ELEVATOR_SET_EVENT:
				user_input.yoke_y = 0 - (static_cast<long>(evt->dwData) / 16384.0); // scale from [-16384,16384] to [-1,1] and reverse the sign
				break;
			case AXIS_AILERONS_SET_EVENT:
				user_input.yoke_x = 0 - (static_cast<long>(evt->dwData) / 16384.0); // scale from [-16384,16384] to [-1,1] and reverse the sign
				break;
			case CENTER_AILERONS_RUDDER_EVENT:
				user_input.yoke_x = 0;
				user_input.rudder = 0;
				break;
			case AXIS_RUDDER_SET_EVENT:
				user_input.rudder = 0 - (static_cast<long>(evt->dwData) / 16384.0); // scale from [-16384,16384] to [-1,1] and reverse the sign
				break;
			case RUDDER_SET_EVENT:
				user_input.rudder = 0 - (static_cast<long>(evt->dwData) / 16384.0); // scale from [-16384,16384] to [-1,1] and reverse the sign
				break;
			case RUDDER_CENTER_EVENT:
				user_input.rudder = 0;
				break;
		}
	}
}

extern "C"
{
	MSFS_CALLBACK bool FBW_gauge_callback(FsContext ctx, int service_id, void* pData)
	{
		switch (service_id)
		{
		case PANEL_SERVICE_PRE_INSTALL:
		{
			// Sent before the gauge is installed.
			printf("A32NX_FBW: Installing\n");
			return true; // TODO: Determine if this is this necessary? The reference gauges have this. Does this indicate a successful install?
		}
		case PANEL_SERVICE_POST_INSTALL:
		{
			// Sent after the gauge is installed.

			// Connect to sim using SimConnect
			if (SUCCEEDED(SimConnect_Open(&hSimConnect, "A32NX_FBW", nullptr, 0, 0, 0)))
			{
				printf("A32NX_FBW: Connected via SimConnect\n");
				RegisterSimVars();
				RegisterInputCapture();
				return true;
			}
			printf("A32NX_FBW: Failed to connect via SimConnect\n");
			return false;
		}
		case PANEL_SERVICE_PRE_DRAW:
		{
			// Sent before the gauge is drawn. The pData parameter points to a sGaugeDrawData structure:
			// - The t member gives the absolute simulation time.
			// - The dt member gives the time elapsed since last frame.
			// TODO: Use the above variables as time references for PID controllers
			SimConnect_CallDispatch(hSimConnect, OnSimConnectEvent, nullptr);
			HandleControlSurfaces();
			return true;
		}
		case PANEL_SERVICE_PRE_KILL:
		{
			// Sent before the gauge is deleted.
			if (hSimConnect)
			{
				SimConnect_Close(hSimConnect);
				hSimConnect = 0;
			}
			return true;
		}
		}
		return false;
	}
}
```

### Compiled wasm module (text format)

https://pastie.io/oavdor.txt
