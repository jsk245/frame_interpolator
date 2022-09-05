## Frame Interpolator

### Description
Frame interpolation is a task where the goal is to predict a frame that fits in between two others. The most common application of tools performing this task is for upscaling video frames per second (to make motion smoother usually). The development of neural net based models for this task are particularly promising as they should generalize better to all types of videos when compared to traditional methods of interpolation using simpler math. 

### Contents + Limitations
In this repo, I include a Python notebook that can be run on Google Colab that contains the code for my variational autoencoder (architecture described below). This model was implemented in JAX using the Haiku library. Tensorflow was used to obtain the data (the DAVIS dataset has been the main one used), and the code to upscale video fps using sklearn's scikit-video package has been provided at the end of the notebook. **I am still in the process of training the model** which will take some time due to computational restraints (for reference, this deep learning frame interpolation paper takes 20 hours to train on 4 Nvidia Titan X (Pascal) gpus: https://arxiv.org/pdf/1708.01692.pdf). Additionally, my architecture is a simplified version of what I originally intended it to be (I initially wanted to include attention to help with combining the feature representations of the separate frames), and I won't be able to perform ablation studies to see which parts of my architecture are essential for performance. I also describe potential changes I would like to test below and in the notebook.

### Initial Results
These are the most promising results from the first two hours of training (the center image is generated by the model, and the model has not seen any of these videos previously). While the model initially performed naive linear interpolation, it has learned to perform interpolation more intelligently. However, as can be seen most easily in the third sequence of images, remnants of the initial strategy are still present (there are two white lines on the bottom right of the image), and all three images are generally blurred (present in naive linear interpolation as well). The colors are not as vibrant as those of the original image, but that should be fixed as the MS-SSIM loss (which improves the structural similiarity between images) evens out with the L1 loss component (which improves the color and contrast). Overall, I believe my model should be able to achieve reasonable results (with interpolation taking roughly 15s per frame), and I would like to revisit the problem in the future.

![Capture1](https://user-images.githubusercontent.com/93054906/188345579-9546c071-6bbc-4e97-88be-36123fbc4045.PNG)
![Capture2](https://user-images.githubusercontent.com/93054906/188345726-a07a6397-cefe-4d03-81a7-ab7b9cad6457.PNG)
![Capture3](https://user-images.githubusercontent.com/93054906/188345614-16ddc65f-34c7-442f-af57-90110edd8f0b.PNG)

### Architecture
This model is essentially an ensemble model of a couple different computer vision papers. The main block used in this model makes use of the Fast Fourier Transform (fft) with a 1x1 convolution applied to it to gather information across the entire image. Since the fft transforms signals into the frequency domain, convolving in this domain has the interpretation of convolving across the entire image, and since I apply the fft in 3 dimensions (height, width, time), I allow the model to learn information through every image fed to it. This block is based on the one presented in this paper: https://arxiv.org/abs/2109.07161. This original paper tackles the task of inpainting, and the main change I've made to it is the application of the global side of the model to the time dimension. 

The main way of connecting the blocks for gradient flow was inspired by this paper: https://arxiv.org/abs/2202.04901. Originally, I based the entire architecture on the inpainting paper, but it was much slower at training on the full DAVIS dataset (it was able to overfit to videos successfully). I also created a model where I included the fft block in between downsampling steps and connected them to the upsampling steps similarly to the paper, but I found that this model still struggled to train effeciently, and I thought that the paper's introduction of feeding downsampled versions of the original frames to the decoder was a smart way to help the model learn.

Finally, I rewrote an implementation of a loss functions that uses a weighted combination of the MS-SSIM and L1 loss functions. The Nvidia research team found this combination of loss functions to work well in image restoration (paper: https://research.nvidia.com/sites/default/files/pubs/2017-03_Loss-Functions-for/NN_ImgProc.pdf), and their implementation was a slightly altered version of the two losses in order to balance them better. My understanding is that the MS-SSIM loss takes information from around every pixel at multiple scales with a gaussian filter to see if the prediction was relatively closs in structure, and the L1 loss compares the immediate color of the predicted pixel vs the original to help with correcting the color. 

### Training + Potential Training Changes
Overall, I trained this model using the Adam optimizer with a cosine decay schedule and an initial learning rate of 1e-3. I simply batched 4 pairs of surrounding frames from 4 different videos at a time, and I would probably either increase the # of frames from each video/the batch size if provided with more RAM. I augmented the frames with a random chance to be flipped horizontally, flipped vertically, presented in reversed order, cropped, and spaced further away from the image being predicted (to simulate lower fps videos). Overall, every frame was resized to 256x256 images, but I think training on higher resolution images is where the fft should help the most in theory as convolutions are unlikely to incorporate distant information well. The balance between the two losses is also quite finicky, and I changed the weights when needed. Finally, I'm interested in testing feeding >2 surrounding images, but I don't think the gains from this would be particularly significant.
