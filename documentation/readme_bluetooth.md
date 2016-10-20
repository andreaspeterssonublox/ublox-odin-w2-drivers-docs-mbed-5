# u-blox Embedded Bluetooth Stack
The u-blox Embedded Bluetooth Stack is optimized for small embedded industrial systems with high requirements on performance and robustness. It is also pre-qualified for Bluetooth 4.0+EDR and contains the following classic profiles; SPP, DUN, PAN (role PANU and NAP,  DID and GAP. For Bluetooth low energy GAP, GATT and DIS are pre-qualified.
Direct function calls and callbacks are used as much as possible and message passing and critical sections are used sparsely.
The block diagram below shows the different modules in the stack.

![](mbed_ble_c_stack_system_overview.png)

The top modules cbBM, cbBSM, cbBCM, cbBSE, cbBSL, cbGATT and cbBTPAN above are exported via the API and they are described below. The other modules are internal and thus not accessible by the application. The functions/callbacks are described in the header files.

## General initialization
The Bluetooth stack is started by calling cbMAIN\_initBt. Note that cbBM_init must not be called from the application since this is done from inside cbMAIN\_initBt.
After cbBM_init has been called the application must wait for the complete callback before issuing any other function calls to the driver.

## Bluetooth Manager(cbBM)
The Bluetooth Manager takes care of static configuration of the Bluetooth stack e.g. local name, class of device, connectable modes etc. Procedures for inquiry and device name are also found here.
## Bluetooth Connection Manager(cbBCM)
This module contains functionality for establishing and tear down a connection to a remote Bluetooth device. The connection can be of different types, ACL connection, low energy connection or profile connection e.g. SPP.

Several connections can be active in parallel. They are separated by the use of a handle which is provided by the stack upon outgoing/incoming connection.

When an ACL/profile connection is set up, security might be needed depending on what Bluetooth version and security requirements the remote and local side is using. The security requirements depend on what protocol/profile is being used. For example, when using SDP it is likely that no security is needed while setting up RFCOMM will involve security. See Bluetooth Security chapter for more info on how security works.

Once a connection has been established, data can be sent to and from the remote device using the Bluetooth Serial(cbBSE) component.
This component also takes care of registering SDP database records and service searching.

An SDP client is a device that is searching for remote services while an SDP server is holding a service record to be found by remote devices. Therefore, if a device acts only as a client, it does not need to register an SDP service record. A single device can function both as an SDP server and as an SDP client.

Note that service records can only be added, not removed. When the device is reset all service records are removed.

The sequence diagram below shows the calls and events used in a successful connection setup and disconnect of a Bluetooth Serial Profile link. The connection is initiated by the application using the Connection Manager and when the connection is established the application receives the connect event callback and opens a corresponding data channel in Bluetooth Serial. When a disconnect occurs the application receives a disconnect event from the Connection Manager.

![](mbed_bt_spp_conn_setup.png)

## Bluetooth Serial(cbBSE)
The Bluetooth Serial component provides a generic data packet interface for SPP and DUN. After a connection has been set up using the Connection Manager a data packet channel can be opened. The data packet channel will be automatically closed when a disconnection occurs. The sequence diagram below shows how data is sent to and received from a remote device using a Bluetooth Serial Port link. For each write operation a write confirmation is received. When data is received from the remote device the application receives a data available event. When the application receives the data available event it fetches the data using the cbBSE_getReadbuf function and then, when the data has been processed calls the cbBSE_readBufConsumed function to notify that the underlying buffers can be reused. Note that the sample code for connection establishment and data transfer is available in the sample applications.

![](mbed_bt_spp_data_transfer.png)

## Bluetooth Security(cbBSM)
There are 7 different security modes to support all kinds of use cases regarding the pairing procedure:
```c
typedef enum
{
    cbBSM_SECURITY_MODE_1_DISABLED = 1,
    cbBSM_SECURITY_MODE_2_BT_2_0,
    cbBSM_SECURITY_MODE_3_FIXED_PIN,
    cbBSM_SECURITY_MODE_4_JUST_WORKS,
    cbBSM_SECURITY_MODE_5_DISPLAY_ONLY,
    cbBSM_SECURITY_MODE_6_DISPLAY_YES_NO,
    cbBSM_SECURITY_MODE_7_KEYBOARD_ONLY
} cbBSM_SecurityMode;
```
Each mode is specified for both Bluetooth 2.0+EDR and 2.1+EDR security since both must be supported depending on what the remote device supports.
All security modes except Security Mode 1 (Security Disabled) for Bluetooth 2.0+EDR devices uses encryption. Hence, security mode 1 (Security Disabled) for Bluetooth 2.1+EDR still uses encryption.
Please note that the Security Mode for the ODIN-W2 does not directly correspond to the Security Mode of the Bluetooth specification.

#### cbBSM_SECURITY_MODE_1_DISABLED
- Remote Device BT 2.1: Auto accept (No man-in-the-middle attack protection, encryption enabled)
- Remote Device BT 2.0: Authentication and encryption disabled.
- Bluetooth Low Energy: Auto accept (No man-in-the-middle attack protection, encryption enabled)

#### cbBSM_SECURITY_MODE_2_BT_2_0

- Enforce BT 2.0 (Service level authentication and encryption enabled)

Please note that the device is not BT 2.1 qualified for this setting. It is included for backward compatibility. Invalid for Bluetooth Low Energy.

#### cbBSM_SECURITY_MODE_3_FIXED_PIN

- Remote Device BT 2.1: Service level authentication and encryption enabled.
- Remote Device BT 2.0: Service level authentication and encryption enabled.
- Bluetooth Low Energy: Service level authentication and encryption enabled.

Please note that this security mode will not work with a general BT 2.1 device. However, it will work between two connectBlue BT 2.1 Serial Port Adapters. Use security mode 4 to make the device work with a general BT 2.1 device.

#### cbBSM_SECURITY_MODE_4_JUST_WORKS

- Remote Device BT 2.1: Auto accept (no man-in-the-middle attack protection, encryption enabled) 
- Remote Device BT 2.0: Service level authentication and encryption enabled. 
- Bluetooth Low Energy: Auto accept (no man-in-the-middle attack protection, encryption enabled)

This security mode is intended for pairing in safe environments. When this mode is set, pairability is automatically disabled.

#### cbBSM_SECURITY_MODE_5_DISPLAY_ONLY

- Remote Device BT 2.1: Service level authentication and encryption enabled. User should be presented a passkey. 
- Remote Device BT 2.0: Service level authentication and encryption enabled. No user interaction required. 
- Bluetooth Low Energy: Service level authentication and encryption enabled. User should be presented a passkey.

This security mode is used when the device has a display that can present a 6-digit value that the user shall enter on the remote device.

#### cbBSM_SECURITY_MODE_6_DISPLAY_YES_NO 

- Remote Device BT 2.1: Service level authentication and encryption enabled. User should compare two values. 
- Remote Device BT 2.0: Service level authentication and encryption enabled. No user interaction required.
 
This security mode is used when the device has a display that can present a 6-digit value that the user shall verify with yes or no to the remote device's presented value. 
Invalid for Bluetooth Low Energy.

#### cbBSM_SECURITY_MODE_7_KEYBOARD_ONLY

- Remote Device BT 2.1: Service level authentication and encryption enabled. User should enter a passkey. 
- Remote Device BT 2.0: Service level authentication and encryption enabled. No user interaction required. 
- Bluetooth Low Energy: Service level authentication and encryption enabled. User should enter a passkey.

This security mode is used when the device only has a keyboard where the user can enter a 6-digit value that is presented on the remote device.

## Bluetooth Personal-Area Network(cbBTPAN)
The PAN profile is used for sending and receiving ethernet frames over a Bluetooth connection. This is typically supported by mobile phones and laptops.

The PAN profile defines three roles:
- PAN User (PANU) and PANU Service - This is the Bluetooth device that uses either the NAP or the GN service.
- Network Access Point (NAP) and NAP Service – A Bluetooth device that supports the NAP service is a Bluetooth device that provides some of the features of an Ethernet bridge to support network services.
- Group Ad-hoc Network (GN) and GN Service – A Bluetooth device that supports the GN service is able to forward Ethernet packets to each of the connected Bluetooth devices, the PAN users, as needed.

PANU and NAP are supported by the ODIN-W2 drivers.

## Bluetooth low energy Serial Port Service(cbBSL)
The Bluetooth low energy Serial component provides a generic data packet interface on top of GATT. After a connection has been set up using the Connection Manager a data packet channel can be opened. The data packet channel will be automatically closed when a disconnection occurs. The usage is similar to cbBSE.

## Bluetooth low energy Generic Attribute Profile(cbBTGATT)
GATT is a profile used for setting and getting attribute data from a remote Bluetooth low energy enabled device. It is used by almost all profiles and services used in Bluetooth low energy. GATT for Classic Bluetooth is not supported by this platform.
The GATT API is almost a one to one mapping of the GATT profile specification. The header files cb_gatt.h, cb_gatt_client.h, and cb_gatt_server.h should work as a reference for the different GATT requests and responses. For all details see the Bluetooth 4.0 specification and for a GATT overview see https://developer.bluetooth.org/TechnologyOverview/Pages/GATT.aspx

GATT is part of the Bluetooth embedded stack and used by various components in the stack. Only one request at a time is supported from the application. The application must wait until all responses from an outstanding request have been received.
For more info see header files.

Note that service changed indications for the GATT server/peripheral are not supported. Thus the product must not alter it's attribute database during it's lifetime. The following functions affect the database:
- cbBCM_enableSps
- cbBCM_enableDevInfoService To avoid any changes in the database the functions should either be included or excluded in the application and this should not change during the lifetime of the product. The order must not change either.

## Bluetooth Test Manager(cbBTM)
This component is used for testing different radio characteristics and assure that radio approvals are followed.

## Bluetooth Qualification
The stack is pre-qualified for the following profiles and protocols:

- SPP  
- DUN  
- PAN (Role PANU and NAP)  
- BNEP  
- GAP  
- SDP  
- RFCOMM  
- L2CAP  
- HCI  
- DID  
- DIS  
- SM  
- ATT  
- GATT  

For more information on qualification please see the ODIN-W2 datasheet and System Integration Manual:  
[https://www.u-blox.com/en/product/odin-w2-series](https://www.u-blox.com/en/product/odin-w2-series)
