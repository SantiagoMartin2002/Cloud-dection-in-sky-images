# Cloud detection in sky images

This repository presents an easy-to-implement yet effective algorithm for detecting clouds in ground-based sky images. The algorithm has been used in the study by [Nie et al. (2020)](https://doi.org/10.1063/5.0014016) [[1]](#1) to identify cloud pixels in sky images for sky condition classification, which enables the development of tailored models for solar nowcasting specific to each sky condition. Here, we demonstrate the effectiveness of our algorithm using some sample data. The full dataset can be assessed via our [SKIPP'D GitHub Repository](https://github.com/yuhao-nie/Stanford-solar-forecasting-dataset).


## Context
Clouds significantly impact PV panel output and pose challenges to the large-scale deployment of solar PV systems. Accurate detection of clouds in sky images is a critical step in cloud modeling, which is essential for reliable solar power forecasting. 

![demo_intermittency](/figs/demo_pv_output_cloudy_day.gif)
<p align=justify>
Figure 1. Power output of a 30-kW roof-top PV system on a partly cloudy day. Note how PV output fluctuates during the cloud events.
</p>

## Cloud detection algorithm
The cloud detection algorithm essentially performs binary classification, i.e., distinguishing each pixel in a sky image as either a cloud or non-cloud pixel. Various methods have been developed for this purpose, ranging from simple thresholding techniques to more complex approaches such as neural network models, Markov Random Fields models, and Gaussian Mixture Models [[2]](#1).

We select the thresholding method, favored for its simplicity (no demanding image labeling involved like machine/deep learning methods), low computational requirements (no GPU required), and capability to implemented near real time. Our algorithm specifically integrates the **normalized red blue ratio (NRBR)** and **clear sky library (CSL)** methods. To better understand our proposed approach, it is essential to first introduce NRBR and CSL.

### NRBR
$\mathrm{NRBR}$ is defined by the following equation: $$\mathrm{NRBR = \frac{B-R}{B+R}}$$
where R and B stand for the pixel values of the red and blue channels of a sky image, , ranging from 0 to 255. According to the Rayleigh scattering law, the intensity of scattered light in clear air is inversely proportional to the fourth power of the wavelength [[3]](#2). This implies that the blue spectrum of visible light is predominately scattered in the clear atmosphere, whereas cloud particles scatter blue and red light almost equally [[2]](#2). It results that clear sky appears blue (pixels with high $\mathrm{NRBR}$) and clouds appear white or grey (pixels with low $\mathrm{NRBR}$). 

Applying a $\mathrm{NRBR}$ threshold pixel-wise (i.e., the pixels are identified as clouds when their $\mathrm{NRBR}<=0.05$) works well when the sun is totally shrouded by clouds, but the results are not satisfactory when it is clear sky or when the sun is partially shrouded by clouds (see Figure 2). The alrgorithm misclassifies the circumsolar pixels as cloud pixels because such pixels are often brighter than other areas and have a white or yellow-white character that activates the $\mathrm{NRBR}$ threshold. 

![demo_NRBR](/figs/demo_NRBR.png)
<p align=justify>
Figure 2. Cloud detection using NRBR method for different sky conditions: (a) sun entirely shrouded by
clouds; (b) clear sky; and (c) sun partially shrouded by clouds. The original sky image is on the left, and the image showing identified cloud pixels is on the right. Red pixels in original image indicates the sun position. Green pixels in the image indicates the identified cloud pixels. Note the misclassification of the pixels in the circumsolar area for scenario (b) and (c).
</p>

### CSL
Chauvin *et al.* [[2]](#2) apply a background subtraction to tackle the issue of misclassifying pixels in the circumsolar area. The essence is to apply a background subtraction using a clear sky image, which is generated from the original image by a fitted empirical sky luminance model, and set the threshold to the difference of $\mathrm{NRBR}$ between the original image and the clear sky image.

We adopt this idea, but instead of generating clear sky image from the original image, we establish a clear sky library (CSL) by sampling 4 clear sky days from our [dataset](https://github.com/yuhao-nie/Stanford-solar-forecasting-dataset), i.e., May 20, August 15, September 23 and October 22 of 2017, with 5-minute frequency. These days are found to be sufficient for representing the track of the sun over a year [^1]. For background subtraction, we match the original image with a clear sky image from the CSL with a similar sun position. The sun position identification is a separate algorithm developed in this study based on camera projection model and the sun angle, which is detailed in the following subsection. 

[^1]: Our study [[1]](#1) only uses 1 year of data from 2017 March to 2018 Feburary, so we find these 4 days are enough for our case. You can expands the CSL based on your needs.

The difference of $\mathrm{NRBR}$ between the original image and the corresponding clear sky image is calculated, i.e., $\Delta\mathrm{NRBR =| NRBR - NRBR_{cs} |}$. $\Delta\mathrm{NRBR}$ is low for clear-sky pixels but is high for cloud pixels in the original image due to the background subtraction. A threshold value of 0.175 is selected empirically and used to identify the cloud pixels (i.e., if $\Delta\mathrm{NRBR}\geq0.175$, the pixels are identified as clouds).

The CSL method can handle the circumsolar area misclassification well for the clear sky condition. However, the method fails when there are clouds near the circumsolar area or when the sun is shrouded by clouds, as the circumsolar area is subtracted out from the clear sky image (see Figure 3 below). The cloud pixels in the circumsolar area are not identified as cloud pixels since the $\Delta\mathrm{NRBR}$ values of these pixels will be lower than the threshold after the clear sky subtraction of the bright circumsolar area.

![demo_CSL](/figs/demo_CSL.png)
<p align=justify>
Figure 3. Cloud detection using CSL method for different sky conditions: (a) sun entirely shrouded by
clouds; (b) clear sky; and (c) sun partially shrouded by clouds. The original sky image is on the left, and the image showing identified cloud pixels is on the right. Red pixels in original image indicates the sun position. Green pixels in the image indicates the identified cloud pixels. Note that although clear sky condition is handled correctly, the circumsolar area is inadvertently substracted out when the sun is entirely obscured by clouds.
</p>

### Our approach (NRBR+CSL)
To correct this flaw of using NRBR and CSL indiviudally, we develop an algorithm that takes advantages of the two methods (referred to as NRBR+CSL hereafter). The process diagram of the NRBR+CSL method is shown in Figure 4 below.

![cloud_detection_alg](/figs/cloud_detection_algorithm.png)
<p align=justify>
Figure 4. The process flow of our proposed cloud detection algorithm. The algorithm makes use of both NRBR and CSL.
</p>

The method uses filters added to the output cloudiness of the CSL method. Specifically, if the cloudiness obtained is lower than a threshold value 0.045, CSL method is used. If it is higher than another threshold value 0.35, then we switch to the NRBR method. If it is in between these two thresholds, then we choose the NRBR method, but only apply it outside of the circumsolar area in the original image. The examples of results from NRBR+CSL method can be found in Figure 5, which show the effectiveness of the method for cloud detection.

![demo_NRBR_CSL](/figs/demo_NRBR_CSL.png)
<p align=justify>
Figure 5. Cloud detection using NRBR+CSL method for different sky conditions: (a) sun entirely shrouded by
clouds; (b) clear sky; and (c) sun partially shrouded by clouds. The original sky image is on the left, and the image showing identified cloud pixels is on the right. Red pixels in original image indicates the sun position. Green pixels in the image indicates the identified cloud pixels. Note all three sky conditions are handled nicely by the proposed NRBR+CSL method.
</p>

### Sun position identification
A sun position identification algorithm is developed for use in the cloud detection. The major steps of the sun position identification algorithm is shown in Figure 6 below. Polar coordinates $(\rho,\theta)$ are used to determine the position sun in a sky image and then converted to Cartesian coordinates $(x^*,y^*)$. $\rho$ and $\theta$ are computed respectively based on the linear correlation with zenith angle $\chi$ and azimuth angle $\xi$ of the sun via camera projection models [[4]](#4). The sun zenith angle $\chi$ and azimuth angle $\xi$ vary with time of year and day and are estimated through empirical functions by Da Rosa [[5]](#5). It should be noted that the daylight savings time change is considered for the estimation of the solar angles.

![sun_identification_alg](/figs/sun_position_identification_algorithm.png)
<p align=center>
Figure 6. Algorithm for identifying the sun position in a sky image.
</p>

## Codes and data
The Python codes for the sun position identification (`sun_position_identification.py`) and cloud detection (`cloud_detection.py`) algorithms are provided in the `codes` directory. In the same folder, the notebook file (`demo.ipynb`) provides guidelines on how to use the functions within the two Python code files. All dependencies are listed in the file `requirements.txt`.

As our cloud detection method is non-parametric, no image labels are required. For demonstration purposes, we select sky images from 3 sunny days and 3 cloudys in 2017 with 1-minute temporal resolution [^2], to validate the sun position identification and the cloud detection algorithms. The image data are from our curated sky image and PV power generation dataset SKIPP'D ([Dataset paper](https://doi.org/10.1016/j.solener.2023.03.043) and [Github repo](https://github.com/yuhao-nie/Stanford-solar-forecasting-dataset)). Data for validating the algorithms are provided in the `sample_data` directory. A description of the data files are provided in the table below.

[^2]: To save time for implementation the algorithm and visualizing the results, we sample it every 3 minutes in the `demo.ipynb`.

|File|Description|
|-----|-----|
|`sky_images_sunny.npy`|Python NumPy array consists of 3 days of sunny sky image data.|
|`time_stamps_sunny.npy`|Python NumPy array of the time stamps corresponding to the sunny sky image data.|
|`sky_images_cloudy.npy`|Python NumPy array consists of 3 days of cloudy sky image data.|
|`time_stamps_cloudy.npy`|Python NumPy array of the time stamps corresponding to the cloudy sky image data.|
|`clear_sky_library/csl_images.npy`|Python NumPy array of 4 days of clear sky image data contained in the clear sky libaray (for use in CSL method).|
|`clear_sky_library/csl_times.npy`|Python NumPy array of time stamps corresponding to the clear sky images.|
|`clear_sky_library/csl_sun_center.npy`|Python NumPy array of the sun position coordinates in formate of (X,Y) corresponding to the clear sky images.|

## Use case demonstration

The videos below show case our sun position identification algorithm and cloud detection algorithm.

https://github.com/yuhao-nie/Cloud-dection-in-sky-images/assets/29718809/3be2315b-aafa-4606-86ec-317a49388eac

https://github.com/yuhao-nie/Cloud-dection-in-sky-images/assets/29718809/ff788b0a-c092-4e59-bc6c-1cdaf80f5939

## Limitations and future work
poor adaptation, parameters need to be tuned for specific dataset. 

Neural network, need labels

## References
<a id="1">[1]</a> 
Nie, Y., Sun, Y., Chen, Y., Orsini, R. and Brandt, A., 2020. PV power output prediction from sky images using convolutional neural network: The comparison of sky-condition-specific sub-models and an end-to-end model. Journal of Renewable and Sustainable Energy, 12(4).

<a id="2">[2]</a> 
Chauvin, R., Nou, J., Thil, S., Traore, A. and Grieu, S., 2015. Cloud detection methodology based on a sky-imaging system. Energy Procedia, 69, pp.1970-1980.

<a id="3">[3]</a>
Bohren, C.F. and Huffman, D.R., 2008. Absorption and scattering of light by small particles. John Wiley & Sons.

<a id="4">[4]</a>
Garcia-Gil, G. and Ramirez, J.M., 2019. Fish-eye camera and image processing for commanding a solar tracker. Heliyon, 5(3).

<a id="5">[5]</a>
Da Rosa, A.V. and Ordóñez, J.C., 2021. Fundamentals of renewable energy processes. Academic Press.
