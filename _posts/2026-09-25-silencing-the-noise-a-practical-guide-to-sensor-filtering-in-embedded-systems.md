---
layout: post
title: 'Silencing the Noise: A Practical Guide to Sensor Filtering in Embedded Systems'
date: 2026-09-25 19:23 -0400
math: true
categories: [Firmware, DSP]
tags: [embedded, firmware, sensors, filtering]
description: "Learn how to implement a two-stage filtering approach using Median and Exponential Moving Average (EMA) filters to clean up noisy sensor data in embedded systems."
toc: true
---

![Sensor Filtering in Embedded Systems](assets/posts/silencing-the-noise-a-practical-guide-to-sensor-filtering-in-embedded-systems/front.jpg)

If you have ever worked with physical sensors, you know one universal truth: the real world is incredibly noisy. Whether it's electrical interference, ambient environmental changes, or just the physics of the sensor itself, raw data is rarely clean enough to use directly in production firmware.

In embedded systems, filtering is the art of extracting the "true" signal from the noise. The landscape of filters ranges from the incredibly simple (like a basic Moving Average) to the computationally heavy (like Kalman or Particle filters). But you don't always need complex math to get great results. Often, a combination of lightweight, well-chosen algorithms is exactly what a resource-constrained microcontroller needs.

In this post, we'll explore how to tame a noisy sensor using a two-stage approach: a **Median Filter** to reject outliers, followed by an **Exponential Moving Average (EMA) Filter** to smooth the response.

## The Hardware: Taming the VL53L8CX

To demonstrate this, we are going to use the **VL53L8CX**, an excellent multizone Time-of-Flight (ToF) sensor.

---

We recently wrote a custom driver for this sensor. Because of the nature of ToF technology, readings can occasionally be polluted by ambient light interference or multi-path reflections, resulting in sudden, sharp outliers. Furthermore, even when looking at a static target, the reading jitters slightly.

To make this data usable, we need to achieve two things:

1. **Omit outliers:** Discard sudden spikes that don't represent reality.
2. **Smooth the response:** Reduce the jitter to get a stable, readable measurement.

To achieve this, we chose to stack an EMA filter on top of a Median filter.

## The Implementation: Median + EMA in C

Let's look at how we implemented these functions in C, keeping embedded performance in mind.

### 1. The Median Filter (Outlier Rejection)

A median filter is fantastic at ignoring sudden, extreme spikes. By sorting a buffer of recent readings and picking the middle value, any brief outlier is automatically pushed to the edges of the sorted array and ignored.

```c
void median_filter(distance_buffer_t *buffer, int16_t *filtered_distance) {
    if (buffer == NULL || filtered_distance == NULL) {
        return; // Error: buffer or filtered_distance pointer is NULL
    }

    int16_t values[buffer_size];
    for (int i = 0; i < buffer_size; i++) {
        values[i] = buffer->distance[i];
    }
    
    // Sort the values to find the median (Bubble Sort)
    for (int i = 0; i < buffer_size - 1; i++) {
        for (int j = i + 1; j < buffer_size; j++) {
            if (values[i] > values[j]) {
                int16_t temp = values[i];
                values[i] = values[j];
                values[j] = temp;
            }
        }
    }
    
    // The median is the middle value in the sorted array
    *filtered_distance = values[buffer_size / 2];
}

```

> **Note for Embedded Devs:** Mathematically, the median of an even-sized array (like 4 or 8) is the average of the two middle elements. However, doing that requires an extra addition and division. In our firmware, just grabbing the upper-middle element (`buffer_size / 2`) provides virtually identical outlier rejection while saving precious CPU cycles!

### 2. The EMA Filter (Smoothing)

Once the outliers are stripped away by the median filter, we pass the data to an Exponential Moving Average filter.

Standard EMA formulas require floating-point math or heavy division, which can be expensive on Cortex-M microcontrollers without an FPU. We optimized this by restricting our scaling factor to powers of 2, allowing us to use bitwise shifts (`>>`) instead of division.

```c
int16_t EMA_Update(int16_t *current_estimate, int16_t new_measurement, uint8_t shift) {
    if (current_estimate == NULL || *current_estimate < 0) {
        return -1; // Error: current_estimate pointer is NULL
    }
    
    // Calculate the difference, shift it, and add to the estimate.
    // Every shift is a division by 2. 
    // shift = 1 applies a 50% weight to the new measurement.
    // shift = 2 applies a 25% weight, etc.
    *current_estimate = *current_estimate + ((new_measurement - *current_estimate) >> shift);
    
    return *current_estimate;
}

```

This is a massive performance win. A bitwise shift executes in a single clock cycle, making this filter incredibly lightweight.

## Testing and Statistical Analysis

To see how this performs in the real world, we aimed the sensor at a wall approximately 1870 mm away and captured 300 samples.

### Scenario 1: Fast Response (Buffer = 4, EMA Shift = 1)

![Filtering with 1/2 EMA and 4-slot buffer](assets/posts/silencing-the-noise-a-practical-guide-to-sensor-filtering-in-embedded-systems/1_4.png){: .d-block .mx-auto style="width: 65%;" }
_Filtering with a 1/2 EMA and 4-slot buffer_

For the first test, we used a small buffer of 4 slots for the median filter (fast update) and an EMA shift of 1 (which gives a 50% weight, or $1/2$, to the new measurement).

| Metric | Raw Data | Median Only | Median + EMA |
|:---:|:---:|:---:|:---:|
| **Average** | 1871.54 mm | 1873.10 mm | 1872.63 mm |
| **Median** | 1872.00 mm | 1873.00 mm | 1873.00 mm |
| **Variance** | 30.65 mm² | 11.47 mm² | 7.22 mm² |
| **Std Dev** | **5.54 mm** | **3.39 mm** | **2.69 mm** |
| **Range** | 31.00 mm | 19.00 mm | 16.00 mm |
{: style="width: auto; margin-left: auto; margin-right: auto;" }

Notice how the averages remain practically identical, meaning we haven't skewed our data. However, the standard deviation drops significantly from 5.54 mm down to 2.69 mm.

### Scenario 2: Heavy Smoothing (Buffer = 8, EMA Shift = 2)

![Filtering with 1/4 EMA and 8-slot buffer](assets/posts/silencing-the-noise-a-practical-guide-to-sensor-filtering-in-embedded-systems/2_8.png){: .d-block .mx-auto style="width: 65%;" }
_Filtering with a 1/4 EMA and 8-slot buffer_

Next, we tweaked the parameters to favor smoothness over speed. We doubled the median buffer to 8 slots, and increased the EMA shift to 2 (giving a 25%, or $1/4$, weight to new measurements).

| Metric | Raw Data | Median Only | Median + EMA |
|:---:|:---:|:---:|:---:|
| **Average** | 1871.12 mm | 1872.23 mm | 1870.74 mm |
| **Median** | 1871.00 mm | 1873.00 mm | 1871.00 mm |
| **Variance** | 29.19 mm² | 6.00 mm² | 3.10 mm² |
| **Std Dev** | **5.40 mm** | **2.45 mm** | **1.76 mm** |
| **Range** | 31.00 mm | 11.00 mm | 8.00 mm |
{: style="width: auto; margin-left: auto; margin-right: auto;" }

With these parameters, the response is incredibly clean, **reducing the standard deviation by a factor of 3X** compared to the raw data (down to just 1.76 mm).

## The Trade-off: Smoothness vs. Latency

In embedded systems, there are no free lunches. By heavily smoothing the data in Scenario 2, we made the system slower to react to sudden, legitimate changes in distance.

To illustrate this, we ran a final test where we introduced an obstacle mid-reading to observe the step response.


| ![Filtering with 1/2 EMA and 4-slot buffer](assets/posts/silencing-the-noise-a-practical-guide-to-sensor-filtering-in-embedded-systems/1_4_response.png){: style="width: 100%;" }<br><span style="display: block; text-align: center;">4-slot buffer with 1/2 EMA</span> | ![Filtering with 1/4 EMA and 8-slot buffer](assets/posts/silencing-the-noise-a-practical-guide-to-sensor-filtering-in-embedded-systems/2_8_response.png){: style="width: 100%;" }<br><span style="display: block; text-align: center;">8-slot buffer with 1/4 EMA</span> |

As seen in the graphs, the 4-slot and $1/2$ EMA filter catches up to the new obstacle distance much faster. This makes sense:

1. **EMA Factor:** The $1/2$ EMA gives twice the mathematical weight to new readings compared to the $1/4$ version.
2. **Median Lag:** An 8-slot buffer takes longer to flush out old data. If an obstacle suddenly appears, it takes at least 4 new readings (half the buffer) before the new distance becomes the median value.

**The Takeaway:** You must tune these parameters based on your project requirements. If you are building a collision avoidance system, you need the fast response of Scenario 1. If you are measuring the fluid level in a static tank, the heavy smoothing of Scenario 2 is perfect.

## Conclusion

Even a relatively simple set of filters can drastically improve the usability of a ToF sensor like the VL53L8CX. By combining a Median filter to strip outliers and a bit-shifted EMA filter to smooth the jitter, we achieved a 3X improvement in signal stability with minimal CPU overhead.

There are, of course, more complex algorithms out there, which I plan to implement and break down in future posts.

Until then, keep your code clean and your signals cleaner!

Source code: [vl53l8cx_filter](https://github.com/krloSc/vl53l8cx_filter).

---