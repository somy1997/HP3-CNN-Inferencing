## FFT based convolution
#### Input pre-processing:
* Input is of the form `Batch_size * input_channels * H * W`
* **Pad the input:** Each Input layer is padded by the required number parallely using cuda kernel pad_input
* **Convert the input to frequency domain by FFT:** FFT of the input is obtained using CUFFT library’s functions 
  * **cufftPlanMany** that creates a plan supporting batched input
  * **cufftExecR2C** is used for real-to-complex forward transform for single precision.

#### Filter pre-processing:
* Each filter is of the form `output_channels * H * W` 
* **Flip around the center:** Each element of the filter is flipped around the center. Not doing this operation on the kernel results in Correlation (not convolution).
				
                                   F(i,j,k) = (D-i, H-j, W-k)

* **Pad the filter:** The filter is padded to the size of the input. This is essential in order to get the correct output by pointwise product in the frequency domain  
* **Align the filter:** The filter is required to have it’s Central element of the kernel in the (0,0) position. This is essential in order to get the correct output by pointwise product in the frequency domain

                            F(i,j,k) = ((i - D/2) % D, (j - H/2) % H, (k - W/2)% W)

* **Convert the filter to frequency domain by FFT:** FFT of the filter is obtained by CUFFT library’s functions 
  * **cufftPlanMany** that creates a plan supporting batched input
  * **cufftExecR2C** is used for real-to-complex forward transform for single precision

#### Convolution Operation:
* Preprocessed input and preprocessed filter in frequency domain are multiplied pointwise
* **Convert back the obtain product using Inverse FFT:** The final convolution result is obtained by CUFFT library’s functions 
  * **cufftPlanMany** that creates a plan supporting batched input
  * **cufftExecC2R** is used for complex-to-real forward transform for single precision
* This operation “performs” cyclic convolution: The convolution kernel wraps around the input borders in all the dimensions. But, we require the output to be clamped in the borders and hence require post-processing on the output.
* **High speed version:** The operation is looped over batch size and the precomputed FFT of input is replicated number of filters times and pointwise product is carried out.  This has a memory read overhead.
* **Low memory version:** The FFTs are calculated when required. It is looped over output channels. This leads to FFTs being calculated for the same input multiple times and hence is slow. This uses less memory though.

#### Output Post-Processing:
* **Crop and stride:** The output obtained in the previous step is cropped to `Input_size - filter_size + 1`  According to the input stride, the required elements are transferred to the output.

## Comparison with cuDNN’s FFT  
CUDNN’s `CUDNN_CONVOLUTION_FWD_ALGO_FFT` has the following shortcomings which are alleviated in our FFT based implementation :
* Input’s feature map `height + 2 * zero-padding` height must equal 256 or less
* _Our implementation doesn’t have any restriction on the input height_
* Input’s feature map `width + 2 * zero-padding` width must equal 256 or less
* _Our implementation doesn’t have any restriction on the input width_
* The vertical and horizontal filter stride must equal 1
* _There is no restriction on the stride_ 
* Filter’s height must be greater than zero-padding height 
* _Filter’s height can be less than padding height_ 
* Filter’s width must be greater than zero-padding width
* _Filter’s width can be less than padding width_

## Additional points
* FFT can be really fast and useful when the filter sizes are large.
* FFT with small kernel size are slow due to insufficient parallelism to cover the memory latency.
* The current implementation is for square images. This can be easily scaled up to images with unequal height and width.
* The computation complexity of FFT-based algorithms depends on the padded image size, and in its turn padded image size depends both on the input and convolution kernel sizes.
* It is an observation that if the FFT dimensions are small multiples of powers of N, where N varies from 2 to 8 for CUFFT, the performance and precision are best. This can be obtained by appropriately padding the image and the filter.

## References
* http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.472.2396&rep=rep1&type=pdf
* https://docs.nvidia.com/cuda/cufft/index.html#fftw-conversion-guide