# Threads

*by [Arturo Castro](http://arturocastro.net)*

*corrections by Brannon Dorsey*

## What's a thread and when to use it

Sometimes in an application we need to execute tasks that will take a while to finish. The perfect example is reading something from disk. In the computer the CPU is way faster than accessing the memory which is way faster than accessing the hard disk. So accessing, for example, an image from the HD can take a while compared to the normal flow of the application.

In openFrameworks, and in general, usually when working with openGL, our application will run in an infinite loop calling update/draw every cycle of the loop. If we have vertical sync enabled, and our screens works at 60Hz, each of those cycles will last around 16ms (1s/(60frames/s))*1000(ms/s). Loading an image from disk can take way more than those 16ms, so if we try to load an image from our update method, for example, we'll notice a pause in our animation.

To solve this we usually use threads. Threads are a way of executing certain tasks inside an application outside of the main flow. That way we can run more than one task at once so things that are slow don't stop the main flow of the application. We can also use threads to accelerate tasks by dividing them in several smaller tasks and running each of those at the same time. You can think of a thread as a subprogram inside your program.

Every application has at least 1 thread. In openFrameworks, that thread is where the setup/update/draw loop happens. We'll call this the main (or openGL) thread. But we can create more threads and each of them will run separately from the others.

So if we want to load an image in the middle of our application, instead of loading our image in update, we can create a thread that loads the image for us. The problem with this is that once we create a thread, the main thread doesn't know when it has finished, so we need to be able to communicate the results from our auxiliary thread to the main one. There's also problems that might arrise from different threads accessing the same areas in memory. We'll need some mechanisms to synchronize the access to shared memory between 2 or more threads.

First let's see how to create a thread in openFrameworks.

## ofThread

Every application has at least one thread, the main thread (also called the GL thread), when it's using openGL.

But as we've said we can create auxiliary threads to do certain tasks that would take too long to run in the main thread. In openFrameworks we can do that using the ofThread class. ofThread is not meant to be used directly, instead we inherit from it and implement a `threadedFunction` which will later get called from the auxiliary thread once we start it:

```cpp
class ImageLoader: public ofThread{
    void setup(string imagePath){
        this->path = imagePath;
    }

    void threadedFunction(){
        ofLoadImage(image, path);
    }

    ofPixels image;
    string path;
}

//ofApp.h

ImageLoader imgLoader;


// ofApp.cpp
void ofApp::keyPressed(int key){
    imgLoader.setup("someimage.png");
    imgLoader.startThread();
}
```

When we call `startThread()`, `ofThread` starts a new thread and returns immediately, that thread will call our `threadedFunction` and will finish when the function ends.

This way the loading of the image happens simultaneously to our update/draw loop and our application doesn't stop while waiting till the image is loaded from disk.

Now, how do we know when our image is loaded? The thread will run separately from the main thread of our application:

![Simple Thread](images/simple_thread.png "Simple Thread")

As we see in the image the duration of loading of the image and thus the duration of the call to threadedFunction is not automatically known to the main thread. Since all our thread does is load the image, we can check if the thread has finished running which will tell us that the image has loaded. For that ofThread has a method: `isThreadRunning()`:

```cpp
class ImageLoader: public ofThread{
    void setup(string imagePath){
        this->path = imagePath;
    }

    void threadedFunction(){
        ofLoadImage(image, path);
    }

    ofPixels image;
    string path;
}

//ofApp.h
bool loading;
ImageLoader imgLoader;
ofImage img;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    if(loading==true && !imgLoader.isThreadRunning()){
        img.getPixelsRef() = imgLoader.image;
        img.update();
        loading = false;
    }
}

void ofApp::draw(){
    if (img.isAllocated()) {
        img.draw(0, 0);
    }
}

void ofApp::keyPressed(int key){
    if(!loading){
        imgLoader.setup("someimage.png");
        loading = true;
        imgLoader.startThread();
    }
}
```


Now as you can see we can only load a new image when the first one has finished loading. What if we want to load more than one? A possible solution would be to start a new thread and ask it if it's been loaded already:

```cpp
class ImageLoader: public ofThread{
    ImageLoader(){
        loading = false;
    }

    void load(string imagePath){
        this->path = imagePath;
        loading = true;
        startThread();
    }

    void threadedFunction(){
        ofLoadImage(image,path);
        loaded = true;
    }

    ofPixels image;
    string path;
    bool loading;
    bool loaded;
}

//ofApp.h
vector<unique_ptr<ImageLoader>> imgLoaders;
vector<ofImage> imgs;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    for(int i=0;i<imgLoaders.size();i++){
        if(imgLoaders[i].loaded){
            if(imgs.size()<=i) imgs.resize(i+1);

            imgs[i].getPixelsRef() = imgLoaders[i].image;
            imgs[i].update();
            imgLoaders[i].loaded = false;
        }
    }
}

void ofApp::draw(){
    for(int i=0;i<imgLoaders.size();i++){
        imgs[i].draw(x,y);
    }
}

void ofApp::keyPressed(int key){
    imgLoaders.push_back(move(unique_ptr<ImageLoader>(new ImageLoader)));
    imgLoaders.back()->load("someimage.png");
}
```

Another possibility would be to use 1 thread only. To do that a possible solution would be to use a queue in our loading thread whenever we want to load a new image. To do this we insert it's path in the queue and when the threadedFunction finishes loading one image it checks the queue. If there's a new image it loads it and it is removed from the queue.

The problem with this is that we will be trying to access the queue from 2 different threads, and as we've mentioned in the memory chapter, when we add or remove elements to a memory structure there's the possibility that the memory will be moved somewhere else. If that happens while one thread is trying to access it we can easily end up with a dangling pointer that will cause the application to crash. Imagine the next sequence of instruction calls from the 2 different threads:

        loader thread: finished loading an image
        loader thread: pos = get memory address of next element to load
        main thread:   add new element in the queue
        main thread:   queue moves in memory to an area with enough space to allocate it
        loader thread: try to read element in pos  <- crash pos is no longer a valid memory address

At this point we might be accessing a memory address that doesn't contain a string anymore, or even trying to access a memory address that is outside of the memory assigned to our application. In this case the OS will kill it sending a segmentation fault signal as we've seen in the memory chapter.

The reason this happens is that since thread 1 and 2 run simultaneously we don't know in which order their instructions area going to get executed. We need a way to ensure that thread 1 cannot access the queue while thread 2 is modifying it and viceversa. For that we'll use some kind of lock: In C++ usually a mutex, in openFrameworks an ofMutex.

But before seeing mutexes, let's see briefly some particulars of using thread while using openGL.

## Threads and openGL

You might have noticed in the previous examples:

```cpp
class ImageLoader: public ofThread{
    ImageLoader(){
        loaded = false;
    }
    void setup(string imagePath){
        this->path = imagePath;
    }

    void threadedFunction(){
        ofLoadImage(image, path);
        loaded = true;
    }

    ofPixels image;
    string path;
    bool loaded;
}

//ofApp.h
ImageLoader imgLoader;
ofImage img;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    if(imgLoader.loaded){
        img.getPixelsRef() = imgLoader.image;
        img.update();
        imgLoader.loaded = false;
    }
}

void ofApp::draw(){
    if (img.isAllocated()) {
        img.draw(0, 0);
    }
}

void ofApp::keyPressed(int key){
    if(!loading){
        imgLoader.setup("someimage.png");
        imgLoader.startThread();
    }
}
```


Instead of using an ofImage to load images, we are using an ofPixels and then in the main thread we use an ofImage to put the contents of ofPixels into it. This is done because openGL, in principle, can only work with 1 thread. That's why we call our main thread the GL thread.

As we mentioned in the advanced graphics chapter and other parts of this book, openGL works asynchronously in some kind of client/server model. Our application is the client sending data and drawing instructions to the openGL server which will send them to the graphics card in it's own times.

Because of that, openGL knows how to work with one thread, the main thread from which the openGL context was created. But if we try to do openGL calls from a different thread we will most surely crash the application, or at least not get the desired results.

When we call `img.loadImage(path)` on an ofImage, it'll actually do some openGL calls, mainly create a texture and upload to it the contents of the image. If we did that from a thread that isn't the GL thread, our application will probably crash or just don't load the texture properly.

There's a way to tell ofImage, and most other objects that contain pixels and textures in openFrameworks, to not use those textures and instead work only with pixels. That way we could use an ofImage to load the images to pixels and later in the main thread activate the textures to be able to draw the images:

```cpp
class ImageLoader: public ofThread{
    ImageLoader(){
        loaded = false;
    }
    void setup(string imagePath){
        image.setUseTexture(false);
        this->path = imagePath;
    }

    void threadedFunction(){
        image.loadImage(path);
        loaded = true;
    }

    ofImage image;
    string path;
    bool loaded;
}

//ofApp.h
ImageLoader imgLoader;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    if(imgLoader.loaded){
        imgLoader.image.setUseTexture(true);
        imgLoader.image.update();
        imgLoader.loaded = false;
    }
}

void ofApp::draw(){
    if (imgLoader.image.isAllocated()){
        imgLoader.image.draw(0,0);
    }
}

void ofApp::keyPressed(int key){
    if(!loading){
        imgLoader.setup("someimage.png");
        imgLoader.startThread();
    }
}
```

There are ways to use openGL from different threads, for example creating a shared context to upload textures in a different thread or using PBO's to map a memory area and later upload to that memory area from a different thread but that's out of the scope of this chapter. In general remember that accessing openGL outside of the GL thread is not safe. In openFrameworks you should only do operations that involve openGL calls from the main thread, that is, from the calls that happen in the setup/update/draw loop, the key and mouse events, and the related ofEvents. If you start a thread and call a function or notify an ofEvent from it, that call will also happen in the auxiliary thread, so be careful to not do any GL calls from there.

A very specific case is sound, sound APIs in openFrameworks, in particular ofSoundStream, create their own threads since sound's timing needs to be super precise. So when working with ofSoundStream be careful not to use any openGL calls and in general apply the same logic as if you where inside the threadedFunction of an ofThread. We'll see more about this in the next sections.
