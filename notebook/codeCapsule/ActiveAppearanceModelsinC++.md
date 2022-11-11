# Active Appearance Models in C++ (Paamela)

> [Active Appearance Models in C++ (Paamela) | Code Capsule](https://codecapsule.com/2010/08/12/active-appearance-models-in-c-plus-plus/)

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/aam_fit.jpg?resize=700%2C455 "AAM Fitting")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/aam_fit.jpg)

My implementation of the Active Appearance Models (AAMs) in C++ is almost done, it is called  **Paamela** . I am currently fixing a couple design issues and finishing up the documentation. Even though I am still not sure whether or not I will make the code open source, I thought it would be nice to share what I have developed so far, in order to help other developers working on similar problems.

## What is Paamela?

 **Paamela** , my implementation of the AAMs, is based on Ian Matthews and Simon Baker’s [Inverse Compositional approach](http://www.ri.cmu.edu/research_project_detail.html?project_id=448). I have developed it in C++ with OpenCV 2.1, and I am really proud to say that it is very accurate and really fast! As of today, the AAMs open source implementations available on the Internet are only partly working, and their respective source codes look pretty messy to me (I can say that because I actually took the time to take a look at the source codes). I am not going to cite any project name here, because I do not want to be disrespectful to the developers who spent time implementing them. A rapid lookup in your favorite search engine will orient you, from there read the code, and make your own opinion. I am not criticizing their design choices, but the coding style, the naming conventions, and the documentation. Indeed, they are inexistent! Therefore, except for the developers themselves, no one is going to understand what the code is doing, and as a result, no one will really be able to reuse the code (which was the whole point of making it open source in the first place). The only clean open source AAMs available now is [Mikkel B. Stegmann’s AAM-API](http://www2.imm.dtu.dk/~aam/), but unfortunately this is not implementing the inverse compositional approach. Since I wanted to have a clean C++ Inverse Compositional AAMs implementation, and as it was inexistent, I decided to do it myself.

I have tried to make the code as clean as possible to enable other developers to understand what the different equations are doing. I have also written an API with a detailed documentation, in order to allow users to understand how to adapt it to their specific problems. The API allows users to create models based on collections of appearances, and to perform AAM fitting of input images from these models. I have also made the implementation independent from the type of data being treated, which enables **Paamela** to handle any kind of input data: faces, hands, bone structures, etc.

## Implemented features

I have implemented many features for  **Paamela** , in order to make the fitting more accurate. These features include:

1. Different illumination normalization techniques: HE (Histogram Equalization) and CLAHE (Contrast Limited Adaptive Histogram Equalization).
2. Object detection to initiate the fitting process: for instance, Haar Feature-based Cascade Classifiers can be specified to roughly detect the position of the objects to fit, and enhance the accuracy of AAMs fitting tasks.
3. Bilinear interpolation during appearance warping.
4. Thresholded variances for the shape and the appearance statistical models.
5. Damping for the gradient-descent update step.
6. Convergence detection.

And I will add soon:

1. Support for fitting using live video through OpenCV.
2. Multi-resolution model based on a Gaussian pyramid.

## Example of a fitting process

In order to show what **Paamela** can do, I am using here the [IMM Face Database](http://www2.imm.dtu.dk/pubdb/views/publication_details.php?id=3160), which is freely available. Be careful if you use this database: people have been nice enough to put this together, so please respect the terms of use given in the description.

For this example, I first build a model based on four different persons. Then I am using a Haar cascade classifier to estimate the initial configuration of the face. Finally, the fitting process goes on and clips the facial features in the input image. Figure 1 shows this fitting process. After only seven iterations, the fit is done, and convergence is declared after 10 iterations.

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/aam_fit.jpg?resize=700%2C455 "AAM Fitting")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/aam_fit.jpg)
Figure 1: AAM Fitting

## The development environment

I am also including below a view of what my development environment looks like. While I am performing experiments, I constantly keep an eye on every single matrix in the model, as shown below in Figure 2. Oh, and you can see *vim* in the background :😉:

[![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/aam_debug-720x450.jpg?resize=720%2C450 "Development environment")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/08/aam_debug.jpg)
Figure 2: AAM development environment (click to enlarge)

## Conclusion

So here it is, **Paamela** is almost ready! I am probably not going to make it open source right now, as I am still deciding if I want to implement something else based on it, and then release this thing and **Paamela** all at once. Nevertheless, if you have any questions or if you want to know more, feel free to drop a comment :🙂:
