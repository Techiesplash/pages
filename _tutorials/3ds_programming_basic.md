---
title: Basic 3DS Homebrewing
description: Basics of how to use open-source tools to make C++ 3DS homebrew.
layout: tutorial
---
### <i class="bi bi-exclamation-triangle"></i> Fair warning, this is pretty old. Should still work.
#### I was also a lot more inexperienced when I wrote this.
---
##### Table of Contents
- Essentials
- - [Hello World](#hello-world)
- - [Accessing ROM files with RomFS](#accessing-rom-files-with-romfs)
- - [Touchscreen Input](#touchscreen-input)
- - [Multithreading](#multithreading)
- Rendering

---
### Notice
These are for use with the DevKitPRO toolchain and LibCtru library (which is distributed with it) -
Please see that for up to date install instructions.

---
# Hello World

'Hello World!' is a very simple program used for checking a program's build system, and showing its basic syntax.
We should do both before going forward. You should have your project set up for this.

And if you are using VSCode, I would recommend you get the include paths set up.
## Begin coding

Start by including 'stdio.h' (or 'cstdio'!) and '3ds.h'.
A quick explanation of '3ds.h'

'3ds.h' is the file containing references to all the VITAL things we can use to program on the system.

It does not include many features, such as (And certainly not limited to) rendering capabilities, aside from running the console)
## Main loop

Start by setting up a main function as usual for C/C++.

To initialize the graphics, we need to run gfxInitDefault().

To initialize the console, we need to run consoleInit([SCREEN HERE], NULL).

Replace '[SCREEN HERE]' with either GFX_TOP or GFX_BOTTOM, depending on what screen you want.

At this point, the code should look something like this:

```cpp
#include <stdio.h>
#include <3ds.h>

int main()
{
    // Initialize the console
    gfxInitDefault();
    consoleInit(GFX_BOTTOM, NULL);
    
    return 0;
}
```

As the console is now ready, you can now print to the screen. You can use puts(), printf(), std::cout, etc.

If we try and run it now, it should immediately close itself, as there is no main loop.

To create a main loop, we create a while loop with aptMainLoop() as its argument:

```cpp
int main()
{
    // Initialize the console
    gfxInitDefault();
    consoleInit(GFX_BOTTOM, NULL);
    puts("Hello, World!");
    
    while(aptMainLoop())
    {
    
    }
    
    return 0;
}
```
## Input

Okay, so we now have a main loop... How do we exit back to the Homebrew Launcher?

Simple. We can scan for input, and break if it detects a button, in this case START, being pressed.

We can scan for input with hidScanInput(), and get which keys are down with hidKeysDown() like so:

```cpp
    // Infinite loop until the app should be closed.
    while(aptMainLoop())
    {
        // Search for inputs.
        hidScanInputs();
        
        // Store inputs in an unsigned 32-bit integer
        u32 kDown = hidKeysDown();
    }
```

Every button is mapped to a different bit on the unsigned 32-bit integer.

We can get specific keys via an AND operation between the integer and the desired key:

~~~ cpp
        // Store inputs in an unsigned 32-bit integer
        u32 kDown = hidKeysDown();
        
        // Exit the loop if START is pressed, thus exiting.
        if(kDown & KEY_START)
            break;
~~~
## Wrapping up

We can add V-Sync with gspWaitForVBlank(), which may help if you want that and don't want to draw anything aside from the console. You can disregard this if you want:

```cpp
        if(kDown & KEY_START)
            break;
            
        // V-Sync
        gspWaitForVBlank();
```

And we are done. You can run the program, and get the words 'Hello, World!' printed on the screen:

![Hello, World!](/images/devkitpro_helloworld_progress_5.png)

---
# Accessing ROM files with RomFS

RomFS allows us to access files stored on the ROM file, cartridge, or application.

It can be used to read things such as graphics assets, audio data, and other data.

## RomFS folder

You may have noticed while building that there is a folder named 'romfs' in your project:
- Project
- - build
- - ***romfs***
- - source
- - icon.png
- - ...

(If it is not already present, create it now.)

This is where all the program files are stored. These files are mounted on 'romfs:/' when the app is running.

If you have any sprites, they will appear in the 'gfx' directory inside of 'romfs:/', and are accessed accordingly.

Create a text file inside of 'romfs', name it something like 'sample.txt' (or 'sample' if you have file extensions off):

- Projects
- - ...
- - romfs
- - - sample.txt

This file will appear as 'romfs:/sample.txt' in the program.

## Using RomFS

We will initialize RomFS, check if a file exists, and then deinitialize.

### Initializing

Initializing RomFS is trivial - It only takes one simple function to do it:

```cpp
    // Initialize RomFS
    romfsInit();

    while(aptMainLoop())
    {
        // ...
```

### Using

To access a file, we do basically the exact same thing we could do on a home computer in C or C++, just with a different drive.

We will check if the file exists, first by trying to open it, and checking if it failed to open. This is what you would normally do in C:

```cpp
        u32 kDown = hidKeysDown();
        
        // Try to open the file for reading
        FILE* file = fopen("romfs:/sample.txt", "r");
        
        if(!file)
            puts("Could not find the sample file!");
        else
        {
            puts("Found the sample file.");
            
            // Close the file
            fclose(file);
        }
        
        if(kDown & KEY_START)
            break;
        // ...
```

You can play with this by adding or removing sample.txt from the 'romfs' folder.

### Deinitializing

Like initializing RomFS, it only takes another simple function to deinitialize:

```cpp
   romfsExit();
        
   return 0;
```

And that is all you need to access files on the ROM.

## Wrapping up

Our code should now look like this:

```cpp
#include <stdio.h>
#include <3ds.h>

int main()
{
    gfxInitDefault();
    consoleInit(GFX_BOTTOM, NULL);
    
    romfsInit();
    
    while(aptMainLoop())
    {
        u32 kDown = hidKeysDown();
        
        // Try to open the file for reading
        FILE* file = fopen("romfs:/sample.txt", "r");
        
        if(!file)
            puts("Could not find the sample file!");
        else
        {
            puts("Found the sample file.");
            
            // Close the file
            fclose(file);
        }
        
        if(kDown & KEY_START)
            break;
    }
    
    romfsExit();
    
    return 0;
}
```

---
# Touchscreen Input

Here we will find the position of a touch on the touchscreen.

It is very simple to do this.
## Getting the touch position

To find the touch position, we need to request the touchPosition type from the OS, and read from that.

We must first create a touchPosition like so:

```cpp
        hidScanInput();
        u32 kDown = hidKeysDown();
        
        // Will contain the position of the touch
        touchPosition touch;
        
        hidTouchRead(&touch);
        
        if(kDown & KEY_START)
            break; 
        //...
```

The touch position is stored in our touchPosition as 'px' and 'py'.
## Printing the information to the screen

To print the position so we may see it, we should, instead of spamming the console with it, overwrite the X and Y position output by setting the position of the output first.

This would make it much easier to read, although we would need some whitespace after it to prevent anything getting printed without overwriting:

```cpp
        // Print the touchscreen coordinates
        // Keep the whitespace if you do not want issues.
        printf(\x1b[1;0H X=%u Y=%u      ", touch.px, touch.py);
        
        if(kDown & KEY_START)
            break; 
        //...
```


The position should default to [0, 0] when no touch is found:

![Failed to load image](/images/devkitpro_touchscreen_progress_demo.png)

## Wrapping up

Our code should now look like this:
```cpp
#include <stdio.h>
#include <3ds.h>

int main()
{
    gfxInitDefault();
    consoleInit(GFX_BOTTOM, NULL);
    
    romfsInit();
    
    while(aptMainLoop())
    {
        hidScanInput();
        u32 kDown = hidKeysDown();
        
        touchPosition touch;
        hidTouchRead(&touch);
        
        printf(\x1b[1;0H X=%u Y=%u      ", touch.px, touch.py);
        
        
        if(kDown & KEY_START)
            break;
    }
    
    romfsExit();
    
    return 0;
}
```

---
# Multithreading

Multithreading is the process of running multiple things concurrently within a single program.

With this, you can distribute the programs workload, improving performance.

An example - a program could use one thread for graphics, and one for logic. The graphical thread would not have to deal with the logic, and so on.
## Setting up a thread
### Thread function

A thread would call another function, similar to the main() function.

Create another function like this right above main():

```cpp
void threadMain(void* arg)
{

}
```

Inside the threadMain() function, we need a loop to control when the thread stops.

```cpp
// Thread stop control
bool runThreads = true;

void threadMain(void* arg)
{
    // Thread stops when runThreads is false
    while(runThreads)
    {
    
    }
}
```
### Adding a delay

Threads can be paused for a specified amount of nanoseconds with svcSleepThread(). In this case, the thread will run every one second, printing "Hello from threadMain()!":

```cpp
bool runThreads = true;

void threadMain(void* arg)
{
    // How long to sleep - In nanoseconds
    u64 duration = 1000000000ULL
    while(runThreads)
    {
        // Pause the thread for the specified nanoseconds
        svcSleepThread(duration);
        
        // Print feedback
        puts("Hello from threadMain()!");
    }
}
```
### Using the thread

To run our thread, we must create it in main(). It must have a higher priority (lower value) than the main() thread, otherwise problems will occur.

We can also specify the size of the stack for our thread.

The thread can be created like this, first fetching the priority of main(), and creating one lower than that set to threadMain():

```cpp
    gfxInitDefault();
    consoleInit(GFX_BOTTOM, NULL);

    // The thread we want to create
    Thread thread;

    // Get the priority of the main thread
    s32 prio = 0;
    svcGetThreadPriority(&prio, CUR_THREAD_HANDLE);

    // New threads from here must be of higher priority than the main thread
    // You can change (void*)(arg) to whatever you want, it will be passed through to the thread func.
    thread = threadCreate(threadMain, (void*)0, 4096, prio-1, -2, false);

    while(aptMainLoop())
    {
        // ...

```

The thread will now run independently of the main loop, so there is nothing more to do to set it up.
## Deinitializing a thread

We must always safely delete a thread after we are done with it. Problems may happen if you do not.

First we set runThreads to false, stopping the threads safely.

Then, we deinitialize the thread:
```cpp
// Stop our thread
runThreads = false;

// Deinitialize our thread
threadJoin(thread, U64_MAX);
threadFree(thread);

return 0;
```
