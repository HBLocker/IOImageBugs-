Though this being a somewhat unsuccessful write up for the discovery of finding a real bug in ImageIO, I still want to do a write up about the journey as it was complicated and fun, plus I have not written a blogpost in a long time. Remeber before you fuzz or do any research, remember everything is up to date!

## Setting the Scene 

After reading a lot of Project0 blog posts in particular the [following](https://googleprojectzero.blogspot.com/2020/04/fuzzing-imageio.html) I decided to give it a go myself. Can an image be read in to memory and trigger a bug? 
After extracting the cache and reversing IOimage in Ida pro there are many undomcumeted file types which this suports, for this write up I chose:


```bash
__cstring:000000018A177779	00000010	C	org.khronos.ktx
__cstring:000000018A177789	00000011	C	org.khronos.ktx2 
```
I  decided a texture file as I made a presumption that it being a texture file it would have some complexity to to it. 


### Understanding the target

A texture file, in the context of computer graphics, is an image or a collection of images used to add surface detail, color, or visual effects to 3D models in computer graphics and video games. Textures are applied to 3D objects to make them look more realistic

- Khronos Group Involvement: The Khronos Group, known for developing open standards in the graphics industry, took on the task of creating a standardized texture format.
  This effort involved collaboration from various companies and developers in the graphics community.

- Creation of KTX Format: KTX files were introduced as a solution. They are designed to be compact, easily parsed, and contain all the necessary information for efficient texture  loading. KTX files support a wide range of texture types, including 2D textures, cubemaps, and 3D textures.

The file follows the following structure:
```
[KTX Identifier] 
[Endianness] 
[Metadata] 
[BytesOfKeyValueData] 
[Key-Value Data]
[Texture Data]
```
Which is then read in by IOimage once it has been passed to it. 

Io image first idenfiies the fule type by the inner identifier at the top once that bound has been passed the rest of the data is read into the the libary spesific functions. 

After then searching for the ktf file name in ida, we can see all the functions used to render this image type , there are rougly 199 diffeent funcitons used for the ktx type:
here are a ferw examples:
```
CreateReader_KTX(void)	__text	
CreateReader_KTX2(void)	__text	
CreateReader_KTX2_ASTC(void)	__text	
CreateReader_KTX2_BC(void)	__text	
CreateReader_KTX2_ETC(void)	__text	
CreateReader_KTX2_PVR(void)	__text	
CreateReader_KTX_ASTC(void)	__text	
CreateReader_KTX_BC(void)	__text	
CreateReader_KTX_ETC(void)	__text	
CreateReader_KTX_PVR(void)	__text	
CreateWriter_KTX(void)	__text	
CreateWriter_KTX2(void)	__text	
GetKtx2FormatInfo(VkFormat,Ktx2FormatInfo *&)
IIO_Reader_KTX2::canDecodeOOP(uint)	__text	
IIO_Reader_KTX2::createReadPlugin(CGImagePlus *,uint,long long)	
IIO_Reader_KTX2::createReadPlugin(ReadPluginInitData *)	
IIO_Reader_KTX2::getImageCount(IIOImageReadSession *,IIODictionary *,CGImageSourceStatus *,uint *)	
IIO_Reader_KTX2::hasCustomImageCountProc(void)	
```

Next is time to obtain sample images after some research and more time readin about this file, I stumbled upon a formum from 2003 which had a few texrture files for the sims, as a user did not like the teture for the grass. As an obscure and neich this issue this is to someone, I am glad though they re made the grass texture as I now have seed files for the harness. 

After writing a harness, this was before this as added in a a sample harness within [Jackalope](https://github.com/googleprojectzero/Jackalope/commit/ce04e4f723126b287da7e3a89db2a9cb28726a5b)  which I used, I then began to fuzz the target and left it for a few days. I used shem as it is quicker and I wanted to get as much coverage and mutations as quick as possible. 

Then, alas a bug! one crash found, now time to triarge and see what it is:
```bash
==83024==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x000102c05bb4 at pc 0x0001003f4088 bp 0x00016fdfe690 sp 0x00016fdfde50
READ of size 1 at 0x000102c05bb4 thread T0
    #0 0x1003f4084 in wrap_strlen+0x164 (libclang_rt.asan_osx_dynamic.dylib:arm64e+0x18084) (BuildId: 
allocated by thread T0 here:
    #0 0x10041ee68 in wrap_malloc+0x94 (libclang_rt.asan_osx_dynamic.dylib:arm64e+0x42e68) (BuildId: f0a7ac5c49bc3abc851181b6f92b308a32000000200000000100000000000b00)
    #1 0x18cadbe74 in ktxTexture2_constructFromStreamAndHeader+0x2d4 (ImageIO:arm64e+0x2c7e74) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #2 0x18cadcecc in ktxTexture2_CreateFromStream+0x9c (ImageIO:arm64e+0x2c8ecc) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #3 0x18c8de9e0 in ETCReadPlugin::initialize(IIODictionary*)+0x2a0 (ImageIO:arm64e+0xca9e0) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #4 0x18c824a80 in IIOReadPlugin::callInitialize()+0x84 (ImageIO:arm64e+0x10a80) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #5 0x18c824794 in IIO_Reader::initImageAtOffset(CGImagePlugin*, unsigned long, unsigned long, unsigned long)+0x68 (ImageIO:arm64e+0x10794) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #6 0x18c82202c in IIOImageSource::makeImagePlus(unsigned long, IIODictionary*)+0x324 (ImageIO:arm64e+0xe02c) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #7 0x18c821908 in IIOImageSource::getPropertiesAtIndexInternal(unsigned long, IIODictionary*)+0x44 (ImageIO:arm64e+0xd908) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #8 0x18c821844 in IIOImageSource::copyPropertiesAtIndex(unsigned long, IIODictionary*)+0x14 (ImageIO:arm64e+0xd844) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #9 0x18c82174c in CGImageSourceCopyPropertiesAtIndex+0x104 (ImageIO:arm64e+0xd74c) (BuildId: 9bd21afa455c3aeaba79392385098d3132000000200000000100000000000d00)
    #10 0x18618f02c in ImageSourceOptionsForCGImageSource_index_+0x3c (AppKit:arm64e+0x12802c) (BuildId: af9f689170ad3c26af08b747344892d232000000200000000100000000000d00)
    #11 0x18618eeb0 in +[NSBitmapImageRep _imagesWithData:hfsFileType:extension:zone:expandImageContentNow:includeAllReps:]+0x1a8 (AppKit:arm64e+0x127eb0) (BuildId: af9f689170ad3c26af08b747344892d232000000200000000100000000000d00)
    #12 0x18618ece0 in +[NSBitmapImageRep _imageRepsWithData:hfsFileType:extension:expandImageContentNow:]+0x54 (AppKit:arm64e+0x127ce0) (BuildId: af9f689170ad3c26af08b747344892d232000000200000000100000000000d00)
    #13 0x18618e7e4 in +[NSImageRep _imageRepsWithContentsOfURL:expandImageContentNow:giveUpOnNetworkURLsWithoutGoodExtensions:]+0x2e0 (AppKit:arm64e+0x1277e4) (BuildId: af9f689170ad3c26af08b747344892d232000000200000000100000000000d00)
    #14 0x186241700 in -[NSImage initWithContentsOfURL:]+0x1c (AppKit:arm64e+0x1da700) (BuildId: af9f689170ad3c26af08b747344892d232000000200000000100000000000d00)
    #15 0x100003674 in main+0x1f0 (load_image_:arm64+0x100003674) (BuildId: 83dc9edf47973cf0a4ae93cee91d216132000000200000000100000000000d00)
    #16 0x182a3fe4c  (<unknown module>)

SUMMARY: AddressSanitizer: heap-buffer-overflow (libclang_rt.asan_osx_dynamic.dylib:arm64e+0x18084) (BuildId: f0a7ac5c49bc3abc851181b6f92b308a32000000200000000100000000000b00) in wrap_strlen+0x164
Shadow bytes around the buggy address:
  0x0070205a0b20: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0070205a0b30: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0070205a0b40: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0070205a0b50: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0070205a0b60: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x0070205a0b70: fa fa fa fa fa fa[01]fa fa fa 00 fa fa fa fd fd
  0x0070205a0b80: fa fa fd fd fa fa fd fa fa fa fd fa fa fa 05 fa
  0x0070205a0b90: fa fa 00 01 fa fa 06 fa fa fa 00 03 fa fa 07 fa
  0x0070205a0ba0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0070205a0bb0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0070205a0bc0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==83024==ABORTING
```

Then realizing that I did not update my machine and I was an out of date version. I then updated my machine to find it had been patched, as I forgot to update my system before fuzzing ImageIO.
But the texture package is buggy! Though the heap overflow is cool.

I decided to use the crash files as new seed data which can be passed back ito the harness, this was an attempt to gain better coverage and hopefully hit some nice edge cases. 

Then with runing the fuzzer again, I then got a kind of bug:

```bash
==1496==ERROR: AddressSanitizer: requested allocation size 0xb8c839133e02a0c2 (0xb8c839133e02b0c8 after adjustments for alignment, red zones etc.) exceeds maximum supported size of 0x10000000000 (thread T0)
    #0 0x1003e2e68
```
This is a kind of bug but not really as it is bad memory allocation that cant really be expoited - Anyway though lets run though what causes the issue. 

The asan logs showed the following was casing the bad memory allocation:

```bash
#0 0x1003e2e68 in wrap_malloc+0x94 (libclang_rt.asan_osx_dynamic.dylib:arm64e+0x42e68)
#1 0x19c83998c in ktxTexture2_constructFromStreamAndHeader+0x3bc (ImageIO:arm64e+0x28d98c)
#2 0x19c83a8e4 in ktxTexture2_CreateFromStream+0x9c (ImageIO:arm64e+0x28e8e4)
#3 0x19c679870 in ETCReadPlugin::initialize(IIODictionary*)+0x2a0 (ImageIO:arm64e+0xcd870)
#4 0x19c5bccf4 in IIOReadPlugin::callInitialize()+0x84 (ImageIO:arm64e+0x10cf4)
#5 0x19c5bca08 in IIO_Reader::initImageAtOffset(CGImagePlugin*, unsigned long, unsigned long, unsigned long)+0x78 (ImageIO:arm64e+0x10a08)
```


Here is a really simplified version of the code as it is rather long, this is some of it commented out and reordered so it is easier to follow
and understand the function ktxTexture2_constructFromStreamAndHeader. 

```cpp
_int64 __fastcall ktxTexture2_constructFromStreamAndHeader(__int64 a1, __int64 a2, __int64 a3, char a4)
{
    __int64 ImageData; // Result variable
    int v8; // Header size variable
    __int64 v43; // Error code variable
    __int64 v44; // Memory address variable for free
    // Variables for handling dynamic memory allocation and data processing
    __int64 v12, v13, v14, v15, v16, v17, v18, v19, v20, v21, v22, v23, v24, v25, v26, v27, v28, v29, v30, v31, v32, v33, v34, v35, v41, v42;

    // Initialize memory to zero
    v44 = *MEMORY[0x1D6AED6D0];
    v43 = 0;
    *(_OWORD *)a1 = 0u;
    // ... (initialize more memory locations to zero)

    // Call ktxTexture_constructFromStream function
    ImageData = ktxTexture_constructFromStream();
    if ((_DWORD)ImageData)
        return ImageData;

    // Check header and handle errors
    ImageData = ktxCheckHeader2_(a3, &v43);
    if (!(_DWORD)ImageData)
    {
        // Extract header information
        v8 = *(_DWORD *)(a3 + 40);

        // Initialize ktxTexture2 structure
        *(_DWORD *)a1 = 2;
        *(_QWORD *)(a1 + 8) = ktxTexture2_vtbl;
        // ... (initialize more fields)

        // Allocate memory for texture data
        v11 = 24LL * (unsigned int)(v8 - 1) + 56;
        v12 = malloc(v11);
        *(_QWORD *)(a1 + 160) = v12;
        if (!v12)
            goto LABEL_44;

        // Initialize texture data
        v13 = v12;
        bzero(v12, v11);
        // ... (populate texture data)

        // Process texture data
        switch (HIWORD(v43))
        {
            case 3:
                // Handle case 3
                v17 = *(_QWORD *)(a3 + 24);
                break;
            case 2:
                // Handle case 2
                *(_DWORD *)(a1 + 40) = *(_DWORD *)(a3 + 24);
                *(_DWORD *)(a1 + 44) = 1;
                // ... (initialize more fields)
                break;
            default:
                // Handle default case
                *(_QWORD *)(a1 + 40) = 0x100000001LL;
                // ... (initialize more fields)
                break;
        }

        // Handle remaining cases and data processing

        // Clean up resources in case of errors
        LABEL_44:
            ImageData = 13LL;
        }

        // Clean up allocated memory
        LABEL_45:
            v38 = *(void **)(a1 + 128);
            if (v38)
                free(v38);
            v39 = *(void ***)(a1 + 160);
            if (v39)
            {
                if (*v39)
                {
                    free(*v39);
                    v39 = *(void ***)(a1 + 160);
                }
                free(v39);
            }
            ktxTexture_destruct(a1);
            return ImageData;
        }
}
```

### Bug overview 

Upon review we can see the malloc function here:
```cpp
v11 = 24LL * (unsigned int)(v8 - 1) + 56;
```
Here, v11 is calculated as a product of 24 and (v8 - 1), to which 56 is added. v8 is derived from a header value 
(*(_DWORD *)(a3 + 40)) and is assumed to be an integer. If v8 is an unexpectedly large value, this calculation can result in an excessively large v11.

```cpp
v12 = malloc(v11);
*(_QWORD *)(a1 + 160) = v12;
```

Malloc is then called to allocate memory for v12, and the pointer to this allocated memory is stored in *(_QWORD *)(a1 + 160). 
The size of the allocated memory, v11, is determined by the earlier calculation.
  

- Subtraction and Type Cast: (v8 - 1) becomes (-100000 - 1), which equals -100001.

-  Type Cast to unsigned int: When you cast -100001 to an unsigned int, it wraps around due to the conversion rules.
  This  will become a large positive value because the negative number is represented using two's complement representation. So, (unsigned int)(-100001) becomes 4294867295 on a typical system with 32-bit integers.
    Multiplication: 24 * 4294867295 results in 102877698280, which is already an incredibly large number.

- Addition: Adding 56 to 102877698280 results in 102877698336.

In this example, due to the large negative value of v8, the calculation leads to an allocation size of 102877698336 bytes. Attempting to allocate such a huge chunk of memory would likely fail and lead to a system error or, in this case, trigger an AddressSanitizer error due to the excessively large allocation size.



