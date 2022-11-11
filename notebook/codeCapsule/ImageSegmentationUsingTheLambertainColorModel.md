# Image segmentation using the Lambertain color model

> [Image segmentation using the Lambertain color model | Code Capsule](https://codecapsule.com/2010/09/01/background-segmentation-lambertain-color-model-opencv/)

![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/09/segmentation_intro.jpg?resize=720%2C264 "segmentation_intro")

As part of my research on image segmentation, I have explored different methods for selecting areas in an image. Recently, I found a statistical color model based upon Lambertain surface reflectance. I have implemented this model using OpenCV 2.1. This article presents the results of some experiments I have run, along with my personal feelings about the model. At the end of the article, you will find links to the source code and to the research papers I used.

## Introduction

In the [Segmentation](http://en.wikipedia.org/wiki/Segmentation_%28image_processing%29) article of Wikipedia, one can read:

> In computer vision, segmentation refers to the process of partitioning a digital image into multiple segments.

As stated above, the goal of image segmentation (also called  *subtraction* ) is to to detect regions of interest in an image. This can be done using color intensity, contrast, or any other metric that allows an acceptable detection. Working in a different color space can improve results in some cases. For instance, the YCbCr color space sometimes yields better results compared to the RGB space. In their publication on background segmentation, Horprasert et al. (1999) developed a statistical model for segmentation based on Lambertain surface reflectance. They demonstrate the efficiency of their model for background segmentation, which I have been able to reproduce. I have also tested the approach of Yacoob and Davis (2006), who use the same model for color segmentation in a single image, but the results I obtained were not satisfying.

## Lambertain color model

The model is quite simple. *N* background images are used to train the model. These images have same size and same resolution, and are views of the same area in space, taken without changing the orientation, position or zooming of the camera. The subject of the picture should be only background in all images, and only the illumination conditions are allowed to change. The model is computed for every pixel position across all *N* images. For each pixel, brightness and chromaticity are evaluated, and a statistical model for that particular pixel is computed. Using a detection rate, we can then tell which values of brightness and chromaticity can be used as threshold to select or reject novel pixel intensities, just like when performing statistical tests.

Once the model has been computed, we can input a test image in the model in order to perform background detection. Each pixel is classified separately, and for each pixel we use the model that corresponds to its position in the image. By comparing the brightness and chromaticity for this pixel to the pre-computed thresholds, we can tell if the pixel is part of the foreground or background.

## Background segmentation experiment

I have implemented the model described in Horprasert et al. (1999), and I have tested it on my own dataset. As training background dataset, I have used images of a room under different illumination conditions, taken with a low-quality webcam (this dataset is available for download at the end of the article). Figure 1(a) shows one of the 26 background images that I am using, and Figure 1(b) shows the test image, in which I want to detect an object.

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation03.jpg?resize=720%2C314 "Figure 1: Background segmentation: Background image and test image")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation03.jpg)
Figure 1: Background segmentation: Background image and test image

I have tried the model on the test image for various detection rates. The resulting classifications are shown in Figure 2. Each pixel is colored depending on the class it is classified in.  **Blue is foreground, Green is background, Red is background shadow, and Black is background highlight** . As we can see, the background segmentation accuracy increases with the detection rate. In Figure 2(a), with a detection rate of 80%, the foreground object almost melts with the background, whereas in Figure 2(f), with a detection rate of 99.9999%, the foreground object is properly segmented from the background. These results are very encouraging, and they confirm what is presented in Horprasert et al. (1999). Also, Horprasert et al. (1999) trained their model on sets of 100 images, whereas I used only 26 images to obtain the results presented in Figure 2.

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation01.jpg?resize=720%2C950 "Figure")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation01.jpg)
Figure 2: Background segmentation: Classification at various detection rates. The detection rates are (a) 80%, (b) 90%, (c) 95%, (d) 99%, (e) 99.999%, and (f) 99.9999%.

In order to make sure we are classifying the right thing, a quick experiment to do consists of using as test image an image that is part of the training set. Figure 3 below presents the result of such a segmentation. As we can see, all we get is plain background (green pixels), with some minor clusters of other pixel classes. This is what is expected, since the test image is just formed with background, and therefore we do not expect to find any foreground object in it.

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation04.jpg?resize=350%2C263 "Figure 3: Background segmentation: Classification of background image")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation04.jpg)
Figure 3: Background segmentation: Classification of background image

## Tips for building a valid training dataset for background segmentation

Let says that you want to detect cars in a parking lot. Therefore you need to put your camera on a tripod to make sure it won’t move, and then you take pictures at different moments of the day to have some variation in the illumination. These pictures will be your training background pictures. But as you want to detect the cars in this parking lot, you cannot use a picture with a car in it. In order to optimize the model, you need to only have a plain background in all of your training pictures.

Also, the quality of the training images is very important. In my background experiment, I have used a low-quality webcam, with really poor colors and contrast. This has been the source of several pixel misclassifications, which are not due to the model, but simply to image quality. This issue is not new, and can be observed with many other image processing techniques such as edge detection, or image alignment for instance. Therefore, the better the image quality, the better the accuracy of the classification.

## Color segmentation experiment

In a more recent publication regarding hair detection in facial images, Yacoob and Davis (2006) have discussed the utility of the Lambertain color model in a color segmentation context. Instead of having a set of *N* training images and one test image, they only use one image. In that image, representing a human face, some pixels are known to represent hair. These pixels are used to train the model, and the model is applied on this same image to detect all the pixels representing hair. They seem to get quite good results, so I wanted to give it a try and see what the Lambertain color model would give when trained such as described by Yacoob and Davis. To do that, I use the test image from my background experiment, and I tell the model to use only the pixel of a region in that image in order to perform the training. This regions is the blue rectangle shown on Figure 4(a). I have chosen this region as it contains most of the yellowish pattern that constitutes the foreground object. Figure 4(b) shows the classification of the image in Figure 4(a), based on the model computed using the rectangular region. As we can see, here the model fails completely. I have tried multiple detection rates, but not a single value led me an acceptable segmentation. I have also tried to segment hair from facial images, and again, it failed. So either I am missing some important condition on the data in order to get a valid segmentation as presented in Yacoob and Davis (2006), or my implementation has a bug (which is possible, but unlikely, since the background segmentation is working properly).

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation02.jpg?resize=720%2C314 "Figure 4: Color segmentation: Input image and classification")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/segmentation02.jpg)
Figure 4: Color segmentation: Input image and classification

Furthermore, something interesting has to be noticed. I don’t know if this is a typo or a modification in the model, but the formula used by Yacoob and Davis for brightness distortion (Eq. 3 in Yacoob and Davis, 2006), differs from the formula used by Horprasert et al. (Eq. 5 in Horprasert et al., 1999). Indeed, the intensity and mean terms are squared in the numerator in Yacoob and Davis, whereas in Horprasert et al. they are not. From Yacoob and Davis’ perspective, it appears to me that since they referenced the model of Horprasert et al., they would have explicitly pointed out any modifications to it. Therefore the hypothesis that it is just a typo seems more plausible. I have made experiment both formulas for color segmentation, and the results are equally bad (at least in my implementation). In order to switch from one version of the formula to the other, take a look at the computeModelMeanStdDev() method in the SingleLCM class, and at the computeBrightnessDistortion() method in the LCM class.

## Source code

The source code for the Lambertain Color Model that I implemented is available under the GNU License 3.0.
It can be browsed here: [http://github.com/goossaert/computer-vision/tree/master/lambertain/](http://github.com/goossaert/computer-vision/tree/master/lambertain/),
and downloaded here: [http://github.com/goossaert/computer-vision/archives/master](http://github.com/goossaert/computer-vision/archives/master).

After you have downloaded it, unpack it and open the Makefile to set the library and include locations to the directory containing OpenCV 2.1 on your computer:

```
$ tar zxvf lambertain.tgz
$ cd lambertain
$ vim Makefile
$ # ... update the library and include locations
```

Now just run *make* to compile:

```
$ make
```

This will create three executables:

* **background** . Performs background segmentation. You can run the program without option to get the list of parameters it requires. Also, the shellscript “test_background.sh” runs the program on the dataset from the article.
* **color** . Performs color segmentation. You can run the program without option to get the list of parameters it requires. Also, the shellscript “test_color.sh” runs the program on the dataset from the article.
* **videocapture** . A small utility that allows you to capture images with a webcam. This is what I used to create my training set, so if you have a webcam, you can use this utility to create your own dataset and rapidly try the algorithm. If you do not have a webcam, I have included the training images that I used with the source code, so that you can experiment with that.

If you are working on Windows, you will need to create a project in the IDE that you use, and add the model files to it.

## Possible improvements and optimizations

* **Threshold selection** . Right now, the algorithm to select the thresholds is very inefficient. Indeed, I simply sort the data and use indexes to find the thresholds, yielding to a complexity of a O(n lg n). This could be optimized by using a selection algorithm, which would bring complexity down to O(n).
* **Multiply, not divide.** Multiplying by the inverse of the denominators instead of dividing by these denominators would speed up computations, as explained in Section 7 of Horprasert et al. (1999).
* **Global variance** . Using global average variance instead of local variance would speed up computations, as explained in Section 7 of Horprasert et al. (1999).
* **Clustering Detection Elimination** . I have not implemented the technique covered in Section 5 of Horprasert et al. (1999), which rectifies the erroneous chromaticity distortion values. This would allow the model to produce more accurate results.

## Conclusion

In this article, I have presented my implementation of the Lambertain Color Model for image segmentation. I have used the work of Horprasert et al. (1999) and of Yacoob and Davis (2006) in order to implement this model. I have been able to reproduce the results presented by Horprasert et al. for background segmentation, but I haven’t been able to do so for the results of Yacoob and Davis. The training of the model is quite slow now, but could be made faster with some the optimizations that I have listed above. Along with the source code for the model, I am also giving a small utility that allows you to create your own dataset with your webcam. And if you do not have a webcam, the dataset used in this article is available with the sources.

Let me know if you use this model in your projects, I’d be happy to see that!

## References

[T. Horprasert, D. Harwood, and L.S. Davis. A statistical approach for real-time robust background subtraction and shadow detection.  *IEEE ICCV* , 1999.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.1244&rep=rep1&type=pdf)
[Y. Yacoob and L.S. Davis. Detection and Analysis of Hair.  *IEEE Transactions on Pattern Analysis and Machine Intelligence* , 1164-1169, 2006.](http://www.umiacs.umd.edu/~yaser/pamis.pdf)
