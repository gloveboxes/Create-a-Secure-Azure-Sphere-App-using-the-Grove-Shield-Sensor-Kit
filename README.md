# Create a Secure Azure Sphere App using the Grove Shield Sensor Kit

|Author|[Dave Glover](https://developer.microsoft.com/en-us/advocates/dave-glover?WT.mc_id=github-blog-dglover), Microsoft Cloud Developer Advocate |
|:----|:---|
|Target Platform | Seeed Studio Azure Sphere MT3620 |
|Developer Platform | Windows 10 or Ubuntu 18.04 |
|Azure SDK | Azure Sphere SDK 19.11 or better |
|Developer Tools| [Visual Studio (The free Community Edition or better)](https://visualstudio.microsoft.com/vs/?WT.mc_id=github-blog-dglover) or [Visual Studio Code (Free OSS)](https://code.visualstudio.com?WT.mc_id=github-blog-dglover)|
|Hardware | [Seeed Studio Grove Shield](https://www.seeedstudio.com/MT3620-Grove-Shield.html), and the [Grove Temperature and Humidity Sensor (SHT31)](https://www.seeedstudio.com/Grove-Temperature-Humidity-Sensor-SHT31.html) |
|Language| C|
|Date|As of December, 2019|

## What is Azure Sphere

Azure Sphere is a secured, high-level application platform with built-in communication and security features for internet-connected devices.

It comprises a secured, connected, crossover microcontroller unit (MCU), a custom high-level Linux-based operating system (OS), and a cloud-based security service that provides continuous, renewable security.

## Hardware Required

This tutorial requires the [Seeed Studio Azure Sphere](https://www.seeedstudio.com/Azure-Sphere-MT3620-Development-Kit-US-Version-p-3052.html), the [Seeed Studio Grove Shield](https://www.seeedstudio.com/MT3620-Grove-Shield.html), and the [Grove Temperature and Humidity Sensor (SHT31)](https://www.seeedstudio.com/Grove-Temperature-Humidity-Sensor-SHT31.html). These parts are available from many online stores including [Seeed Studio](www.seeedstudio.com).

Be sure to plug the Grove Temperature Sensor into one of the I2C connectors on the Grove Shield.

![Azure Sphere with shield](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/azure-sphere-shield.png)

## Set up your Development Environment

This tutorial assumes Windows 10 and [Visual Studio (The free Community Edition or better)](https://visualstudio.microsoft.com/vs/?WT.mc_id=github-blog-dglover). For now, Azure Sphere templates are only available for Visual Studio. However, you can clone and open this solution on Windows and Ubuntu 18.04 with [Visual Studio Code](https://code.visualstudio.com/?WT.mc_id=github-blog-dglover).

Follow the Azure Sphere [Overview of set up procedures](https://docs.microsoft.com/en-au/azure-sphere/install/overview?WT.mc_id=github-blog-dglover) guide.

## Azure Sphere SDK

This tutorial assumes you are using the [Azure Sphere SDK 19.11](https://docs.microsoft.com/en-us/azure-sphere/resources/release-notes-1911?WT.mc_id=github-blog-dglover) or better which uses the CMake Build System.

This tutorial uses a fork of the Seeed Studio [Grove Shield Library](https://github.com/Seeed-Studio/MT3620_Grove_Shield) that has been updated to support Azure Sphere SDK 19.11.

## Clone the MT3620 Grove Shield Library

1. Create a folder where you plan to build your Azure Sphere applications.
2. Clone the MT3620 Grove Shield Library.

	Open a command window and change to the directory where you plan to build your Azure Sphere applications.

	```bash
	git clone https://github.com/gloveboxes/MT3620_Grove_Shield.git
	```

## Create a new Visual Studio Azure Sphere Project

Start Visual Studio and create a new project in the same directory you cloned the MT3620 Grove Shield Library.

It is important to create the Visual Studio Project in the same folder you cloned the MT3620 Grove Shield as there are relative links to this library in the application you will create.

```text
azure-sphere
	|- MT3620_Grove_Shield
	|- YourAzureSphereApplication
```

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-create-new-project.png)

### Select Azure Sphere Project Template

Type **sphere** in the search box and select the Azure Sphere Blink template.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-select-azure-sphere-blink.png)

### Configure new Azure Sphere Project

Name the project and set the save location.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-configure-new-project.png)

### Open the CMakeLists.txt file

CMakelists.txt defines the build process, the files and locations of libraries and more.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-open-cmakelists.png)

### Add a Reference to MT3620_Grove_Shield_Library

Two items need to be added:

1. The source location on the MT3620 Grove Shield library. Note, this is the relative path to the Grove Shield library.
2. Add MT3620_Grove_Shield_Library to the target_link_libraries definition. This is equivalent to adding a reference.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-configure-cmakelists.png)

## Set the Application Capabilities

The application manifest defines what resources will be available to the application. Define the minimum set of privileges required by the application. This is core to Azure Sphere security and is also known as the [Principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege).

1. Review the [Grove Shield Sensor Capabilities Quick Reference](#grove-shield-sensor-capabilities-quick-reference) to understand what capabilities are required for each sensor in the library.
2. Open **app_manifest.json**
3. Add Uart **ISU0**
   - Note, access to the I2C SHT31 temperature/humidity sensor via the Grove Shield was built before Azure Sphere supported I2C. Hence calls to the sensor are proxied via the Uart.
4. Note, GPIO 9 is used to control an onboard LED.

```json
{
  "SchemaVersion": 1,
  "Name": "AzureSphereBlink1",
  "ComponentId": "a3ca0929-5f46-42b0-91ba-d5de1222da86",
  "EntryPoint": "/bin/app",
  "CmdArgs": [],
  "Capabilities": {
    "Gpio": [ 9 ],
    "Uart": [ "ISU0" ],
    "AllowedApplicationConnections": []
  },
  "ApplicationType": "Default"
}
```

### Update the Code

The following code includes the Grove Sensor headers, opens the Grove Sensor, and the loops reading the temperature and humidity and writes this information to the debugger logger.

Replace all the existing code in the **main.c** file with the following:

```c
#include <signal.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <errno.h>

#include <applibs/log.h>
#include <applibs/gpio.h>

// Grove Temperature and Humidity Sensor
#include "../MT3620_Grove_Shield/MT3620_Grove_Shield_Library/Grove.h"
#include "../MT3620_Grove_Shield/MT3620_Grove_Shield_Library/Sensors/GroveTempHumiSHT31.h"

static volatile sig_atomic_t terminationRequested = false;

static void TerminationHandler(int signalNumber)
{
	// Don't use Log_Debug here, as it is not guaranteed to be async signal safe
	terminationRequested = true;
}

int main(int argc, char* argv[])
{
	Log_Debug("Application starting\n");

	// Register a SIGTERM handler for termination requests
	struct sigaction action;
	memset(&action, 0, sizeof(struct sigaction));
	action.sa_handler = TerminationHandler;
	sigaction(SIGTERM, &action, NULL);

	// Change this GPIO number and the number in app_manifest.json if required by your hardware.
	int fd = GPIO_OpenAsOutput(9, GPIO_OutputMode_PushPull, GPIO_Value_High);
	if (fd < 0) {
		Log_Debug(
			"Error opening GPIO: %s (%d). Check that app_manifest.json includes the GPIO used.\n",
			strerror(errno), errno);
		return -1;
	}

	// Initialize Grove Shield and Grove Temperature and Humidity Sensor
	int i2cFd;
	GroveShield_Initialize(&i2cFd, 115200);
	void* sht31 = GroveTempHumiSHT31_Open(i2cFd);

	const struct timespec sleepTime = { 1, 0 };
	while (!terminationRequested) {

		GroveTempHumiSHT31_Read(sht31);
		float temp = GroveTempHumiSHT31_GetTemperature(sht31);
		float humi = GroveTempHumiSHT31_GetHumidity(sht31);
		Log_Debug("Temperature: %.1fC\n", temp);
		Log_Debug("Humidity: %.1f\%c\n", humi, 0x25);

		GPIO_SetValue(fd, GPIO_Value_Low);
		nanosleep(&sleepTime, NULL);

		GPIO_SetValue(fd, GPIO_Value_High);
		nanosleep(&sleepTime, NULL);
	}
}
```

## Deploy the Application to the Azure Sphere

1. Connect the Azure Sphere to your computer via USB
2. Ensure you have [claimed](https://docs.microsoft.com/en-au/azure-sphere/install/claim-device?WT.mc_id=github-blog-dglover), [connected](https://docs.microsoft.com/en-au/azure-sphere/install/configure-wifi?WT.mc_id=github-blog-dglover), and [developer enabled](https://docs.microsoft.com/en-au/azure-sphere/install/qs-blink-application?WT.mc_id=github-blog-dglover) your Azure Sphere.

3. Select **GDB Debugger (HLCore)** from the **Select Startup** dropdown.
	![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-start-application.png)
4. From Visual Studio, press **F5** to build, deploy, start, and attached the remote debugger to the Azure Sphere.

### View the Debugger Output

Open the _Output_ window to view the output from **Log_Debug** statements in _main.c_.

You can do this by using the Visual Studio **Ctrl+Alt+O** keyboard shortcut or click the **Output** tab found along the bottom/right of Visual Studio.

![Visual Studio View Output](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-view-output.png)

### Set a Debug Breakpoint

Set a debugger breakpoint by clicking in the margin to the left of the line of code you want the debugger to stop at.

In the **main.c** file set a breakpoint in the margin of the line that reads the Grove temperature and pressure sensor **GroveTempHumiSHT31_Read(sht31);**.

 ![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-set-breakpoint.png)

### Stop the Debugger

**Stop** the debugger by using the Visual Studio **Shift+F5** keyboard shortcut or click the **Stop Debugging** icon.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/vs-stop-debugger.png)

## Azure Sphere Application Cloud Deployment

Now you have learnt how to "Side Load" an application onto Azure Sphere it is time to learn about the [Deployment Basics]() to _Cloud Deploy_ an application.

## Finished 完了 fertig finito ख़त्म होना terminado

Congratulations, you created a secure Internet of Things Azure Sphere application.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/finished.jpg)

## Appendix

### Grove Shield Sensor Capabilities Quick Reference

| Sensors  | Socket | Capabilities |
| :------------- | :------------- | :----------- |
| Grove Light Sensor  | Analog | "Gpio": [ 57, 58 ], "Uart": [ "ISU0"] |
| Grove Rotary Sensor | Analog | "Gpio": [ 57, 58 ], "Uart": [ "ISU0"] |
| Grove 4 Digit Display | GPIO0 or GPIO4 | "Gpio": [ 0, 1 ] or "Gpio": [ 4, 5 ] |
| Grove LED Button | GPIO0 or GPIO4 |  "Gpio": [ 0, 1 ] or "Gpio": [ 4, 5 ] |
| Grove Oled Display 96x96 | I2C | "Uart": [ "ISU0"]  |
| Grove Temperature Humidity SHT31 | I2C | "Uart": [ "ISU0"] |
| Grove UART3 | UART3 | "Uart": [ "ISU3"] |
| LED 1 | Red <br/> Green <br/> Blue | "Gpio": [ 8 ] <br/> "Gpio": [ 9 ] <br/> "Gpio": [ 10 ] |
| LED 2 | Red <br/> Green <br/> Blue | "Gpio": [ 15 ] <br/> "Gpio": [ 16 ] <br/> "Gpio": [ 17 ] |
| LED 3 | Red <br/> Green <br/> Blue | "Gpio": [ 18 ] <br/> "Gpio": [ 19 ] <br/> "Gpio": [ 20 ] |
| LED 4 | Red <br/> Green <br/> Blue | "Gpio": [ 21 ] <br/> "Gpio": [ 22 ] <br/> "Gpio": [ 23 ] |

For more pin definitions see the __mt3620_rdb.h__ in the MT3620_Grove_Shield/MT3620_Grove_Shield_Library folder.

### Azure Sphere Grove Kit

| Azure Sphere   |  Image  |
| ---- | ---- |
| [Azure Sphere MT3620 Development Kit](https://www.seeedstudio.com/Azure-Sphere-MT3620-Development-Kit-US-Version-p-3052.html)|
| [Azure Sphere MT3620 Development Kit Shield](https://www.seeedstudio.com/Grove-Starter-Kit-for-Azure-Sphere-MT3620-Development-Kit.html). <br/> Note, you can also purchase the parts separately. | ![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/seeed-studio-grove-shield-and-sensors.jpg) |

### Azure Sphere MT3620 Developer Board Pinmap

The full Azure Sphere MT3620 Board Pinmap can be found on the [Azure Sphere MT3620 Development Kit](https://www.seeedstudio.com/Azure-Sphere-MT3620-Development-Kit-US-Version-p-3052.html) page.

![](https://raw.githubusercontent.com/gloveboxes/Create-a-Secure-Azure-Sphere-App-using-the-Grove-Shield-Sensor-Kit/master/resources/mt3620-dev-board-pinmap.png)
