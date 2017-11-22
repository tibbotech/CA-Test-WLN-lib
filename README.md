# CA-Test-WLN-lib

To download the most recent project without installing GIT, please press the green "Clone or Download" button and select "Download ZIP".



About the Application
-------------------

This download contains four separate demo projects, corresponding to those described in the WLN (Wi-Fi Association) library documentation:



Step 1: The Simplest Example
-------------------

This step corresponds to **test_wln_lib_1**.

Let's start with a simple example of connecting to an access point named "TIBB1", which is configured for WEP64 security. The password is "12345678AB". All your code really has to do is start the "Wi-Fi engine" by calling wln_start().

To illustrate the use of callback procedures, we set green status LED on in ***callback_wln_ok()***. We turn this LED off in ***callback_wln_failure()***.

Debug output we've got after running the code:

```
WLN> ---START---
WLN> ACTIVE SCAN for TIBB1
WLN> ASSOCIATE with TIBB1 (bssid: 192.63.14.197.236.216, ch: 11)
WLN> ---OK(associated in WEP64, WEP128, or no-security mode)---
```
You can test the "association persistence" by turning your access point off and on. You will see how the library will restore the association:
```
WLN> ERROR: disassociation (or link loss with the access point)
WLN> ACTIVE SCAN for TIBB1
WLN> ERROR: access point not found
WLN> ACTIVE SCAN for TIBB1
WLN> ERROR: access point not found
...
WLN> ACTIVE SCAN for TIBB1
WLN> ASSOCIATE with TIBB1 (bssid: 192.63.14.197.236.216, ch: 11)
WLN> ---OK(associated in WEP64, WEP128, or no-security mode)---
```
And here is the code itself...

#### global.tbh:

```
'DEFINES-------------------------------------------------------------
#define WLN_DEBUG_PRINT 1
'For version of TIDE prior to 5.04.03, this value should be defined
'#define LIB_VER_2_10 'Define the library version here LIB_VER_2_00/LIB_VER_2_01/LIB_VER_2_03/LIB_VER_2_04/LIB_VER_2_10 

#if PLATFORM_ID=EM500 or PLATFORM_ID=EM500W or PLATFORM_ID=EM1202W
	#define WLN_RESET_MODE 1 'there will be no dedicated reset, and all other lines are fixed
#elif PLATFORM_ID=EM1206 or PLATFORM_ID=EM1206W or PLATFORM_ID=DS1101W or PLATFORM_ID=DS1102W
	#define WLN_CLK PL_IO_NUM_14
	#define WLN_CS PL_IO_NUM_15
	#define WLN_DI PL_IO_NUM_12
	#define WLN_DO PL_IO_NUM_13
	#define WLN_RST PL_IO_NUM_11
#else
	'EM1000, NB1010,...
	#define WLN_CLK PL_IO_NUM_53
	#define WLN_CS PL_IO_NUM_49
	#define WLN_DI PL_IO_NUM_52
	#define WLN_DO PL_IO_NUM_50
	#define WLN_RST PL_IO_NUM_51
#endif

'INCLUDES------------------------------------------------------------
include "sock\trunk\sock.tbh" 'this lib is necessary for the WLN lib's operation
include "wln\trunk\wln.tbh"
```
#### main.tbs:

```
include "global.tbh"

'====================================================================
sub on_sys_init()
	
	#if SRC_LIB_VER < &h020003
		if wln_start("TIBB1",WLN_SECURITY_MODE_WEP64,"12345678AB",PL_WLN_DOMAIN_FCC)<>WLN_STATUS_OK then
			sys.halt
		end if
	#endif

	#if SRC_LIB_VER >= &h020003
		if wln_start("TIBB1",WLN_SECURITY_MODE_WEP64,"12345678AB",PL_WLN_DOMAIN_FCC,YES,PL_WLN_ASCAN_INFRASTRUCTURE)<>WLN_STATUS_OK then
			sys.halt
		end if
	#endif
		
end sub

'--------------------------------------------------------------------
sub on_sys_timer()
	wln_proc_timer()
end sub

'--------------------------------------------------------------------
sub on_sock_data_arrival()
	wln_proc_data()
end sub

'--------------------------------------------------------------------
sub on_wln_task_complete(completed_task as pl_wln_tasks)
	wln_proc_task_complete(completed_task)
end sub

'--------------------------------------------------------------------
sub on_wln_event(wln_event as pl_wln_events)
	wln_proc_event(wln_event)
end sub
```
#### device.tbs:

```
include "global.tbh"

'====================================================================
sub callback_wln_ok()
	pat.play("G~",PL_PAT_CANINT)
end sub

'--------------------------------------------------------------------
sub callback_wln_failure(wln_state as en_wln_status_codes)
	pat.play("-",PL_PAT_CANINT)
end sub

'--------------------------------------------------------------------
sub callback_wln_pre_buffrq(required_buff_pages as byte)
end sub

'--------------------------------------------------------------------
sub callback_wln_rescan_result(current_rssi as byte, scan_rssi as byte, different_ap as no_yes)
end sub

sub callback_wln_starting_association()
end sub

sub callback_wln_rescan_for_better_ap()
end sub
```





Step 2: Adding TCP Comms
-------------------

This step corresponds to **test_wln_lib_2**.

Now let's take the previous example and create a TCP socket that will accept connections over the Wi-Fi interface. We will make a simple loopback socket that echoes back the data.



> **Note:**
> May we suggest our very own [I/O NINJA](http://ioninja.com/) terminal/sniffer software as the tool for testing the loopback socket? Hint: use Connection Socket plugin.



Only main.tbs needs to be changed from the previous example. The socket is configured in ***on_sys_init()***. Notice how we don't just pick a random socket number to use. We obtain the socket from the SOCK library by calling sock_get(). This is very important! Once any part of your project uses the **SOCK library** (and **WLN** library does), all other parts of your project must do the same. Mess will ensue if you fail to handle socket numbers correctly.

In the code below, we first call ***wln_start()*** and then configure the loopback socket. There is no significance to this order and you can reverse it. Same goes for ***on_sock_data_arrival()***. There is no special reason to call ***wln_proc_data()*** first, and handle the loopback socket next. You can reverse the order and everything will still work.

#### main.tbs:

```

include "global.tbh"

dim tcp_sock as byte

'====================================================================
sub on_sys_init()
	wln.ip="192.168.1.50" '<--- set suitable IP here
	
	#if SRC_LIB_VER < &h020003
		if wln_start("TIBB1",WLN_SECURITY_MODE_WEP64,"12345678AB",PL_WLN_DOMAIN_FCC)<>WLN_STATUS_OK then
			sys.halt
		end if
	#endif

	#if SRC_LIB_VER >= &h020003
		if wln_start("TIBB1",WLN_SECURITY_MODE_WEP64,"12345678AB",PL_WLN_DOMAIN_FCC,YES,PL_WLN_ASCAN_INFRASTRUCTURE)<>WLN_STATUS_OK then
			sys.halt
		end if
	#endif
	
	'configure loopback socket
	tcp_sock=sock_get("TCP")
	sock.num=tcp_sock
	sock.rxbuffrq(1)
	sock.txbuffrq(1)
	sys.buffalloc
	sock.protocol=PL_SOCK_PROTOCOL_TCP
	sock.localportlist="1000"
	sock.allowedinterfaces="WLN"
	sock.inconmode=PL_SOCK_INCONMODE_ANY_IP_ANY_PORT
	sock.reconmode=PL_SOCK_RECONMODE_3
end sub

'--------------------------------------------------------------------
sub on_sys_timer()
	wln_proc_timer()
end sub

'--------------------------------------------------------------------
sub on_sock_data_arrival()
	wln_proc_data()

	'loopback tcp data
	if sock.num=tcp_sock then
		sock.setdata(sock.getdata(sock.txfree))
		sock.send
	end if
end sub

'--------------------------------------------------------------------
sub on_wln_task_complete(completed_task as pl_wln_tasks)
	wln_proc_task_complete(completed_task)
end sub

'--------------------------------------------------------------------
sub on_wln_event(wln_event as pl_wln_events)
	wln_proc_event(wln_event)
end sub
```





Step 3: Trying WPA
-------------------

This step corresponds to **test_wln_lib_3**.

OK then, now let's try to associate with an access point configured for WPA2-PSK security. Your can try WPA-PSK by yourself.



> **Note:**
> If your access point is configured for WPA-PSK/WPA2-PSK mode, select WPA2 on the device side.



With WPA, things get a bit tricker because of a pre-shared master key (PMK). This is the actual password used between your device and the access point. The key is calculated from a "human" password (that you define) and the access point's SSID (name). ***Wln_wpa_mkey_get()*** does the math. The bad news is that it takes 2 minutes... yep, sorry, 8000 iterations involving sha1 need that long. The good news is that you only need to do this once after changing the password or SSID. You can then just save the resulting key for the future use. **STG library** comes handy for this.

Our project now has a new function — connect_to_ap() — that calls ***wln_start()***. This function takes all the same arguments as wln_start(). Connect_to_ap() stores the SSID, human password, and pre-shared master key into the EEPROM.  Wln_wpa_mkey_get() is only called if the SSID or human password change, or if the "PMK" setting is invalid. For the "PMK" setting to be valid, it must contain a 32-character string — the PMK always has this length. Keeping the PMK in the EEPROM allows us to avoid recalculating it — a huge time saver. ***Callback_wln_mkey_progress_update()*** is called repeatedly from within wln_wpa_mkey_get() while the PMK calculation is in progress. We put pat.play there to assure the user that the system isn't dead.

Below is the debug output we've got when connecting to the access point named "TIBB1".

```
WLN> ---START---
WLN> ACTIVE SCAN for TIBB1
WLN> ASSOCIATE with TIBB1 (bssid: 192.63.14.197.236.216, ch: 11)
WLN> Pre-associated for WPA1-PSK or WPA2-PSK, starting handshake process
WLN> Pairwise handshake, process step 1 of 4
WLN> Pairwise handshake, process step 2 of 4
WLN> Pairwise handshake, process step 3 of 4
WLN> Pairwise handshake, process step 4 of 4
WLN> ---OK(associated in WPA2-PSK mode)---
```



> **Note:**
> Some access points, notably CISCO devices, are so impatient during the WPA1/WPA2 handshaking process, that printing debug info makes them timeout. If you notice that the WLN library is stuck in the endless association loop, and you are sure that your password is set correctly, then try disabling debug printing! It may be the reason why the association fails.



The new iteration of our project is listed below. Notice how we set #define WLN_WPA 1 in global.tbh. This is necessary to enable WPA support.

We show one more trick in this example. Instead of having a listening TCP socket, we have a loopback socket that connects to a certain destination IP whenever there is a successful association. To ensure persistent connection, we put a polling code into on_sys_timer. Not the most beautiful of solutions, but gets the job done. The TCP connection is discarded whenever we get callback_wln_failiure().



> **Note:**
> Test outgoing connections by setting sock.targetip to your PC's address and using our I/O NINJA terminal/sniffer software. This software has a unique Listener Socket plugin that will allow your PC to receive incoming TCP connections.



#### global.tbh:

```
'DEFINES-------------------------------------------------------------
#define WLN_DEBUG_PRINT 1
#define WLN_WPA 1
'For version of TIDE prior to 5.04.03, this value should be defined
'#define LIB_VER_2_10 'Define the library version here LIB_VER_2_00/LIB_VER_2_01/LIB_VER_2_03/LIB_VER_2_04/LIB_VER_2_10 


#if PLATFORM_ID=EM500 or PLATFORM_ID=EM500W or PLATFORM_ID=EM1202W
	#define WLN_RESET_MODE 1 'there will be no dedicated reset, and all other lines are fixed
#elif PLATFORM_ID=EM1206 or PLATFORM_ID=EM1206W or PLATFORM_ID=DS1101W or PLATFORM_ID=DS1102W
	#define WLN_CLK PL_IO_NUM_14
	#define WLN_CS PL_IO_NUM_15
	#define WLN_DI PL_IO_NUM_12
	#define WLN_DO PL_IO_NUM_13
	#define WLN_RST PL_IO_NUM_11
#else
	'EM1000, NB1010,...
	#define WLN_CLK PL_IO_NUM_53
	#define WLN_CS PL_IO_NUM_49
	#define WLN_DI PL_IO_NUM_52
	#define WLN_DO PL_IO_NUM_50
	#define WLN_RST PL_IO_NUM_51
#endif

'INCLUDES------------------------------------------------------------
include "sock\trunk\sock.tbh" 'this lib is necessary for the WLN lib's operation
include "settings\trunk\settings.tbh" 'this lib is necessary to save pre-shared master key
includepp "settings.xtxt"
include "wln\trunk\wln.tbh"

'DECLARATIONS--------------------------------------------------------
declare function connect_to_ap(byref ap_name as string, security_mode as pl_wln_security_modes, byref key as string, domain as pl_wln_domains) as en_wln_status_codes
declare tcp_sock as byte
```
#### main.tbs:

```
include "global.tbh"

dim tcp_sock as byte

'====================================================================
sub on_sys_init()
	wln.ip="192.168.1.50"						'<--- set suitable IP here
	
	stg_start()
	if connect_to_ap("TIBB1",WLN_SECURITY_MODE_WPA2,"12345678",PL_WLN_DOMAIN_FCC)<>WLN_STATUS_OK then
		sys.halt
	end if

	'configure loopback socket
	tcp_sock=sock_get("TCP")
	sock.num=tcp_sock
	sock.rxbuffrq(1)
	sock.txbuffrq(1)
	sys.buffalloc
	sock.protocol=PL_SOCK_PROTOCOL_TCP
	sock.targetinterface=PL_SOCK_INTERFACE_WLN
	sock.targetip="192.168.1.67"				'<----- set suitable target IP here
	sock.targetport=1000						'<----- set suitable target port here
end sub

'--------------------------------------------------------------------
sub on_sys_timer()
	wln_proc_timer()
	
	'this code will ensure persistent connection to the target IP
	if wln_check_association()=PL_WLN_ASSOCIATED then
		sock.num=tcp_sock
		if sock.statesimple=PL_SSTS_CLOSED then
			sock.connect
		end if
	end if
end sub

'--------------------------------------------------------------------
sub on_sock_data_arrival()
	wln_proc_data()

	'loopback tcp data
	if sock.num=tcp_sock then
		sock.setdata(sock.getdata(sock.txfree))
		sock.send
	end if
end sub

'--------------------------------------------------------------------
sub on_wln_task_complete(completed_task as pl_wln_tasks)
	wln_proc_task_complete(completed_task)
end sub

'--------------------------------------------------------------------
sub on_wln_event(wln_event as pl_wln_events)
	wln_proc_event(wln_event)
end sub
```
#### device.tbs:

```
include "global.tbh"

'====================================================================
function connect_to_ap(byref ap_name as string, security_mode as pl_wln_security_modes, byref key as string, domain as pl_wln_domains) as en_wln_status_codes
	dim pmk as string(32)

	#if WLN_WPA
		if security_mode=WLN_SECURITY_MODE_WPA1 or security_mode=WLN_SECURITY_MODE_WPA2 then 
			if stg_get("APN",0)<>ap_name or stg_get("PW",0)<>key or stg_sg("PMK",0,pmk,EN_STG_GET)<>EN_STG_STATUS_OK then
				'recalculate the key
				pmk=wln_wpa_mkey_get(key,ap_name)
				stg_set("PMK",0,pmk)
				stg_set("APN",0,ap_name)
				stg_set("PW",0,key)
			else
				pmk=stg_get("PMK",0) 'the key stays the same
			end if
		else
			pmk=key
		end if
	#else
		pmk=key
	#endif
	
	#if SRC_LIB_VER < &h020003
		connect_to_ap=wln_start(ap_name,security_mode,pmk,domain)
	#endif

	#if SRC_LIB_VER >= &h020003
		connect_to_ap=wln_start(ap_name,security_mode,pmk,domain,YES,PL_WLN_ASCAN_INFRASTRUCTURE)
	#endif

end function

'--------------------------------------------------------------------
sub callback_wln_ok()
	pat.play("G~",PL_PAT_CANINT)
end sub

'--------------------------------------------------------------------
sub callback_wln_failure(wln_state as en_wln_status_codes)
	pat.play("-",PL_PAT_CANINT)
	sock.num=tcp_sock
	sock.discard
end sub

'--------------------------------------------------------------------
sub callback_wln_pre_buffrq(required_buff_pages as byte)
end sub

'--------------------------------------------------------------------
sub callback_wln_rescan_result(current_rssi as byte, scan_rssi as byte, different_ap as no_yes)
end sub

'--------------------------------------------------------------------
sub callback_wln_mkey_progress_update(progress as byte)
	pat.play("B-**",PL_PAT_CANINT)
end sub

'--------------------------------------------------------------------
sub callback_stg_error(byref stg_name_or_num as string,index as byte,status as en_stg_status_codes)
end sub

'--------------------------------------------------------------------
sub callback_stg_pre_get(byref stg_name_or_num as string,index as byte,byref stg_value as string)
end sub

'--------------------------------------------------------------------
sub callback_stg_post_set(byref stg_name_or_num as string, index as byte,byref stg_value as string)
end sub

sub callback_wln_starting_association()
end sub

sub callback_wln_rescan_for_better_ap()
end sub
```
#### settings.xtxt (the underlying configuration file):

```
>>APN	E	S	1	0	32	A	^	Access point name
>>PW	E	S	1	0	32	A	^	Password
>>PMK	E	S	1	32	32	A	12345678901234567890123456789012	Pre-shared master key

#define STG_DESCRIPTOR_FILE "settings.xtxt"
#define STG_MAX_NUM_SETTINGS 3
#define STG_MAX_SETTING_NAME_LEN 3
#define STG_MAX_SETTING_VALUE_LEN 32
```



Step 4: Roaming Between Access Points
-------------------

This step corresponds to **test_wln_lib_4**.

Finally, here is the coolest example of them all — the code for a mobile device that roams between several access points that all have the same SSID (name). Typically, you use such a setup to create a larger wireless network. The example from the previous topic would work, but not very well if your device was mobile.

Say, you walk around with your GA1000-based gadget in your pocket. At boot, it would execute ***wln_start()***, which would in turn search for the target wireless network, find several access points, and select the one with the strongest signal. The GA1000 would then latch onto this access point and won't let go until the signal gets so bad that the association is lost. The WLN library would then try to find a better access point to work with. Sounds like there is no problem, right?

Wrong! In practice, you will find that at a certain distance from the access point you get into the "grey area of communications" where the link between your gadget and the access point becomes spotty, but not quite bad enough for the association to fail. And, since the association won't quite fail, your device won't try to find a better access point to link to. Meanwhile, you will experience data delivery delays, unstable comms, etc.

Here is the solution. Additional code in ***on_sys_timer*** periodically checks current signal strength (***wln.rssi***). Whenever it falls below a predefined constant RSSI_THRESHOLD, the code will do wln_rescan() in an attempt to "find a better deal". Rescan result is checked in ***callback_wln_rescan_result()***. If a different and better access point is found, ***wln_change()*** will switch your device to using it.

It is important to understand why the program searches for better access points only when the signal drops below RSSI_THRESHOLD. Scanning is a disruptive process that temporarily interferes with data comms. There is no reason to scan if the current signal is perfect! Exactly what value should RSSI_THRESHOLD have depends on your device, antenna, etc. Find it through trial and error.



#### global.tbh:

```
'DEFINES-------------------------------------------------------------

#define WLN_DEBUG_PRINT 1
#define WLN_WPA 1
'For version of TIDE prior to 5.04.03, this value should be defined
'#define LIB_VER_2_10 'Define the library version here LIB_VER_2_00/LIB_VER_2_01/LIB_VER_2_03/LIB_VER_2_04/LIB_VER_2_10 

#if PLATFORM_ID=EM500 or PLATFORM_ID=EM500W or PLATFORM_ID=EM1202W
	#define WLN_RESET_MODE 1 'there will be no dedicated reset, and all other lines are fixed
#elif PLATFORM_ID=EM1206 or PLATFORM_ID=EM1206W or PLATFORM_ID=DS1101W or PLATFORM_ID=DS1102W
	#define WLN_CLK PL_IO_NUM_14
	#define WLN_CS PL_IO_NUM_15
	#define WLN_DI PL_IO_NUM_12
	#define WLN_DO PL_IO_NUM_13
	#define WLN_RST PL_IO_NUM_11
#else
	'EM1000, NB1010,...
	#define WLN_CLK PL_IO_NUM_53
	#define WLN_CS PL_IO_NUM_49
	#define WLN_DI PL_IO_NUM_52
	#define WLN_DO PL_IO_NUM_50
	#define WLN_RST PL_IO_NUM_51
#endif

'INCLUDES------------------------------------------------------------
include "sock\trunk\sock.tbh" 'this lib is necessary for the WLN lib's operation
include "settings\trunk\settings.tbh" 'this lib is necessary to save pre-shared master key
includepp "settings.xtxt"
include "wln\trunk\wln.tbh"

'DECLARATIONS--------------------------------------------------------
declare function connect_to_ap(byref ap_name as string, security_mode as pl_wln_security_modes, byref key as string, domain as pl_wln_domains) as en_wln_status_codes
declare tcp_sock as byte
	#if SRC_LIB_VER < &h020003
		declare function wln_rescan(byref ap_name as string) as en_wln_status_codes
	#endif

	#if SRC_LIB_VER >= &h020003
		declare function wln_rescan() as en_wln_status_codes
	#endif

const RESCAN_PERIOD=30							'<----- change as needed
const RSSI_THRESHOLD=200						'<----- change as needed
const AP_NAME="TIBB1"							'<----- change as needed
const AP_PASSWORD="12345678AB"					'<----- change as needed
const AP_SECURITY=WLN_SECURITY_MODE_WPA2		'<----- change as needed
'use WLN_SECURITY_MODE_DISABLED, WLN_SECURITY_MODE_WEP64, WLN_SECURITY_MODE_WEP128, WLN_SECURITY_MODE_WPA1, or WLN_SECURITY_MODE_WPA2

```
#### main.tbs:

```
include "global.tbh"

'--------------------------------------------------------------------
dim tcp_sock as byte
dim rescan_tmr as byte

'====================================================================
sub on_sys_init()
	wln.ip="192.168.1.50"						'<----- set suitable IP here
	
	stg_start()
	
	if connect_to_ap(AP_NAME,AP_SECURITY,AP_PASSWORD,PL_WLN_DOMAIN_FCC)<>WLN_STATUS_OK then
		sys.halt
	end if

	'configure loopback socket
	tcp_sock=sock_get("TCP")
	sock.num=tcp_sock
	sock.rxbuffrq(1)
	sock.txbuffrq(1)
	sys.buffalloc
	sock.protocol=PL_SOCK_PROTOCOL_TCP
	sock.targetinterface=PL_SOCK_INTERFACE_WLN
	sock.targetip="192.168.1.67"				'<----- set suitable target IP here
	sock.targetport=1000						'<----- set suitable target port here
end sub

'--------------------------------------------------------------------
sub on_sys_timer()
	wln_proc_timer()
	
	if wln_check_association()=PL_WLN_ASSOCIATED then
		'this code ensures persistent connection to the target IP
		sock.num=tcp_sock
		if sock.statesimple=PL_SSTS_CLOSED then
			sock.connect
		end if

		'this code handles access point roaming
		if rescan_tmr>0 then
			rescan_tmr=rescan_tmr-1
			if rescan_tmr=0 then
				rescan_tmr=RESCAN_PERIOD
				if wln.rssi<=RSSI_THRESHOLD then

					#if SRC_LIB_VER < &h020003
						wln_rescan(AP_NAME) 
					#endif

					#if SRC_LIB_VER >= &h020003
						wln_rescan()
					#endif
				
				end if
			end if
		end if
	else
		rescan_tmr=RESCAN_PERIOD
	end if
end sub

'--------------------------------------------------------------------
sub on_sock_data_arrival()
	wln_proc_data()

	'loopback tcp data
	if sock.num=tcp_sock then
		sock.setdata(sock.getdata(sock.txfree))
		sock.send
	end if
end sub

'--------------------------------------------------------------------
sub on_wln_task_complete(completed_task as pl_wln_tasks)
	wln_proc_task_complete(completed_task)
end sub

'--------------------------------------------------------------------
sub on_wln_event(wln_event as pl_wln_events)
	wln_proc_event(wln_event)
end sub
```
#### device.tbs:

```
include "global.tbh"

'====================================================================
function connect_to_ap(byref ap_name as string, security_mode as pl_wln_security_modes, byref key as string, domain as pl_wln_domains) as en_wln_status_codes
	dim pmk as string(32)

	#if WLN_WPA
		if security_mode=WLN_SECURITY_MODE_WPA1 or security_mode=WLN_SECURITY_MODE_WPA2 then 
			if stg_get("APN",0)<>ap_name or stg_get("PW",0)<>key or stg_sg("PMK",0,pmk,EN_STG_GET)<>EN_STG_STATUS_OK then
				'recalculate the key
				pmk=wln_wpa_mkey_get(key,ap_name)
				stg_set("PMK",0,pmk)
				stg_set("APN",0,ap_name)
				stg_set("PW",0,key)
			else
				pmk=stg_get("PMK",0) 'the key stays the same
			end if
		else
			pmk=key
		end if
	#else
		pmk=key
	#endif

	#if SRC_LIB_VER < &h020003
		connect_to_ap=wln_start(ap_name,security_mode,pmk,domain)
	#endif

	#if SRC_LIB_VER >= &h020003
		connect_to_ap=wln_start(ap_name,security_mode,pmk,domain,YES,PL_WLN_ASCAN_INFRASTRUCTURE)
	#endif


end function

'--------------------------------------------------------------------
sub callback_wln_ok()
	pat.play("G~",PL_PAT_CANINT)
end sub

'--------------------------------------------------------------------
sub callback_wln_failure(wln_state as en_wln_status_codes)
	pat.play("-",PL_PAT_CANINT)
	sock.num=tcp_sock
	sock.discard
end sub

'--------------------------------------------------------------------
sub callback_wln_pre_buffrq(required_buff_pages as byte)
end sub

'--------------------------------------------------------------------
sub callback_wln_rescan_result(current_rssi as byte, scan_rssi as byte, different_ap as no_yes)
	dim s as string(32)

	if different_ap=NO or scan_rssi<current_rssi or scan_rssi-current_rssi<5 then
		exit sub
	end if
	
	select case AP_SECURITY
	case WLN_SECURITY_MODE_DISABLED:
		s=""
	case WLN_SECURITY_MODE_WEP64,WLN_SECURITY_MODE_WEP128:
		s=AP_PASSWORD
	case else:
		s=stg_get("PMK",0)
	end select

	wln_change(AP_NAME,AP_SECURITY,s)
end sub

'--------------------------------------------------------------------
sub callback_wln_mkey_progress_update(progress as byte)
	pat.play("B-**",PL_PAT_CANINT)
end sub

'--------------------------------------------------------------------
sub callback_stg_error(byref stg_name_or_num as string,index as byte,status as en_stg_status_codes)
end sub

'--------------------------------------------------------------------
sub callback_stg_pre_get(byref stg_name_or_num as string,index as byte,byref stg_value as string)
end sub

'--------------------------------------------------------------------
sub callback_stg_post_set(byref stg_name_or_num as string, index as byte,byref stg_value as string)
end sub

sub callback_wln_starting_association()
end sub

sub callback_wln_rescan_for_better_ap()
end sub
```
#### settings.xtxt (the underlying configuration file):

```
>>APN	E	S	1	0	32	A	^	Access point name
>>PW	E	S	1	0	32	A	^	Password
>>PMK	E	S	1	32	32	A	12345678901234567890123456789012	Pre-shared master key

#define STG_DESCRIPTOR_FILE "settings.xtxt"
#define STG_MAX_NUM_SETTINGS 3
#define STG_MAX_SETTING_NAME_LEN 3
#define STG_MAX_SETTING_VALUE_LEN 32

```



For more detail about this project, please visit <a href="http://tibbo.com/programmable/applications/examples/lib_wln.html" target="_blank">Project Description Page</a>