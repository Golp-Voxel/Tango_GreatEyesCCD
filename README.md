# GreatEyes CCD - Tango Device Server

This repository contains the driver for controlling a GreatEyes CCD with the Tango Control. After cloning this repository with the following command

```
git clone https://github.com/Golp-Voxel/Tango_GreatEyesCCD.git
```

It is necessary to create the `tango-env` using the following command:

```
python -m venv tango-env
```

After activating it you can install all the models to run this tool by using the command:

```
pip install -r Requirements.txt
```

To complete the installation, it is necessary to copy the `GreatEyes_D.bat` template and change the paths to the installation folder. And the command to run the `...\tango-env\Scripts\activate` script. 

Then copy the `setting.ini` template and fill in the path to the dlls for the GreatEyes CCD.

```ini
[DEFAULT]
DLL =  "C://path//to//folder//with//greateyes.dll"
```

Remember to change the `\` to `//`.

On start-up (`init_device`) the server loads the `greateyes.dll`, connects to the camera (USB by default, Ethernet is also supported by the SDK wrapper in `Code/greateyes.py`) and switches on the cooling system. If no camera is found, the device is set to the `OFF` state.

The acquisition parameters (exposure time, readout speed, binning, crop, shutter timings, trigger configuration, target cooling temperature, ...) are defined at the top of `GreatEyes_D.py` in the *parameter setup* section.

## Attributes

- [Image_foto](#image_foto)
- [ExposureTime](#exposuretime)

### Image_foto

Read-only image attribute (2D array of int, up to 2100 x 2100). Reading this attribute triggers the acquisition of a full frame and returns the image.

```python
image = GreatEyes_CCD.Image_foto
```

### ExposureTime

Read/write attribute with the exposure time of the camera in milliseconds (`DevULong64`). The new value is used on the next acquisition.

```python
GreatEyes_CCD.ExposureTime = 1500   # ms
print(GreatEyes_CCD.ExposureTime)
```

## Available commands

- [get_foto_JSON](#get_foto_json)
- [getTemperature](#gettemperature)
- [setTemperature](#settemperature)
- [OnCooling](#oncooling)
- [OffCooling](#offcooling)

### get_foto_JSON

Acquires a full frame and returns it as a JSON string where the image is stored in the key `"Image"`.

```python
get_foto_JSON()
```

To recover the image as a numpy array:

```python
nd_image_array = json.loads(GreatEyes_CCD.get_foto_JSON())
array_p = np.array(nd_image_array["Image"])
plt.imshow(array_p)
```

### getTemperature

Reads the sensor and backside temperatures of the camera.

```python
getTemperature()
```

It returns a JSON string with the following format (temperatures in degree Celsius):

```
{
    "sensor"   : <sensor temperature>,
    "backside" : <backside temperature>
}
```

### setTemperature

Sets the target temperature (in degree Celsius) for the cooling control. The value must be inside the range supported by the cooling hardware, otherwise it is rejected.

```python
setTemperature(temperature)
```

Where `temperature` is an integer. It returns a JSON string such as:

```
{
    "result"  : true/false,
    "details" : <message>
}
```

### OnCooling

Initializes the temperature control, sets the target temperature, and reads back the sensor and backside temperatures.

```python
OnCooling()
```

It returns `{"result": true}` on success or `{"result": false}` if the setup failed.

### OffCooling

Switches off the temperature control.

```python
OffCooling()
```

It returns `{"result": true}` on success or `{"result": false}` if it failed.

## Example of Tango Client code

```python
import tango
import json
import numpy as np
import matplotlib.pyplot as plt

GreatEyes_CCD = tango.DeviceProxy(<GreatEyes_Tango_location_on_the_database>)
print(GreatEyes_CCD.state())
# Acquisitions can take a long time, increase the timeout if needed
GreatEyes_CCD.set_timeout_millis(60000)

# Cooling control
GreatEyes_CCD.OnCooling()
print(GreatEyes_CCD.setTemperature(-10))
print(json.loads(GreatEyes_CCD.getTemperature()))

# Set the exposure time to 1500 ms
GreatEyes_CCD.ExposureTime = 1500

# Take a photo
J = GreatEyes_CCD.get_foto_JSON()
nd_image_array = json.loads(J)

array_p = np.array(nd_image_array["Image"])
plt.imshow(array_p)

# Switch off the cooling at the end of the session
GreatEyes_CCD.OffCooling()
```

# References

- [greateyes GmbH cameras](https://www.greateyes.de/)
- The manufacturer SDK documentation can be found in the `Manufacturer_Manual` folder.
