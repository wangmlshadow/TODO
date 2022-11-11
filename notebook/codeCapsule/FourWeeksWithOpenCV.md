# Four weeks with OpenCV

> [Four weeks with OpenCV | Code Capsule](https://codecapsule.com/2010/07/26/four-weeks-with-opencv/)

As part my C++ implementation of the AAMs (Active Appearance Models), I have been using OpenCV 2.1 full-time for about four weeks now. I thought I would share my feeling of the pros and cons about OpenCV. After reading this post, you should be able to know whether OpenCV is the solution you need or not.

## Why OpenCV?

AAMs are a family of algorithms of Computer Vision, and require matrix computations and some image processing. So after choosing C++ as the language for my implementation, I needed to find two solid open source libraries, one for the algebra and another one for the image processing. I wanted to have real C++ matrix operations to have a clean C++ class system and not a C interface. Why? Because matrix libraries with C interfaces make you deal with a lot of intermediate variables and let you manage the memory, which is really error prone. I looked around and found things like GSL (GNU Scientific Library) and Blitz++ for algebra, and IPL, CImg and OpenCV for image processing. Then, the important point was to choose the right combination of algebra/image library, since I would need to convert from matrix to image and conversely. It turned out that OpenCV actually had a complete C++ matrix interface, with everything needed for basic algebra operations. OpenCV could take care of converting between matrices and images matrices, so I wouldn’t need to bother about it. Therefore, I have chosen OpenCV, as it gives me C++ matrices with fast algebra along with image processing, and this is all in one library. Awesome!

## Make sure you use the right interface of OpenCV

I have been sneaking on the Internet for open source applications that are using OpenCV, in order to see what production code using OpenCV looks like. What really bugged me is that very often people are not using it right. They are developing C++ applications, but they are using the C interface to OpenCV. They allocate and release the memory themselves, and they do not use the *Mat* class, which is at the core of the awesomeness of the C++ interface. So make sure that you are using the right interface to OpenCV. If you want to code in C, this is totally fine, use the C interface. But if you choose C++, then make sure you use the C++ interface, and that you access you data using classes and not pointers to structs.

## My personal feeling about OpenCV

While implementing the AAMs with OpenCV, I learned a lot about the different things that OpenCV has to offer. Globally, I think that OpenCV is a really good library and I strongly recommend its use for image processing development, but there are still a few things that are annoying me. So here I make a list of the pros and cons I have found.

### Matrix operations

#### Pros:

* **Real C++ interface:** Matrices are handled by the *Mat* class. This class is just a header containing information about the matrix and a pointer to the array that really stores the matrix values. When you assign a matrix to another, only the header is copied, and a counter is incremented to keep track of how many variables have a reference to the array storing the values. When the counter reaches zero, the array is released. So you do not have to worry about the memory allocation and deallocation, OpenCV takes care of that for you. Finally, even though this header solution make function calls and returns easier, the fastest way to pass matrices remains to use  *const references* .
* **Matrix expressions:** As C++ enables operator overriding, the Mat class offers a large set of operations that you can apply directly to Mat objects. For instance, if you want to perform an element-wise addition of two matrices, you do not need to call a function such as `add(mat1, mat2)`, you can directly type `mat1 + mat2`. This makes the translation from equations to code a lot easier.
* **Ready-to-use algorithms:** OpenCV also has many algorithms for algebra that are working out of the box. This includes PCA (Principal Component Analysis) and SVD (Singular Values Decomposition) for instance. Many scientific computing libraries I have come across do not have all these algorithms available, and this can considerably slow down the development process of an application. Indeed, imagine you just need to have PCA implemented. PCA is pretty simple to program, but it would require you a day to program it cleanly, given that you are willing to spend the time to code all the necessary unit tests in order to make sure your computations are correct. And even then, if later some of your code based on this PCA is not working, you will still have doubt about your implementation. Well, OpenCV is providing you with all you need, tested, fast, and ready to use. Absolutely awesome.

#### Cons:

* **Column and row access:** It is possible to get individual or ranges of rows and columns. Let’s say you want to assign something to the first row of a matrix:

  ```
  Mat A = Mat::zeros( 3, 4, CV_32F ); // 3-by-4 matrix filled with 0
  Mat foo = Mat::ones( 1, 4, CV_32F ) * 4; // row matrix filled with 4
  A.row( 0 ) = foo; // set the first row of A to the values contained in foo (but this is not working)
  ```

  Now we are supposed to have the values in the first row of A all set to 4, right? Well, this is not the case. Why? Because when accessing ranges of rows or columns, new matrices are created with a new header and a pointer to the area in the array that just refers to the range of rows or columns, here just one row. When assigning something to that row, the header is replaced, and only the pointer to the array that contains the values is updated: the values in the original matrix are left unchanged. In order to update the values correctly, we need to use the addition-assignment operator instead:

  ```
  Mat A = Mat::zeros( 3, 4, CV_32F ); // 3-by-4 matrix filled with 0
  Mat foo = Mat::ones( 1, 4, CV_32F ) * 4; // row matrix filled with 4
  A.row( 0 ) += foo; // set the first row of A to the values contained in foo (this is working)
  ```

  So I don’t know if I have missed something while reading the OpenCV documentation, but I am pretty sure I have not. And even in the case that I missed something, this behavior seems really inconsistent and I consider it to be a huge flaw in the C++ matrix interface of OpenCV. It forces programmers to think in terms of how the matrix is handled inside the class. Therefore the encapsulation is broken and the inner design of the class interferes with the choices programmers make. In my opinion, this is wrong.
* **Matrix expressions are limited:** Matrix expressions are a huge improvement compared to the C interface of OpenCV, as they allow to reduce the number of function calls and intermediate variables. However, other languages and platforms, such as matlab, are still doing a better job in my opinion. For instance, a mere gradient evaluation using automatic differentiation takes me only 2 lines in matlab. With OpenCV, it requires me 10 lines with the C++ interface, and 16 lines with the C interface. So the C++ interface really brings some improvements, but we are still really far from what is possible with fully math-oriented languages.

### Image processing

#### Pros:

* **Easy I/O operations:** It is really easy to read and write images in multiple format from files. Only one function call is necessary to get the data of the image.
* **Ready-to-use filters:** Just like for the algebra, many algorithms have been included. You have histogram equalization, Gaussian blur, resizing, convolution filters, and many others, working out of the box!
* **Conversion between images and matrices:** Compared to the C interface, the C++ interface makes it really easy to access images like matrices. Actually, when you read an image you get its content directly in a matrix. This allows algebra operations on images to be performed without worrying about any conversion issues.
* **Easy display:** It is really easy to display an image. So when developing an algorithm that modifies images, it is really easy to plot the image to check visually what is going on. This is extremely useful when debugging.

#### Cons:

* **Image representations:** Depending on how the image is accessed, it will be either a one channel matrix (for grayscale images), or a three-channel matrix (for RGB images). Then you have to treat the matrices differently depending on the primitive type they are made of (unsigned char, float, …) and the number of channels they have. So if you want to have the best performance, you will need to have different functions for all the different representations of images you have in your application.

## Conclusion

Even though some flaws are present, OpenCV remains an amazing open source library that can help you program faster image-related applications. The real C++ matrix interface, along with the easy access to images and the incredible amount of included algorithms one can use out of the box, OpenCV is to me the best available solution as of today. The access to matrices is not perfect, as we have seen for the row and column ranges, but I think other C++ matrix interfaces would probably present the same issues (this needs to be compared to other libraries). One thing is for sure: it is C/C++, therefore it is fast! And actually, given all the ready-to-use linear algebra methods that are already implemented in OpenCV, I am wondering if OpenCV could be used for algebra only, and replace scientific libraries.
