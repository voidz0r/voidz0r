---
title: "Diving into the DICOM file format"
slug: "diving-into-the-dicom-file-format"
date: 2024-02-06
weight: 2
images: ["logo/void.png"]
---

## Introduction
While doing some research about file formats, I occasionally found some references about the DICOM file format. Since I never heard about it I've decided to dig deeper.

### 16/02/2024 Update
It seems that the developer was already aware of the Double-Free issue since it has been reported a few weeks ago and it's ready to be merged into the main branch. The issue has been reported in the following [Github issue - Double Free](https://github.com/ImagingDataCommons/libdicom/issues/82) and [Github Issue - Floating point](https://github.com/ImagingDataCommons/libdicom/issues/83).

## The format
DICOM, acronym for Digital Imaging and Communications in Medicine, is both a communication protocol and a file format. As the name suggests, the format is widely used by a series of medical devices such as radiography, magnetic resonance imaging and radiation therapy. The protocol itself has been release in 1985 by American College of Radiology (ACR) and National Electrical Manufacturers Association (NEMA). The communication part of the protocol was already covered by Sina Yazdanmehr during Black Hat 2023. Following the link to [his research](https://i.blackhat.com/EU-23/Presentations/EU-23-Yazdanmehr-Millions_of_Patient_Records_at_Risk.pdf)

The main principle behind the format is to group information into data sets which will contain information about the patient such as his ID, first name and last name, and associated image(s). The file format also acts like an encapsulator since it supports several other standards such as JPEG, GIF, PNG and so on.

A typical DICOM file would look like the following:
```
   Free text          Magic number
┌──────────────┬──────────────────────┬─────────────────────┐
│              │                      │                     │
│    header    │   \x44\x49\x43\x4d   │    data elements    │
│              │        "DICM"        │                     │
│              │                      │                     │
│              │                      │                     │
│  1024 bits   │       4 bytes        │                     │
└──────────────┴──────────────────────┴─────────────────────┘
```
Each subsequent data element will have a Value Representation, which can be expressed as "Implicit" or "Explicit", depending if the specific VR is supplied or not. Explicit VR needs to be represented differently depending on the supplied VR value.


```
Data Element with explicit VR of AE, AS, AT, CS, DA, DS, DT, FL, FD, IS, LO, LT, PN, SH, SL, SS, ST, TM, UI, UL and US
┌────────────────────────────────────┬──────────────────────┬────────────────┬─────────────────┐
│               TAG                  │         VR           │   Value length │      Value      │
├─────────────────┬──────────────────┼──────────────────────┼────────────────┼─────────────────┤
│   group number  │  element number  │         VR           │     Length     │ Even number of  │
│                 │                  │ Value Representation │                │ bytes encoded   │
│                 │                  │                      │                │ according to VR │
│     16 bits     │      16 bits     │       16 bits        │                │                 │
│   unsigned int  │   unsigned int   │   2 byte character   │     16 bits    │                 │
└─────────────────┴──────────────────┴──────────────────────┴────────────────┴─────────────────┘
```

When a different VR from the previous ones is used, the Data element needs to be provided as following:

```
Data Element with explicit VR other than the previous ones
┌────────────────────────────────────┬───────────────────────────────────┬─────────────────┬──────────────────┐
│               TAG                  │                VR                 │   Value length  │       Value      │
├─────────────────┬──────────────────┼─────────────────────┬─────────────┼─────────────────┼──────────────────┤
│   group number  │  element number  │         VR          │  Reserved   │      Length     │ Even number of   │
│                 │                  │ Value Representation│             │                 │ bytes encoded    │
│                 │                  │                     │             │                 │ according to VR  │
│     16 bits     │      16 bits     │       16 bits       │             │                 │                  │
│   unsigned int  │   unsigned int   │   2 byte character  │  16 bits    │     32 bits     │                  │
└─────────────────┴──────────────────┴─────────────────────┴─────────────┴─────────────────┴──────────────────┘
```

While implicit Data Elements are represented as follow:

```
Implicit Data Element
┌────────────────────────────────────┬────────────────┬───────────────────┐
│               TAG                  │  Value length  │       Value       │
├─────────────────┬──────────────────┼────────────────┼───────────────────┤
│   group number  │  element number  │    Length      │  Even number of   │
│                 │                  │                │  bytes encoded    │
│                 │                  │                │  according to VR  │
│     16 bits     │      16 bits     │                │                   │
│   unsigned int  │   unsigned int   │   32 bits      │                   │
└─────────────────┴──────────────────┴────────────────┴───────────────────┘
```

Each VR will represent a different type of data like, for instance:
- AE - max 16 bytes - Application Entity
- AS - 4 bytes - Age string 
- AT - 4 bytes - Attribute
- DA - 8 bytes - The date expressed as YYYYMMDD like for example: 20240207

And so on. All the VRs can be found in the official [DICOM Standard](https://dicom.nema.org/medical/dicom/current/output/html/part05.html#table_6.2-1) documentation.

Here is a valid DICOM file viewed with the `xxd` tool:

![DICOM file](/dicom/01_xxd.png)


## Fuzzing the libdicom library
One of the many libraries that parse .dcm files is the `libdicom` library, which can be found on [Github](https://github.com/ImagingDataCommons/libdicom).

While the library uses the `meson` tool in order to build the project, AFL doesn't seems to be compatible with it. I converted the .ninja files using a tool called `ninjatool` which generates a Makefile from a ninja one. It is availabe on Github as well [ninjatool.py](https://github.com/bonzini/qemu/blob/meson-poc/scripts/ninjatool.py).

In order to fuzz the library we need an entry point for the fuzzer, which takes an input file (or stdin) and pass it to the libdicom library. The example on the official github page can be used:

```c
#include <stdlib.h>
#include <dicom/dicom.h>

#include <stdlib.h>
#include <dicom/dicom.h>
int main(int argc, char *argv[]) {
    const char *file_path = argv[1]; // We change this value to argv[1] so it can accept a filename on stdin
    DcmError *error = NULL;

    DcmFilehandle *filehandle = dcm_filehandle_create_from_file(&error, file_path);
    if (filehandle == NULL) {
        dcm_error_log(error);
        dcm_error_clear(&error);
        return 1;
    }

    const DcmDataSet *metadata =
        dcm_filehandle_get_metadata_subset(&error, filehandle);
    if (metadata == NULL) {
        dcm_error_log(error);
        dcm_error_clear(&error);
        dcm_filehandle_destroy(filehandle);
        return 1;
    }

    const char *num_frames;
    uint32_t tag = dcm_dict_tag_from_keyword("NumberOfFrames");
    DcmElement *element = dcm_dataset_get(&error, metadata, tag);
    if (element == NULL ||
        !dcm_element_get_value_string(&error, element, 0, &num_frames)) {
        dcm_error_log(error);
        dcm_error_clear(&error);
        dcm_filehandle_destroy(filehandle);
        return 1;
    }

    printf("NumerOfFrames == %s\n", num_frames);

    dcm_filehandle_destroy(filehandle);

    return 0;
}
```

After loading some .DCM files that I found on Google, I was happy to see that a lot of crashes showed up on AFL.

## Crash 1 - Floating point exception
The first crash was regarding a floating point exception and it's regarding the `get_tiles` function of the `dicom-file.c` file.


Analyzing via GDB shows the following:

![Floating Point Exception](/dicom/fp1/02_floating_point_exception.png)

And the corresponding function is:
```c
static bool get_tiles(DcmError **error,
                      const DcmDataSet *metadata,
                      uint32_t *tiles_across,
                      uint32_t *tiles_down)
{
    int64_t width;
    int64_t height;
    uint32_t frame_width;
    uint32_t frame_height;

    if (!get_frame_size(error, metadata, &frame_width, &frame_height)) {
        return false;
    }

    // TotalPixelMatrixColumns is optional and defaults to Columns, ie. one
    // frame across
    width = frame_width;
    (void) get_tag_int(NULL, metadata, "TotalPixelMatrixColumns", &width); // <--- crash here

    // TotalPixelMatrixColumns is optional and defaults to Columns, ie. one
    // frame across
    height = frame_width;
    (void) get_tag_int(NULL, metadata, "TotalPixelMatrixRows", &height);

    *tiles_across = (uint32_t) width / frame_width + !!(width % frame_width);
    *tiles_down = (uint32_t) height / frame_height + !!(height % frame_height);

    return true;
}
```

Putting a breakpoint on line `152` shows that width and height are actually zero, so the exception is because the value pointed by `tiles_across` is dividing zero by zero.

![GDB Breakpoint](/dicom/fp1/03_breakpoint.png)

The width and height comes from the call to the function `get_frame_size` which takes the reference to `uint32_t frame_width` and `uint32_t frame_height` as parameters.

```c
if (!get_tag_int(error, metadata, "Columns", &width) ||
        !get_tag_int(error, metadata, "Rows", &height)) {
        return false;
    }

    *frame_width = (uint32_t) width;
    *frame_height = (uint32_t) height;
```

But where the "Columns" and "Rows" comes from? They can be found on the `dicom-dict-tables.c`:
```c
const struct _DcmAttribute dcm_attribute_table[] = {
    // ...
    {0X00280010, DCM_VR_TAG_US, "Rows"},
    {0X00280011, DCM_VR_TAG_US, "Columns"},
    // ...
}
```
And if we search for the tag value on the payload (little endian order), we should see a `2800 1000` for Rows and `2800 1100` for Columns.

![Input file](/dicom/fp1/04_hex.png)

So in this case the "Columns" tag value is used to determine the `frame_width` and calculate the number of tiles across, leading to the floating point exception.

Note: this bug is also valid for the tag `2800 1000` or Rows/Height.

## Crash 2 - Double free on Data set headers
This one looked way more interesting from an exploitation point of view.

![Segfault](/dicom/dfree/05_segfault.png)

By looking at the debug messages, which can be activated by using the `DCM_DEBUG=1` env variable, it is obvious that the library tried to `free` a specific value twice, first on tag `0200 3030` and then on some address on the heap.

Inspecting the crash with gdb shows:

![Inspect with GDB](/dicom/dfree/07_gdb_rootcause.png)

So the library, for some reason, calls the `dcm_element_destroy` function twice. By putting a breakpoint we see:

![Break #1](/dicom/dfree/08_first_break.png)
![Break #2](/dicom/dfree/09_second_break.png)

A look at the source code shows this:

![First free](/dicom/dfree/10_firstfree.png)
![Second free](/dicom/dfree/11_secondfree.png)

So this means that whenever there are duplicated tags into the the DCM body, the `dcm_dataset_insert` fails, `free` the element and returns false. In the `parse_meta_element_create` there will be another `free` because of that boolean result, which is causing the double free. But what the message `free(): double free detected in tcache 2` means?

The message shows that the glibc protection against double free got triggered and prevented to double free the memory chunk. The tcache is a heap caching mechanism introduced from the glibc 2.26 back in 2017 and it offers some performance gains by creating per-thread caches for chunks up to a certain size. This means that it will first look into the tcache bins before traversing small, fast, large or unsorted bins, whenever a chunk is allocated or freed.

## Crash 3 - Double free on the Data set body
Almost in the same way as above, it is possible to achieve a double free by putting another element with the same tag in the body.

![Crash 1](/dicom/dfree_body/01_crash_gdb.png)

And the root cause

![Crash 2](/dicom/dfree_body/02_debug.png)


## Crash 4 - Double free on sequence elements
As specified in the documentation, DICOM format supports sequences which is a way to append multiple items into a list to achieve a nested data set.

By looking at the following crash:

![Crash - Sequences](/dicom/dfree_sequences/01_crash.png)

And putting some breakpoints:

![Crash - Sequences - Breakpoint hit #1](/dicom/dfree_sequences/02_first_free.png)

![Crash - Sequences - Breakpoint hit #2](/dicom/dfree_sequences/03_second_free.png)

We can clearly see that data is hitting the `dcm_dataset_insert` first before beying free'd on the `dcm_sequence_destroy` function. Subsequentially, another `dcm_sequence_destroy` is getting triggered, this time from the `parse_meta_sequence_end` function.

If we follow the functions in the source code we see:

![Crash - Sequences - Root cause #2](/dicom/dfree_sequences/05_firstfree.png)

![Crash - Sequences - Root cause #1](/dicom/dfree_sequences/04_rootcause.png)

If we look at the screenshots above we can understand why this is happening:
- The dataset is first parsed by the `dcm_dataset_insert` function which is responsible for parsing the dataset. If a tag name is already in the dataset, it fails with a message and `free` the element from memory
- The dataset is then parsed by the `parse_meta_sequence_end` function which calls another `free`, but this time the pointer to the heap has been free'd already by the previous function, so the pointer points to another memory area.

Until now we used what AFL generated from the input files, but how we can control each element and build our own DCM file?

## DCM Builder in the building!
While there are some libraries that parse .dcm files, like `pydicom`, I had the necessity for a tool that builds arbitrary data elements and force the library to load sequence of data in a specific order. So I've built the `dicombuilder` library.

The library is pretty simple, it uses a dictionary of all the possible values, declared in the `dicttable.py` file like the following:

![Dicombuilder dicttable](/dicom/dicom_builder/01_dicttable.png)

Along with some helper functions like `find_by_hex` or `find_by_name` to look for a specific element using the name, instead of the hex value. In the end, I was able to build a dicom file in the following way:

![Building a DICOM file](/dicom/dicom_builder/02_building_dicomfile.png)

I will upload the package to PyPi and you can install the package via `pip install -e dicombuilder .` after cloning it from ![Github](https://github.com/voidz0r/dicombuilder)


## Impact
Controlling the flow of execution in this case will be pretty hard, because of the amount of memory mitigations in place. By the time of writing this, I was using glibc 2.35. Here are some techniques that are available:

- decrypt_safe_linking
- fastbin_dup
- fastbin_dup_consolidate
- fastbin_reverse_into_tcache
- house_of_botcake
- house_of_einherjar
- house_of_lore
- house_of_mind_fastbin
- house_of_spirit
- house_of_water
- large_bin_attack
- mmap_overlapping_chunks
- overlapping_chunks
- poison_null_byte
- safe_link_double_protect
- tcache_house_of_spirit
- tcache_poisoning
- tcache_stashing_unlink_attack
- unsafe_unlink

All the above methodologies require to have a precise control about what needs to be free'd and in which order. Giving our scenario:
- We don't have control on the size. The majority of the allocations are handled in a precise way and even if we force a specific length, there are some additional checks made on each VR and pre-defined sizes
- We don't have control of the `free`/`malloc` order. The application parse the entire file in a top-down flow, starting from allocating memory regions while reading the data elements to free the memory at the end. With this in mind it's trivial, if not impossible, to force a specific `free` and `malloc` sequence.

So the result will be just a crash.

# Related impact
Some projects such as "openslide" uses libdicom library to parse .dcm files. As a result, even with programs like QuPath, the application crashes with the following error:

![QuPath Crash](/dicom/qupath.png)

Which is a heap corruption error:

![QuPath Windows error code](/dicom/qupath_error.png)