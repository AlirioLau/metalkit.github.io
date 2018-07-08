---
published: true
title: Introducing the Metal framework
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/chapter01_3.png" alt="Metal" height="150" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">What is Metal? What are its advantages over other frameworks? Which GPUs are supported by Metal? How to initialize a Metal device.</div></div>
layout: post
---
The [__Metal__](https://developer.apple.com/metal/)  framework that was announced at [__WWDC 2014__](https://developer.apple.com/videos/play/wwdc2014-603/) for `iOS` and at `WWDC 2015` also for `OS X` and `tvOS`. `Metal` is an interface for programming the `Graphics Processing Unit (GPU)` in your computer. The main advantages of using `Metal` are:

- provides the lowest overhead access to the `GPU`, hence it reduces all bottlenecks usually caused by data transferring between the `CPU` and `GPU` in other frameworks. 
- provides up to __10__ times the number of draw calls compared to `OpenGL`. `Metal`, however, is not cross-platform like `OpenGL` is, so it is not meant to be a replacement for `OpenGL`.
- allows to also run `compute` applications with performance levels comparable to similar technologies such as `CUDA` or `OpenCL`.
- has a custom shader language that allows shaders precompiling so they are a lot faster at run time. 
- has built-in memory and resource management particularized to these platforms.

Since `Metal` does not run in the `Xcode` simulator and since we cannot assume all our readers have an `iOS` device that has an __A7__ processor or newer, we will rather create an `OS X` project instead. In `Xcode` create a `Cocoa Application`. In the storyboard, drag and drop a `Label` onto the `View Controller`. Center it, enlarge it, to make sure we can display 2 lines of text. Set necessary constraints. Your storyboard should look like this: 

![alt text](https://github.com/MetalKit/images/blob/master/chapter01_1.png?raw=true "1")

Next, go to __ViewController.swift__ and create an `IBOutlet` for the label we just created. You can name it simply __label__ or anything else you wish. Finally, let's write some code. Your class should look like this:

{% highlight swift %} 
import Cocoa

class ViewController: NSViewController {

    @IBOutlet weak var label: NSTextField!
    
    override func viewDidLoad() {
        super.viewDidLoad()

        let devices = MTLCopyAllDevices()
        guard let _ = devices.first else {
            fatalError("Your GPU does not support Metal!")
        }
        label.stringValue = "Your system has the following GPU(s):\n"
        for device in devices {
            label.stringValue += "\(device.name!)\n"
        }
    }
}
{% endhighlight %}

Let's go over the code above. First we need to `import Metal` because we are calling the __MTLCopyAllDevices()__ function which belongs to the `Metal` framework. However, since `Cocoa` already imports `Metal` and the `AppKit` framework which lets us use classes such as `NSViewController`, we don't need another import line just for `Metal`. 

Then, inside __viewDidLoad()__ is where all the magic happens. We create a `Metal` device by calling `MTLCopyAllDevices()` and then we simply query for its name so we can display it as the label text. Note that `MTLCopyAllDevices()` is only available in `OS X`. For `iOS/tvOS` devices use `MTLCreateSystemDefaultDevice()` instead. A `device` is an abstraction of the `GPU` and provides us a few methods and properties, such as __name__ which we used above.

If you run the project, you should be able to see the following output:

![alt text](https://github.com/MetalKit/images/blob/master/chapter01_2.png?raw=true "2")

There is not much to see, but for now you learned how to "talk" to the `GPU` at the lowest possible level. I want to thank [@warrenm](https://twitter.com/warrenm) without whose guidance and inspiration these tutorials would not have existed. In his book, [Metal by Example](https://gum.co/metalbyexample), you can find a lot of high quality `Metal` projects written in Objective-C. The [source code](https://github.com/MetalKit/metal) for this article is posted on Github.

Until next time!
