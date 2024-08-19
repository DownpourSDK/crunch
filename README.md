crunch/crnlib v1.04 - Advanced DXTn texture compression library

Public Domain - Please see license.txt. 

Portions of this software make use of public domain code originally
written by Igor Pavlov (LZMA), RYG (crn_ryg_dxt*), and Sean Barrett (stb_image.c).

If you use this software in a product, an acknowledgment in the product 
documentation would be highly appreciated but is not required.

## Overview

crnlib is a lossy texture compression library for developers that ship
content using the DXT1/5/N or 3DC compressed color/normal map/cubemap
mipmapped texture formats. It was written by the same author as the open
source [LZHAM compression library](http://code.google.com/p/lzham/).

It can compress mipmapped 2D textures, normal maps, and cubemaps to
approx. 1-1.25 bits/texel, and normal maps to 1.75-2 bits/texel. The
actual bitrate depends on the complexity of the texture itself, the
specified quality factor/target bitrate, and ultimately on the desired
quality needed for a particular texture. 

crnlib's differs significantly from other approaches because its
compressed texture data format was carefully designed to be quickly
transcodable directly to DXTn with no intermediate recompression step.
The typical (single threaded) transcode to DXTn rate is generally
between 100-250 megatexels/sec. The current library supports PC
(Win32/x64) and Xbox 360. Fast random access to individual mipmap levels
is supported.

crnlib can also generates standard .DDS files at specified quality
setting, which results in files that are much more compressible by
LZMA/Deflate/etc. compared to files generated by standard DXTn texture
tools (see below). This feature allows easy integration into any engine
or graphics library that already supports .DDS files.

The .CRN file format supports the following core DXTn texture formats:
DXT1 (but not DXT1A), DXT5, DXT5A, and DXN/3DC

It also supports several popular swizzled variants (several are
also supported by AMD's Compressonator): 
DXT5_XGBR, DXT5_xGxR, DXT5_AGBR, and DXT5_CCxY (experimental luma-chroma YCoCg).

## Recommended Software

AMD's [Compressonator tool](https://github.com/GPUOpen-Tools/Compressonator)
is recommended to view the .DDS files created by the crunch tool and the included example projects.

Note: Some of the swizzled DXTn .DDS output formats (such as DXT5_xGBR)
read/written by the crunch tool or examples deviate from the DX9 DDS
standard, so DXSDK tools such as DXTEX.EXE won't load them at all or
they won't be properly displayed.

## Compression Algorithm Details

The compression process employed in creating both .CRN and
clustered .DDS files utilizes a very high quality, scalable DXTn
endpoint optimizer capable of processing any number of pixels (instead
of the typical hard coded 16), optional adaptive switching between
several macroblock sizes/configurations (currently any combination of
4x4, 8x4, 4x8, and 8x8 pixel blocks), endpoint clusterization using
top-down cluster analysis, vector quantization (VQ) of the selector
indices, and several custom algorithms for compressing the resulting
endpoint/selector codebooks and macroblock indices. Multiple feedback
passes are performed between the clusterization and VQ steps to optimize
quality, and several steps use a brute force refinement approach to improve 
quality. The majority of compression steps are multithreaded.

The .CRN format currently utilizes canonical Huffman coding for speed
(similar to Deflate but with much larger tables), but the next major
version will also utilize adaptive binary arithmetic coding and higher
order context modeling using already developed tech from the my LZHAM
compression library.

## Supported File Formats

crnlib supports two compressed texture file formats. The first
format (clustered .DDS) is simple to integrate into an existing project
(typically, no code changes are required), but it doesn't offer the
highest quality/compression ratio that crnlib is capable of. Integrating
the second, higher quality custom format (.CRN) requires a few
typically straightforward engine modifications to integrate the
.CRN->DXTn transcoder header file library into your tools/engine.

### .DDS

crnlib can compress textures to standard DX9-style .DDS files using
clustered DXTn compression, which is a subset of the approach used to
create .CRN files.(For completeness, crnlib also supports vanilla, block
by block DXTn compression too, but that's not very interesting.)
Clustered DXTn compressed .DDS files are much more compressible than
files created by other libraries/tools. Apart from increased
compressibility, the .DDS files generated by this process are completely
standard so they should be fairly easy to add to a project with little
to no code changes.

To actually benefit from clustered DXTn .DDS files, your engine needs to
further losslessly compress the .DDS data generated by crnlib using a
lossless codec such as zlib, lzo, LZMA, LZHAM, etc. Most likely, your
engine does this already. (If not, you definitely should because DXTn
compressed textures generally contain a large amount of highly redundant
data.)

Clustered .DDS files are intended to be the simplest/fastest way to
integrate crnlib's tech into a project.

### .CRN

The second, better, option is to compress your textures to .CRN files
using crnlib. To read the resulting .CRN data, you must add the .CRN
transcoder library (located in the included single file, stand-alone
header file library inc/crn_decomp.h) into your application. .CRN files
provide noticeably higher quality at the same effective bitrate compared
to clustered DXTn compressed .DDS files. Also, .CRN files don't require
further lossless compression because they're already highly compressed.

.CRN files are a bit more difficult/risky to integrate into a project, but
the resulting compression ratio and quality is superior vs. clustered .DDS files.

### .KTX

crnlib and crunch can read/write the .KTX file format in various pixel formats.
Rate distortion optimization (clustered DXTc compression) is not yet supported
when writing .KTX files. 

The .KTX file format is just like .DDS, except it's a fairly well specified
standard created by the Khronos Group. Unfortunately, almost all of the tools I've
found that support .KTX are fairly (to very) buggy, or are limited to only a handful
of pixel formats, so there's no guarantee that the .KTX files written by crnlib can
be reliably read by other tools.

## Creating Compressed Textures from the Command Line (crunch.exe)

The simplest way to create compressed textures using crnlib is to
integrate the bin\crunch.exe or bin\crunch_x64.exe) command line tool
into your texture build toolchain or export process. It can write DXTn
compressed 2D/cubemap textures to regular DXTn compressed .DDS,
clustered (or reduced entropy) DXTn compressed .DDS, or .CRN files. It
can also transcode or decompress files to several standard image
formats, such as TGA or BMP. Run crunch.exe with no options for help.

The .CRN files created by crunch.exe can be efficiently transcoded to
DXTn using the included CRN transcoding library, located in full source
form under inc/crn_decomp.h.

Here are a few example crunch.exe command lines:

1. Compress blah.tga to blah.dds using normal DXT1 compression:
  * `crunch -file blah.tga -fileformat dds -dxt1`

2. Compress blah.tga to blah.dds using clustered DXT1 at an effective bitrate of 1.5 bits/texel, display image statistic:
  * `crunch -file blah.tga -fileformat dds -dxt1 -bitrate 1.5 -imagestats`

3. Compress blah.tga to blah.dds using clustered DXT1 at quality level 100 (from [0,255]), with no mipmaps, display LZMA statistics:
  * `crunch -file blah.tga -fileformat dds -dxt1 -quality 100 -mipmode none -lzmastats`

3. Compress blah.tga to blah.crn using clustered DXT1 at a bitrate of 1.2 bits/texel, no mipmaps:
  * `crunch -file blah.tga -dxt1 -bitrate 1.2 -mipmode none`

4. Decompress blah.dds to a .tga file:
  * `crunch -file blah.dds -fileformat tga`

5. Transcode blah.crn to a .dds file:
  * `crunch -file blah.crn`

6. Decompress blah.crn, writing each mipmap level to a separate .tga file:
  * `crunch -split -file blah.crn -fileformat tga`

crunch.exe can do a lot more, like rescale/crop images before
compression, convert images from one file format to another, compare
images, process multiple images, etc.

Note: I would have included the full source to crunch.exe, but it still
has some low-level dependencies to crnlib internals which I didn't have
time to address. This version of crunch.exe has some reduced
functionality compared to an earlier eval release. For example, XML file
support is not included in this version.

## Using crnlib

The most flexible and powerful way of using crnlib is to integrate the
library into your editor/toolchain/etc. and directly supply it your
raw/source texture bits. See the C-style API's and comments in
inc/crnlib.h.

To compress, you basically fill in a few structs in and call one function:

```c
void *crn_compress( const crn_comp_params &comp_params,
                    crn_uint32 &compressed_size,
                    crn_uint32 *pActual_quality_level = NULL,
                    float *pActual_bitrate = NULL);
```

Or, if you want crnlib to also generate mipmaps, you call this function:

```c
void *crn_compress( const crn_comp_params &comp_params,
                    const crn_mipmap_params &mip_params,
                    crn_uint32 &compressed_size,
                    crn_uint32 *pActual_quality_level = NULL,
                    float *pActual_bitrate = NULL);
```

You can also transcode/uncompress .DDS/.CRN files to raw 32bpp images
using `crn_decompress_crn_to_dds()` and `crn_decompress_dds_to_images()`.

Internally, crnlib just uses inc/crn_decomp.h to transcode textures to
DXTn. If you only need to transcode .CRN format files to raw DXTn bits
at runtime (and not compress), you don't actually need to compile or
link against crnlib at all. Just include inc/crn_decomp.h, which
contains a completely self-contained CRN transcoder in the "crnd"
namespace. The `crnd_get_texture_info()`, `crnd_unpack_begin()`,
`crnd_unpack_level()`, etc. functions are all you need to efficiently get
at the raw DXTn bits, which can be directly supplied to whatever API or
GPU you're using. (See example2.)

Important note: When compiling under native client, be sure to define
the `PLATFORM_NACL` macro before including the `inc/crn_decomp.h` header file library.

## Known Issues/Bugs

* crnlib currently assumes you'll be further losslessly compressing its
output .DDS files using LZMA. However, some engines use weaker codecs
such as LZO, zlib, or custom codecs, so crnlib's bitrate measurements
will be inaccurate. It should be easy to allow the caller to plug-in
custom lossless compressors for bitrate measurement.

* Compressing to a desired bitrate can be time consuming, especially when
processing large (2k or 4k) images to the .CRN format. There are several
high-level optimizations employed when compressing to clustered DXTn .DDS
files using multiple trials, but not so for .CRN.

* The .CRN compressor does not currently use 3 color (transparent) DXT1
blocks at all, only 4 color blocks. So it doesn't support DXT1A
transparency, and its output quality suffers a little due to this
limitation. (Note that the clustered DXTn compressor used when
writing clustered .DDS files does *not* have this limitation.)

* Clustered DXT5/DXT5A compressor is able to group DXT5A blocks into
clusters only if they use absolute (black/white) selector indices. This
hurts performance at very low bitrates, because too many bits are
effectively given to alpha.

* DXT3 is not supported when writing .CRN or clustered DXTn DDS files.
(DXT3 is supported by crnlib's when compressing to regular DXTn DDS
files.) You'll get DXT5 files if you request DXT3. However, DXT3 is
supported by the regular DXTn block compressor. (DXT3's 4bpp fixed alpha
sucks verses DXT5 alpha blocks, so I don't see this as a bug deal.)

* The DXT5_CCXY format uses a simple YCoCg encoding that is workable but
hasn't been tuned for max. quality yet.

* Clustered (or rate distortion optimized) DXTc compression is only
supported when writing to .DDS, not .KTX. Also, only plain block by block
compression is supported when writing to ETC1, and .CRN does not support ETC1.
