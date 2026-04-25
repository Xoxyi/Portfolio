+++
title = "JPEG Image Compression"
date = 2024-01-01
description = "An implementation and analysis of the JPEG compression algorithm, comparing DCT and DFT transforms in terms of PSNR and human visual perception."
[extra]
math = true
+++

# JPEG Image Compression

**Authors:** Andrea Zito, Davide Camino  
**Affiliation:** Department of Computer Science, University of Turin

---

## Introduction

The chosen project is an implementation of the JPEG image compression algorithm. The goal of the project is to evaluate different compression techniques — in particular, to experimentally study the advantages that the Discrete Cosine Transform (DCT) offers over the Discrete Fourier Transform (DFT), both in terms of Peak Signal-to-Noise Ratio (PSNR) and in terms of qualitative human perception.

---

## JPEG Pipeline

The JPEG pipeline consists of several steps. The first ones prepare the image for the DCT; the subsequent ones exploit the domain change brought about by the DCT and are responsible for encoding the image transform as compactly and efficiently as possible.

### High-Level Steps

![JPEG pipeline overview](/images/Jpeg/pipeline.jpg)

1. **Color space conversion** — we move from a raw image on three channels (Red, Green, Blue) to a representation via Luminance, Blue Chrominance, and Red Chrominance.
2. **Subsampling** — since the human eye is more sensitive to variations in brightness than to color differences, we can subsample the Chrominance channels.
3. **Blockify** — rather than transforming the entire image, we divide it into 8×8 pixel squares so that, statistically, each small block does not contain too many different pieces of information.
4. **DCT** — the heart of JPEG compression. It converts the spatial representation of pixels into a frequency description. The DCT result provides the correlation of a block with the basis functions shown below, giving an indication of the frequencies that make up the block. We will return to this transform later when analyzing the differences between DCT and DFT.

![DCT basis functions](/images/Jpeg/Dctjpeg.png)

5. **Quantization** — setting aside Subsampling, this is the step that makes JPEG non-invertible (it is a lossy compression). The 64 values obtained from the DCT are divided by a quantization factor and rounded to the nearest integer. The quantization matrix has been carefully engineered taking into account human visual perception, seeking the best trade-off between quality and compression.
6. **ZigZag Reordering** — due to both the DCT and quantization, the non-zero values in the 8×8 block concentrate in the upper-left corner. The ZigZag reading order is designed to create the longest possible runs of zeros.
7. **Differential Encoding** — empirically, the largest value in the block is typically found in the first cell, which represents frequency 0 (hence called the DC component). Since this value is almost always very large and does not change much between consecutive blocks, it can be reduced by computing the difference between the first cell of block *i* and the first cell of block *i − 1*. The remaining cells have a much less regular distribution, so this technique is applied only to the DC component.
8. **Zero-Length Encoding** — now that values in the blocks are small and we have long runs of zeros, we can compress these runs by storing only their length. We do this only for the value 0, because it has been observed that runs of other values are not long enough to justify RLE for all values.
9. **Huffman Encoding** — after Zero-Length Encoding, the amount of information to be written is the minimum achievable. We then apply Huffman coding to produce a binary file in which this information is stored as compactly as possible. To conveniently write the translation table, it is useful for all values to be positive; since they are not, before Huffman encoding we clip the values to the range [−128, 127] and then shift them to [0, 256].

---

## Color Space Conversion

```haskell
rgb2ycbcr :: ImgRGB -> ImgYCbCr
```

The conversion from the RGB color space to the YCbCr color space is implemented with the following formula:

$$Y  = 0.299 \cdot r + 0.587 \cdot g + 0.114 \cdot b$$
$$Cb = 128 - 0.168736 \cdot r - 0.331264 \cdot g - 0.5 \cdot b$$
$$Cr = 128 + 0.5 \cdot r - 0.418688 \cdot g - 0.081312 \cdot b$$

Our implementation delegates to the `cv2` library function:

```python
def rgb2ycbcr(img) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    ycbcr = cv2.cvtColor(img, cv2.COLOR_RGB2YCrCb)
    y, cb, cr = cv2.split(ycbcr)
    return y, cb, cr
```

---

## Subsampling

```haskell
subsample :: ImgYCbCr -> ImgYCbCr
```

The subsampling scheme used is 4:2:0: the resolution of the chroma part of the image is halved both horizontally and vertically by averaging 2×2 pixel blocks.

![4:2:0 chroma subsampling illustration](/images/Jpeg/Subsampling.png)

Again, the `cv2` library function is used:

```python
def subsample(chroma: np.ndarray) -> np.ndarray:
    return cv2.resize(chroma, (chroma.shape[1] // 2,
                               chroma.shape[0] // 2),
                      interpolation=cv2.INTER_LINEAR)
```

---

## Blockify

```haskell
blockfy :: ImgYCbCr -> ([Block], [Block], [Block])
```

The image is divided into 8×8 blocks for each color space component, yielding a triple of lists of 8×8 blocks:

```python
def blockify(img: np.ndarray) -> np.ndarray:
    blocks = np.zeros((img.shape[0] // 8 * img.shape[1] // 8, 8, 8))
    n = 0
    for i in range(0, img.shape[0], 8):
        for j in range(0, img.shape[1], 8):
            blocks[n] = img[i:i+8, j:j+8]
            n += 1
    return blocks
```

---

## DCT

```haskell
dctBlocks :: [Block] -> [Block]
```

Our implementation applies SciPy's `dct` function to each block. Since the blocks are two-dimensional, the transform is applied first along rows then along columns:

```python
def dct_transform(blocks: np.ndarray) -> np.ndarray:
    dct_blocks = np.zeros((blocks.shape[0], 8, 8))
    for i in range(blocks.shape[0]):
        dct_blocks[i] = dct(
            dct(blocks[i], axis=0, norm='ortho', type=2),
            axis=1, norm='ortho', type=2
        )
    return dct_blocks
```

---

## Quantization

```haskell
quantize :: Block -> [Block] -> [Block]
```

Given two 8×8 quantization matrices — one for luminance (Y) and one for chrominance (Cb, Cr) — quantization divides each component of each image block by the corresponding component of the quantization matrix. The matrices used are:

**Luminance**

```
16  11  10  16   24   40   51   61
12  12  14  19   26   58   60   55
14  13  16  24   40   57   69   56
14  17  22  29   51   87   80   62
18  22  37  56   68  109  103   77
24  35  55  64   81  104  113   92
49  64  78  87  103  121  120  101
72  92  95  98  112  100  103   99
```

**Chrominance**

```
17  18  24  47  99  99  99  99
18  21  26  66  99  99  99  99
24  26  56  99  99  99  99  99
47  66  99  99  99  99  99  99
99  99  99  99  99  99  99  99
99  99  99  99  99  99  99  99
99  99  99  99  99  99  99  99
99  99  99  99  99  99  99  99
```

---

## ZigZag Reordering

```haskell
zigZag :: [BlockI] -> [Block1D]
```

The ZigZag reordering is implemented using a lookup table that specifies the reading order of cells within a block:

```
 0   1   8  16   9   2   3  10
17  24  32  25  18  11   4   5
12  19  26  33  40  48  41  34
27  20  13   6   7  14  21  28
35  42  49  56  57  50  43  36
29  22  15  23  30  37  44  51
58  59  52  45  38  31  39  46
53  60  61  54  47  55  62  63
```

The matrix is treated as a flat array where each value indicates the source cell of the block, read left-to-right and top-to-bottom starting from 0.

---

## Differential Encoding

```haskell
diffEncoding :: [Block1D] -> [Block1D]
```

At this point in the pipeline, blocks are one-dimensional lists of 64 integer values. Differential encoding replaces, for each block *i*, the value at position 0 with its difference relative to the first value of block *i − 1*. The first block is left unchanged.

---

## Zero-Length Encoding

```haskell
zeroLength :: [Block1D] -> [Int]
```

Since blocks contain many consecutive zeros, Zero-Length Encoding replaces runs of consecutive zeros with `[…, 0, n, …]` where *n* is the count of consecutive zeros.

For example:

- original vector: `[5, 0, 0, 3, 0, 0, 0, 1, 7, 0, 9, 0, 0, 0, 0]`
- encoded vector:  `[5, 0, 2, 3, 0, 3, 1, 7, 0, 1, 9, 0, 4]`

---

## Huffman Encoding

```haskell
huffman :: [Int] -> [Byte]
```

To find the bit sequence to assign to each value after ZRLE, a binary tree is constructed where leaves represent those values. Leaves closer to the root correspond to higher-frequency values; leaves farther from the root correspond to lower-frequency values. To encode a value, we trace the path from root to its leaf.

For decoding, the tree does not need to be stored directly — it can be reconstructed from two pieces of information: how many symbols have each code length, and which symbols have which length.

In practice, during development we found it complex to guarantee that the encoding and decoding trees were identical. To solve this, we first built an encoding tree based purely on symbol frequencies, extracted the structural information from it, and then rebuilt a second encoding tree from that information. This guarantees that the tree used for encoding is the same one used for decoding.

In the JPEG implementation, luminance (Y) and chrominance (Cb, Cr) are coded with two separate Huffman trees.

---

## Experiments

### Test Images

For our experiments we used 64 RAW images shot with a camera and selected to cover a sufficient variety of subjects. The image below shows a collage of all the photos used.

![Collage of 64 test images](/images/Jpeg/collage.jpg)

The native resolution is 4000×6000. To test the algorithms at different resolutions, we resized each image to half its original dimensions and then to one tenth.

---

### ZRLE Compression Rate

We study compression capacity by evaluating the ratio between the number of values to store in uncompressed and compressed form. We stop the pipeline just before Huffman encoding and consider the ratio between the uncompressed image and the ZRLE-compressed image. The uncompressed size (per channel) assumes one byte per value:

$$\text{dim} = 3 \cdot \text{width} \cdot \text{height} \quad \text{bytes}$$

The compressed size is the length of the ZRLE-encoded stream, also assuming one byte per value.

![ZRLE compression rate across resolutions](/images/Jpeg/compression_rate.png)

Even in the worst case, the applied transforms compress the image by at least a factor of 10. As expected, higher-resolution images achieve better average compression ratios: the 8×8 blocks cover increasingly local regions of the image, making them more homogeneous and producing more zero-valued entries after quantization, which in turn enables stronger ZRLE compression.

---

### Huffman Compression Rate

In this experiment we consider the complete JPEG pipeline and analyze how much additional compression Huffman encoding provides.

The compressed file size is estimated from three components: two tables for reconstructing the Huffman tree (as described above) — four tables in total, two for luminance and two for chrominance — plus the three raw byte streams (Y, Cb, Cr). Each table entry is counted as one byte.

![Huffman compression rate across resolutions](/images/Jpeg/compression_rate_huf.png)

Adding Huffman encoding yields significantly higher compression ratios than the pipeline up to ZRLE alone. This gain arises because many values in the ZRLE stream occur with high frequency and can therefore be encoded in fewer than 8 bits.

---

### DCT vs. DFT

In this section we ask whether the Cosine transform is genuinely more efficient than the Fourier transform. Since the JPEG quantization matrix is designed specifically for the DCT, we cannot use it directly for comparison. Instead, we use a binary matrix that retains only the upper-left values in the transformed block (corresponding to the lowest frequencies). Using the same quantization matrix for both DCT and DFT gives equal compression rates; we then compute the PSNR between the original image and the image compressed first with DCT, then with DFT, and compare the results.

![PSNR comparison: DCT vs. DFT](/images/Jpeg/PSNR.png)

Regardless of the original image quality, DCT consistently achieves better PSNR than DFT. We also observed that with the binary quantization matrix, PSNR is higher than with the JPEG quantization matrix for the same transform — though compression is considerably worse. The next experiment investigates the relationship between PSNR and perceived visual quality.

---

### PSNR vs. Human Perception

As a final experiment we investigated how well PSNR reflects human visual perception. The comparison is again between DCT with the JPEG quantization matrix and DFT with the binary matrix from the previous section. The experiment aims to show that two compressions with similar PSNR values can produce perceptually very different decoded images.

![Comparison image: full view](/images/Jpeg/PSNR_fig1.png)

At first glance both images look satisfactory, but close inspection reveals significant differences. The zoom below shows that fine details remain sharp in the DCT-compressed image while they are badly corrupted by the DFT compression.

![Comparison image: detail zoom](/images/Jpeg/PSNR_fig1_zoom.png)

This degradation is most likely attributable to the different periodization assumptions of the two transforms, combined with the fact that the binary quantization matrix acts as a sharp ideal band-pass filter, which is known to introduce ringing and Gibbs-like distortions at edges.

This raises the question: why does the DFT image score higher on PSNR despite looking worse? To investigate, we turn to an image where the difference between the two transforms is visible without zooming in.

![Comparison image: homogeneous region](/images/Jpeg/PSNR_fig2.png)

Focusing on a uniform area — for instance, a single tile — makes the answer clear.

![Comparison image: homogeneous region zoom](/images/Jpeg/PSNR_fig2_zoom.png)

In flat regions the DFT result looks better. Two factors explain this: first, the binary quantization matrix retains far more information; second, where there are no edges — and high frequencies are negligible — the DFT introduces no distortion, whereas the DCT image shows clear blocking artifacts from quantization. These artifacts inflate the PSNR and carry the same mathematical weight as the edge distortions introduced by DFT. Perceptually, however, these two types of artifact are weighted very differently: viewers find the edge corruption from DFT far less tolerable than the blocking from DCT.

---

## Conclusion

The implementation of the JPEG algorithm and the analysis of its various compression stages have demonstrated the effectiveness of the DCT over the DFT, both in terms of compression ratio and perceived quality. The DCT, backed by a quantization matrix designed specifically for it, preserves greater visual fidelity in detailed areas and avoids the edge artifacts typical of the DFT. 

The compression-rate analysis shows that Huffman encoding substantially increases overall efficiency, particularly for high-resolution images where JPEG's block structure achieves dramatic space reductions while maintaining acceptable quality.

The visual comparison also highlights an important limitation of PSNR as a quality metric: it captures pixel-level similarity to the original but does not reflect human perception, which finds DCT-compressed images more pleasant even when their PSNR is lower than DFT-compressed counterparts. Alternative perceptual metrics have been proposed in the literature — evaluating their effectiveness in this context would be a natural direction for future work.

In summary, the JPEG compression pipeline — with its DCT core and purpose-built quantization matrix — provides a robust and well-balanced solution between efficiency and visual quality, confirming its enduring status as a standard for image compression.

---

## References

1. G. K. Wallace, "The JPEG still picture compression standard," *IEEE Transactions on Consumer Electronics*, vol. 38, no. 1, pp. xviii–xxxiv, 1992.
2. N. Ahmed, T. Natarajan, and K. R. Rao, "Discrete cosine transform," *IEEE Transactions on Computers*, vol. C-23, no. 1, pp. 90–93, 1974.
3. D. A. Huffman, "A method for the construction of minimum-redundancy codes," *Proceedings of the IRE*, vol. 40, no. 9, pp. 1098–1101, 1952.
