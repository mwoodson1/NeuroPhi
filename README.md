# NeuroPhi
Final project for 15-418 Parallel Computer Architecture and Programming. NeuroPhi is a framework for training convolutional neural networks on the Xeon Phi. We made use of the Fast Fourier Transform in performing the convolution and proposed several ways to improve memory efficiency. Also implemented parallel batch training and other tricks. Achieved state of the art training speeds at time of writing.

## Motivation
NeuroPhi differentiates itself from existing parallel CNN training frameworks by computing convolutions in the Fourier domain instead of the time domain, which is optimal for large images and kernels.

Unlike the existing parallel CNN training frameworks that rely on CUDA and GPU computation, NeuroPhi's target hardware is a CPU and Intel Xeon Phi Coprocessor. Neural network operations are computationally dense and highly parallelizable; as GPU core counts continue to increase, GPU ANN training is not compute limited, but bandwith limited by the rate at which data can be offloaded to the GPU. The Xeon Phi coprocessor, Intel's competitor to NVIDIA's Tesla, supports code written for x86  architecture, and therefore has the potential to run neural network training algorithms with fewer bandwidth limited offloads. By writing NeuroPhi for CPU and Xeon Phis, we also give users the convenience of extending the framework with standard C++ parallelization tools including OpenMP, Cilk, and ISPC.

## Background
Neural networks are a biologically-inspired machine learning algorithm. The ability to learn using a hierarchy of simple transfer functions has made neural networks key computational models for applications such as computer vision and speech recognition. However, large neural networks--which may have several hidden layers, thousands to millions of parameters--are very computationally expensive to train.

Convolutional neural networks are a type of neural network that reduce the number of parameters required to do efficient feature extraction by applying the same filter(s) to several contiguous regions of an image. The feature extraction phase of the learning process may have many layers, including, most notably:

### Convolutional layers
The input parameters of convolutional layers are a set of learnable filters. Each one of these filters is then convolved with the input volume to produce its output. The process of convolution can be expressed by the following formula in 1D

![](http://pages.jh.edu/~bmesignals/New/disc_conv_eqn.gif)

The following is a naive implementation of 2D convolution:
```
for w in 1..W
  for h in 1..H
    for x in 1..K
      for y in 1..K
        output(w, h) += input(w+x, h+y) * filter(x, y)
      end
    end
  end
end
```

Most CNN libraries, including <a href="http://caffe.berkeleyvision.org/doxygen/classcaffe_1_1ConvolutionLayer.html">Caffe</a>,  use linear algebra tricks to manipulate the matrices involved in the convolution and take advantage of fast matrix multiplication with cblas. In particular one can "stack" the filters on top of one another and do 1 large convolution instead of several smaller ones.
</p>

<h3>Approach</h3>
<p>We began our work from the existing <a href="https://github.com/sdemyanov/ConvNet">ConvNet</a> framework, which supports serial CNN training on a CPU, very minimal parallelization for CNN training on a CPU, and GPU CNN training using <a href="https://code.google.com/p/cuda-convnet2/">CudaConvnet</a>. We aimed to dramatically increase the parallelization of the CPU version of ConvNet, make it suitable for doing CNN training/classification on large images, and offload some of its computations to the Xeon Phi coprocessor.</p>
<p>Neural network training consists of forward propagation, backwards propagation, and weight updates of the various layeys of the network. We timed ConvNet's serial CPU implementation of CNN training in the computationally intensive stages of forward propagation, backwards propagation, and weight updates for the convolutional, fully-connected, and scaling layers, and determined that most of the time during network training was spent on convolutional layer computations. Back propgation through the convolutional layer was particularly slow, and this training stage is dominated by image convolutions. We therefore decided to focus our efforts on optimizing the parallelization of the actual convolution operation.</p>
<p>NeuroPhi's convolutional layer uses Fast Fourier Transforms, which transform the convolution operator into a simple element-wise product of the FFT of the image and FFT of the kernel. Though there is overhead in computing an FFT and inverse FFT, the convolution in the Fourier domain is highly vectorizable and is only O(nlogn) compared to the O(n^2) time domain implementation.</p>
<p>One <a href=http://arxiv.org/pdf/1410.0759.pdfcommon>complaint of the FFT convolution implementation</a> is that it doubles the memory useage of the convolution operation, since the convolution kernel must be padded to match the size of the input. However, a trick can be employed for convolutions applied to images. A convolution input typically has a real and imaginary component. Since images have only real components, the expanded kernel can be made to occupy the "complex" portion of the image. This halves the memory useage of the computation and approximately halves the latency of the operation, since only one FFT is now required.</p>
<p>Another important optimization for a parallel CNN training framework is concurrent training on multiple images, known as parallel batch processing. New network weights are calculated concurrently over multiple images, and the weights from several images are then reduced so that a single weight change is applied to the network for a batch of images. This can be acheived by multithreading image computations through the network and heeding the critical region in which weight computations across images (threads) are summed.</p>
<p></p>
<div>
    <a href="https://plot.ly/~dmrager/118/" target="_blank" title="Time for 128 images to complete various CNN training stages" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/118.png" alt="Time for 128 images to complete various CNN training stages" style="max-width: 100%;width: 525px;"  width="525" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:118" src="https://plot.ly/embed.js" async></script>
</div>
<p></p>
<h3>Results</h3>
All results below were obtained by running code on a single node of <code>latedays</code>, on two, six-core Xeon e5-2620 v3 processors.
<p></p>
<p>We compared three implementations of image convolution: 1) the naive, time domain implementation pictured in the  background section 2) A recursive FFT implementation of convolution, new to NeuroPhi, and 3) An unrolled FFT implementation of convolution, new to NeuroPhi. Convolution runtimes on 512x512 pixel images (roughly the average size of images in the ImagNet database) for serial versions of the three algorithms as a function of kernel size are shown below. While runtime quickly blows up for large kernel sizes in the time domain, it remains fairly constant over kernel sizes in the two Fourier domain algorithms.</p>
<div>
    <a href="https://plot.ly/~dmrager/135/" target="_blank" title="Serial Convolution Implementations" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/135.png" alt="Serial Convolution Implementations" style="max-width: 100%;width: 464px;"  width="464" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:135" src="https://plot.ly/embed.js" async></script>
</div>

<p>We then implemented parallelized versions of all three algorithms. The time domain algorithm was parallelized with OpenMP multi-threading and ISPC vector intrinsics. The recursive FFT algorithm was parellized with <code>cilk_spawn</code>'s fork-and-join parallelism. The unrolled FFT algorithm was already so fast after parallelizing it with ISPC vector intrinsics that spawning new threads was generally too much overhead and just incurred latency. Cilk had too much overhead and only a single for loop was efficiently parallelized with OpenMP. It is also worth noting that while OpenMP provides <code>#pragma vector</code> and <code>#pragma simd</code>  directives for vector operations, we got signficantly better speedup by writing vector intrinsics with ISPC. Runtimes of all three parallel implementations on 512x512 pixel images as a function of kernel size are shown below. Once again, while runtime quickly blows up for large kernel sizes in the parallel time domain algorithm, it remains fairly constant over kernel sizes in the two parallel Fourier domain algorithms.</p>
<div>
    <a href="https://plot.ly/~dmrager/158/" target="_blank" title="Parallel Convolution Implementations" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/158.png" alt="Parallel Convolution Implementations" style="max-width: 100%;width: 441px;"  width="441" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:158" src="https://plot.ly/embed.js" async></script>
</div>

<p>Serial implementation to parallel implementation speed-ups for the three algorithms are shown below. </p>
<div>
    <a href="https://plot.ly/~dmrager/167/" target="_blank" title="Speedup vs. Kernel Size" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/167.png" alt="Speedup vs. Kernel Size" style="max-width: 100%;width: 525px;"  width="525" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:167" src="https://plot.ly/embed.js" async></script>
</div>

<p>Our fastest convolution performance was acheived using the unrolled, vectorized FFT with the complex component convolution trick previously described, manual control of the thread count to match the number of execution contexts on a latedays machine, and setting the <code>THREAD_AFFINITY</code> compiler flag to <code>SCATTER</code>. This was unusual behavior as one would think having the data in each thread being sent to different cores would hinder performance. With more time, we would like to investigate why this aided performance.</p>
<p>The figure below compares performance of our fastest parallel unrolled FFT to well known parallel CNN training frameworks <a href=http://arxiv.org/pdf/1412.7580v1.pdf>Facebook FFT</a>, CudaConvNet2, and Caffe. <a href="https://github.com/soumith/convnet-benchmarks">Benchmark</a> times were take from the link. Timings are for a single pass through 5 convolution layers for a 128x128 image and 11x11 kernel. Note that the NeuroPhi was actually tested on 9x9 and 15x15 kernels and that our 11x11 kernel time is an interpolation of our time between these two values. However, the estimated 11x11 kernel time is consistent with the rest of our data. GPU data from all the established frameworks was obtained on a single Titan Black NVIDIA GPU. Our implementation ran on single a latedays node CPU.  This estimated comparison shows that our parallel FFT algorithm is 3x as fast as Caffe's convolution and 2x as fast as cudaConvNet's. Our implementation is approximately as fast Facebook's recently released FFT implementation.</p>
<div>
    <a href="https://plot.ly/~dmrager/160/" target="_blank" title="5 Conv Layers, 128x128 image, 11x11 Kernel" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/160.png" alt="5 Conv Layers, 128x128 image, 11x11 Kernel" style="max-width: 100%;width: 441px;"  width="441" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:160" src="https://plot.ly/embed.js" async></script>
</div>
<p>Though memory usage is always higher for FFT implementations of convolution than time domain implementations, the complex image component trick described in the "Approach" section did halve the memory useage of a standard FFT implementation. Shown below is the mean memory useage across the various kernel sizes reported in previous figures as a function of time. The time domain convolution essentially only requires memory equal to the size of the the image and the kernel, which is constant over time. A standard FFT implementation requires 2x memory at the start, as the kernel is padded to match the image size. We then have to round the image to the next power of 2 in order for us to avoid border effects, which doubles memory again to 4x the size of the image. Using the trick, we can store both the image and kernel in the same matrix. And finally, theoretically we could use a FFT implementation that does not require power of 2 inputs which would then not require us to round to the next power of 2. This would save additional memory. Data for this plot was obtained by averaging the memory used at 3 points in the program:very beggining, after resizing of images and rounding, and very end. The memory used was averaged over all kernel sizes for each implementation. We assume that memory usage for an implementation would be roughly consistent across kernel sizes and believe this to be a valid based on preliminary data analysis.</p>
<div>
    <a href="https://plot.ly/~dmrager/176/" target="_blank" title="Memory usage relative to Image Size vs Time" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/176.png" alt="Memory usage relative to Image Size vs Time" style="max-width: 100%;width: 525px;"  width="525" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:176" src="https://plot.ly/embed.js" async></script>
</div>

<p>We focussed our initial parallel batch training efforts on backpropagation and were specifically interested in how batch training, independent of convolution optimizations or other vectorizations, would improve speedup. The figure below shows the time for 128 images to be processed through the convolutional layer in backprogation when those images are processed serially vs in parallel with OpenMP multi-threading. We were able to acheive an additional ~1.8x speedup on backpropagation through the convolutional layer as compared to the standard CPU implementation. To acheive this speedup, it was important to manually control set the number of thread to 12 for the 12 <code>latedays</code> execution contexts and to statically schedule the threads to receive chunks of about 10 images each. All images in our dataset were the same size, so it made sense to essentially pre-assign every image to a thread to reduce thread scheduling overhead.</p>

<div>
    <a href="https://plot.ly/~dmrager/174/" target="_blank" title="Mean Time for 128 Images Through Conv. Backpropagation" style="display: block; text-align: center;"><img src="https://plot.ly/~dmrager/174.png" alt="Mean Time for 128 Images Through Conv. Backpropagation" style="max-width: 100%;width: 525px;"  width="525" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="dmrager:174" src="https://plot.ly/embed.js" async></script>
</div>

<h3>Challenges and Future Directions</h3>
<p>One of the primary challenges of our project was integrating ConvNet's MATLAB frontend with C++ backend code and especially with the Xeon Phi. Though we were able to offload portions of standalone C++ code to the Phi, such as our fft scripts, there were many linking and library issues when trying to integrate the entire MATLAB embedded, ConvNet code base. We believe that parallel batch processing, in particular, could benefit from being run on the many-cored Phi, and would like to extend NeuroPhi to this framework in the future. We would also like to extend our fast FFT convolution implementation to support images that are not powers of two to reduce its memory footprint.</p>

<p>Please find references for this work linked throughout this page, as appropriate.</p>
