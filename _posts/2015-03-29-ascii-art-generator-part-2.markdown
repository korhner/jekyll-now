---
layout: post
title: "Ascii art generator in Java - Part 2"
date: 2015-03-29 16:54:46
categories:
- Java
- Image-Processing
published: true
---

Since I got a lot of feedback for my [my previous post]({% post_url 2015-02-14-ascii-art-generator %}) 
where I built a very simple Ascii art generator in Java (find it on <a href="https://github.com/korhner/asciimg">GitHub</a>), I decided to continue with the project and add some more features to it you will hopefully enjoy even more. 
I've redesigned the major part of it and made it very extensible so it is very easy to use to test different algorithms, create different outputs, etc.
In this post, I will present the new architecture of the project, so you can easily integrate it into your own code and extend it as needed.
<!--more--> 

## Architecture:


![_config.yml]({{ site.baseurl }}/assets/img/asciimg/asciimg_cls_diagram.png)

<hr/>  

### AsciiImgCache

Before any ascii art rendering takes place, it is necessary to create an instance of this class. 
It takes a font and a list of characters to use as parameters and it creates a map of images for every character.
There is also a default list of characters if you don't want to bother comming up with your own.  
In case you are curious:
{% highlight Java %}
private static final char[] defaultCharacters = 
    "$@B%8&WM#*oahkbdpqwmZO0QLCJUYXzcvunxrjft/\\|()1{}[]?-_+~<>i!lI;:,\"^`'. "
{% endhighlight %}

Example:
{% highlight Java %}
// use only '/' '\' and ' '
AsciiImgCache mediumBlackAndWhiteCache = AsciiImgCache.
    create(new Font("Courier", Font.BOLD, 10), new char[] {'\\', ' ', '/'});

// use default list
AsciiImgCache largeFontCache = AsciiImgCache.
    create(new Font("Courier",Font.PLAIN, 16));
{% endhighlight %}

<hr/> 

### BestCharacterFitStrategy

This is the abstraction of the algorithm used for determining how similar a part of the source image with each character is. It has one method:
{% highlight Java %}
float calculateError(final GrayscaleMatrix character, final GrayscaleMatrix tile);
{% endhighlight %}

The implementation should compare two images and return a float error. Each character will be compared and the one that returns the lowest error will be selected. 
Currently there two implementations available: ColorSquareErrorFitStrategy and StructuralSimilarityFitStrategy.

#### ColorSquareErrorFitStrategy

Very simple to understand, it compares every pixel and calculates <a href="http://en.wikipedia.org/wiki/Mean_squared_error">Mean squared error</a> of the grayscale differences.
Mathematically speaking its:    
![_config.yml]({{ site.baseurl }}/assets/img/asciimg/mse.gif)  
Where n is the number of pixels, and C and T are pixels from character and tile image respectively.

#### StructuralSimilarityFitStrategy

The structural similarity (SSIM) index algorithm claims to reproduce human perception and its aim is to improve on traditional methods like MSE.
I will not get into much details about how it works, you can read more on <a href="http://en.wikipedia.org/wiki/Structural_similarity">Wikipedia</a> if you want to know more.
I experimented a bit with it and implemented a version that seemed to produce the best results for this case.

<hr/> 

### AsciiConverter<T>

This is the hearth of the process and it contains all the logic for tiling source image and utilizing concrete implementations for calculating character best fit.
However, it doesn't know how to create the concrete ascii art - it needs to be subclassed. 
There are two implementations currently: AsciiToImageConverter and AsciiToStringConverter - which as you probably guessed, produce image and string output.

<hr/> 

## Example usage

Since a code snippet is worth a thousand words, I will show you the whole process in action that should wrap up all the pieces:

{% highlight Java %}
// initialize cache
AsciiImgCache cache = AsciiImgCache.create(new Font("Courier",Font.BOLD, 6));

// load image
BufferedImage portraitImage = ImageIO.read(new File("image.png"));

// initialize converters
AsciiToImageConverter imageConverter = 
    new AsciiToImageConverter(cache, new ColorSquareErrorFitStrategy());
AsciiToStringConverter stringConverter = 
    new AsciiToStringConverter(cache, new StructuralSimilarityFitStrategy());

// image output
ImageIO.write(imageConverter.convertImage(portraitImage), "png", 
    new File("ascii_art.png"));
// string converter, output to console
System.out.println(stringConverter.convertImage(portraitImage));
{% endhighlight %}

Here are some sample images generated with various parameters:

<a href="/assets/img/asciimg/orig.png">Original picture</a>

<a href="/assets/img/asciimg/large_square_error.png">16 pts font, MSE</a>  
<a href="/assets/img/asciimg/large_ssim.png">16 pts font, SSIM</a>  
<a href="/assets/img/asciimg/medium_square_error.png">10 pts font with 3 characters, MSE</a>  
<a href="/assets/img/asciimg/medium_ssim.png">10 pts font with 3 characters, SSIM</a>  
<a href="/assets/img/asciimg/small_square_error.png">6 pts font, MSE</a>  
<a href="/assets/img/asciimg/small_ssim.png">6 pts font, SSIM</a>  

## Further work

Some ideas that I have in mind:

* Research and try to implement more image comparison algorithms
* Try to preprocess the image to get better results (improve contrast, use edge detection, etc.)
* Image processing could be parallelized for performance improvent, try it and measure the gain to see if is worth it.
* Add some more converters (i.e. html output)
* Add support for output with colored characters
* Add tests to the project

If have any ideas for improvement or find any problems with the code, feel free to comment or contribute to this repository via <a href="https://github.com/korhner/asciimg">GitHub</a>!