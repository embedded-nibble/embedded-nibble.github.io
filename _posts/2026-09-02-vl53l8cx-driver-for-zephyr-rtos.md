---
layout: post
title: Writing an I2C Zephyr Driver Module for the VL53L8CX Sensor
date: 2026-09-02 10:00:00 -0600
categories: [Firmware, Drivers & HAL]
tags: [zephyr, vl53l8cx, tof, lidar, sensor, driver, RTOS, nrf52840, i2c, devicetree]
description: Porting ST's Ultra Lite Driver for the VL53L8CX time-of-flight sensor into a reusable Zephyr module for real-time distance-matrix capture.
toc: true
---

> **Project:** [VL53L8CX Zephyr Driver](https://github.com/krloSc/vl53l8cx_driver)

I wanted to use a VL53L8CX breakout board with an nRF52840 development board and read its distance matrix in real time over I2C. The challenge was not only getting data from the sensor, but making that data easy to use from a Zephyr application.

Rather than creating a one-off test project, I built a reusable Zephyr module. It can be added through a `west.yml` manifest, configured in DeviceTree and Kconfig, and used through the Zephyr sensor API.

<!-- Image placeholder: nRF52840 development board wired to the VL53L8CX breakout board. -->

## The starting point: an API without a Zephyr home

The VL53L8CX is a multizone time-of-flight sensor. Instead of returning one distance, it can return a 4x4 or 8x8 grid of measurements: 16 or 64 zones per frame. That makes it useful for lightweight spatial awareness, presence detection, and simple object tracking without a full camera pipeline.

Zephyr already had drivers for earlier ST ranging sensors such as the VL53L0X and VL53L1X, but not for the VL53L8CX. ST did provide a strong foundation in its Ultra Lite Driver (ULD) API, including the firmware-loading and ranging logic. The gap was the platform boundary: ST's API expected a small set of board-specific functions, while Zephyr expects a driver to participate in its DeviceTree, Kconfig, build, module, and sensor subsystems.

The initial hypothesis was that adapting `platform.h` and its companion implementation would be sufficient. It was true for communicating with the device; it was not enough to make the work a Zephyr driver that others could actually consume.

## Defining the integration contract first

I started from the outer boundary rather than from the sensor API. In Zephyr, a DeviceTree binding documents how hardware appears to the build system. The binding declares the I2C-compatible device plus optional pins for low-power control and data-ready interrupts:

```yaml
description: VL53L8CX time-of-flight distance sensor

compatible: "st,vl53l8cx"

include: [sensor-device.yaml, i2c-device.yaml]

properties:
  lpn-gpios:
    type: phandle-array
    description: Driving LPN low puts the VL53L8CX into low power mode.

  int-gpios:
    type: phandle-array
    description: An interrupt is raised when a distance measurement is ready.
```

With the binding in place, an application can describe the physical connection in its board overlay:

```c
&i2c0 {
    status = "okay";
    clock-frequency = <I2C_BITRATE_FAST>;

    vl53l8cx@29 {
        compatible = "st,vl53l8cx";
        reg = <0x29>;
        int-gpios = <&gpio0 11 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
        lpn-gpios = <&gpio0 12 GPIO_ACTIVE_HIGH>;
    };
};
```

This small piece is more than configuration. It separates board wiring from driver logic, lets Zephyr validate the hardware description, and gives the driver a portable way to obtain its I2C bus, address, and GPIOs.

## Bridging ST's platform API to Zephyr

ST's ULD intentionally delegates transport and timing details to a platform layer. The port stores Zephyr's I2C device and the DeviceTree-selected I2C address in ST's `VL53L8CX_Platform` structure, then implements ST's required read, write, reset, and delay functions with Zephyr primitives.

The platform layer handles the full range of I2C traffic required by the ULD. Small register writes use Zephyr's `i2c_write()`, while reads and larger transfers use `i2c_transfer()` with the sensor's 16-bit register address. The same multi-byte functions also split large payloads into chunks, which is important during sensor initialization and firmware loading. Finally, the ULD delay function maps directly to Zephyr's `k_msleep()`.

```c
uint8_t VL53L8CX_WrByte(VL53L8CX_Platform *platform,
                        uint16_t register_address, uint8_t value)
{
    uint8_t tx[3] = {
        (uint8_t)(register_address >> 8),
        (uint8_t)register_address,
        value,
    };

    return i2c_write(platform->i2c_dev, tx, sizeof(tx), platform->address);
}

#define VL53L8CX_I2C_CHUNK_SIZE 1024U

while (offset < size) {
    uint16_t reg = (uint16_t)(RegisterAddress + offset);
    uint32_t chunk = size - offset;
    uint8_t reg_buffer[2] = {
        (uint8_t)(reg >> 8),
        (uint8_t)reg,
    };

    if (chunk > VL53L8CX_I2C_CHUNK_SIZE) {
        chunk = VL53L8CX_I2C_CHUNK_SIZE;
    }

    struct i2c_msg msgs[] = {
        { .buf = reg_buffer, .len = sizeof(reg_buffer), .flags = I2C_MSG_WRITE },
        { .buf = &p_values[offset], .len = chunk,
          .flags = I2C_MSG_RESTART | I2C_MSG_READ | I2C_MSG_STOP },
    };

    if (i2c_transfer(i2c_dev, msgs, ARRAY_SIZE(msgs), address) != 0) {
        return 1;
    }
    offset += chunk;
}

uint8_t VL53L8CX_WaitMs(VL53L8CX_Platform *platform, uint32_t time_ms)
{
    k_msleep(time_ms);
    return 0;
}
```

`VL53L8CX_RdByte()` delegates to the multi-byte read function with a one-byte length, so the same addressing path works for both small reads and larger blocks. The `lpn-gpios` line is also used for a hardware reset. This was the first decisive integration point: once the ULD could communicate through Zephyr's I2C API, ST's initialization and ranging functions became available to the driver.

## Turning the port into a Zephyr driver

Getting bytes across I2C was only the transport layer. A proper Zephyr driver needs configuration from DeviceTree, runtime state, initialization, logging, error handling, and a public API.

The driver configuration keeps the hardware-facing details as Zephyr specifications:

```c
struct st_vl53l8cx_config {
    struct i2c_dt_spec i2c;
    struct gpio_dt_spec lpn;
    struct gpio_dt_spec int_gpio;
};
```

During initialization, the driver verifies the I2C bus, maps the DeviceTree values into ST's platform structure, checks that the sensor is alive, loads the VL53L8CX firmware through the ULD, applies the Kconfig-selected ranging settings, and starts ranging.

```c
if (!i2c_is_ready_dt(&config->i2c)) {
    return -ENODEV;
}

data->st_dev.platform.i2c_dev = config->i2c.bus;
data->st_dev.platform.address = config->i2c.addr;

status = vl53l8cx_is_alive(&data->st_dev, &is_alive);
if (status != 0 || !is_alive) {
    return -ENODEV;
}

status = vl53l8cx_init(&data->st_dev);
if (status != 0) {
    return -EIO;
}
```

For the standard Zephyr sensor API, `SENSOR_CHAN_DISTANCE` reports a robust center-zone distance. The driver averages the four center cells and discards invalid values. This makes the sensor straightforward to use in applications that need one forward-looking distance.

That API alone would hide the VL53L8CX's most valuable capability, though: its spatial matrix. I therefore added a small driver-specific accessor that copies the latest frame into an application buffer:

```c
int vl53l8cx_get_distance_matrix(const struct device *dev, uint16_t *matrix)
{
    struct st_vl53l8cx_data *data = dev->data;

    memcpy(matrix, data->distance_matrix_mm,
           sizeof(data->distance_matrix_mm));
    return 0;
}
```

The result is a balanced interface: standard Zephyr code can ask for `SENSOR_CHAN_DISTANCE`, while applications that need the 4x4 or 8x8 frame can opt into the matrix API.

## Packaging it for other projects

Accessibility was a requirement, not a final cleanup step. The source tree includes the ST ULD sources, the platform adaptation, the Zephyr driver, Kconfig definitions, DeviceTree bindings, and CMake wiring. The `zephyr/module.yml` file marks the repository as a Zephyr module:

```yaml
build:
  kconfig: Kconfig
  cmake: .
  settings:
    dts_root: .
```

That allows an application to bring the driver in through its west manifest:

```yaml
manifest:
  projects:
    - name: vl53l8cx_driver
      url: https://github.com/krloSc/vl53l8cx_driver.git
      revision: main
      path: modules/lib/vl53l8cx_driver
```

The module approach keeps the driver independent of a specific application and avoids modifying Zephyr or nRF Connect SDK source trees directly.

## Proving the complete path

The final proof was deliberately simple: fetch a sample, retrieve the matrix, print the values through serial, and plot the incoming frames with a small Python program. It exercised the whole path from DeviceTree through I2C transport, ST firmware, Zephyr driver state, application code, UART output, and host visualization.

```c
uint16_t distance_matrix_mm[64] = {0};

ret = sensor_sample_fetch(sensor);
if (ret == 0) {
    ret = vl53l8cx_get_distance_matrix(sensor, distance_matrix_mm);
}
```

The application configuration is equally important. It must enable I2C, the sensor subsystem, and the driver. I also increased the main stack to account for initialization work and the application workload:

```conf
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_I2C=y
CONFIG_SENSOR=y
CONFIG_ST_VL53L8CX=y
CONFIG_ST_VL53L8CX_RESOLUTION_8X8=y
CONFIG_ST_VL53L8CX_MODE_CONTINUOUS=y
CONFIG_ST_VL53L8CX_RANGING_FREQUENCY=15
```

![VL53L8CX distance capture](assets/posts/vl53l8cx-driver-for-zephyr-rtos/capture.gif)

*Figure: Real-time distance matrix output from the VL53L8CX as printed by a Python visualization script, with a hand passing in front of the sensor during the capture.*

## Outcome and next steps

The result is an open, module-based Zephyr driver for the VL53L8CX that provides real-time I2C ranging on an nRF52840-class target. It retains the value of ST's ULD while presenting the sensor through Zephyr's configuration and driver conventions.

The current implementation is intentionally focused on I2C and polling. The next engineering steps are clear:

- Add SPI transport to reduce transfer latency and speed up firmware loading.
- Use the sensor's data-ready interrupt instead of relying only on polling.
- Add application-appropriate filtering and confidence handling for noisy scenes.
- Expand the public API as more of the VL53L8CX feature set becomes useful in Zephyr applications.

The complete source, including the binding, platform layer, driver, is available in the [project repository](https://github.com/krloSc/vl53l8cx_driver).