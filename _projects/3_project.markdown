---
layout: page
title: Artificial Intelligence
description: We focus on the spiking neural networks and medical imaging analysis. 
img: /assets/img/ai_cover_image.jpeg
# redirect: https://unsplash.com
importance: 3
category: work
---
For further interest, please refer to [Deng et. al. 2021, ICLR](https://openreview.net/forum?id=FZ1oTwcXchK) and [Li et. al. 2021, ICML](https://arxiv.org/abs/2106.06984).

## SNN model
Spiking neuron networks (SNNs) are biologically inspired networks, which have received increasing attention due to their efficient computation. During their development, many mathematical models have been proposed to describe neuron behavior. The most widely used neuron model for SNN is the Leaky Integrate-and-Fire (LIF) model, which uses simple differential equations to describe the membrane potential  behavior of neurons. Its format of explicit iteration is governed by

$$ v_{temp}(t+1)=\alpha*v(t)+Ws(t) $$

where $$v$$ and $$v_{temp}$$ is the membrane potential, $$\alpha$$ means the decay factor, $$W$$ denotes the synapse weight, and $s$ denotes the spike input at time t.

When the membrane potential exceeds the pre-defined threshold $$V_{th}$$, it would produce a spike output $$\theta$$, and the membrane potential will decrease by two reset mechanisms:

soft reset: $$v(t+1)=v_{temp}(t+1)-V_{th}$$  
hard reset: $$v(t+1)=v_{temp}(t+1)*(1-\theta)$$

## works
The major bottleneck of the spiking neuron network is how to acquire a well-performance SNN, especially on a complex dataset, since directly using the backpropagation algorithm is not suitable for SNN.  
Currently, two ways can effectively obtain excellent SNNs: surrogate gradient and ANN to SNN. ANN to SNN means converting a well-performed source ANN to a target SNN. Surrogate gradient methods use a soft relaxed function to replace the hard step function and train SNN by BPTT.

### ANN to SNN
In this subsection, We use the soft reset and IF model ($$\alpha=1$$) to reduce the information loss between the source ANN and target SNN. Our approach is based on threshold-balancing method, which contain three steps:  
1. Train the source ANN that only contains Convolutional, average pooling, fully connected layer, and ReLU activation function. Record the maximum activation of each layer.
2. Copy the network parameter to target SNN and replace the ReLU function with the IF model. The threshold of each layer is setting to the maximum activation.
3. Run the target SNN with enough simulation length to achieve acceptable accuracy.  

The converted SNN needs thousands of simulation time to achieve the same accuracy as source ANN, which does not meet the high-efficiency characteristics. So our work is to explore how to obtain higher accuracy, and lower inference latency converted SNN.  

#### [Layer-wise conversion error](https://openreview.net/forum?id=FZ1oTwcXchK)
We split the conversion error into clipping error and flooring error. When the source ANN activation is larger than $$V_{th}$$, 
the SNN neuron's output is smaller than the output of the source neuron (clipping). And the reason for the flooring error is that
SNN can't send accurate values to the next layer. The threshold balancing method can eliminate the clipping error since 
it uses maximum activation as the threshold. However, it also will increase the flooring error, which can be reduced by
increasing simulation length. Here, we first find an equation to approximate the target SNN's spike frequency:

$$\bar{s}^{l+1} = V_{th}/T \cdot clip(\left \lfloor (T/V_{th}) \cdot W^l \bar{s}^l,0,T \right \rfloor)$$

Though analyze the output error between the source ANN and converted SNN, we decompose the conversion error into the output error of each layer. As a result, we can make the converted SNN closer to the source ANN by simply reducing the output error of each layer. Here we propose a method to reduce the flooring error: only to increase the SNN's bias by $$V_{th}/2T$$.

{% highlight ruby %}
for l=1 to L do 
  SNN.layer[l].thresh<-ANN.layer[l].maximum_activation
  SNN.layer[l].weight<-ANN.layer[l].weight
  SNN.layer[l].bias<-ANN.layer[l].bias + SNN.layer[l].thresh / (2*T)
end for
{% endhighlight %}


#### [Layer-wise Caibration]()

**Adaptive threshold.** We found that threshold balancing will cause a considerable flooring error, 
especially when the simulation length is not enough, since it uses the maximum activation as the SNN's threshold. 
In practice, In practice, an Appropriate reduction of the threshold will increase the SNN performance.  
Here, we minimize the optimization problem, which is formulated by

$$\min_{V_{th}} (clipfloor(x^l,T,V_{th}^l)-ReLU(x^l)).$$

We use a grid search with linarly sample N grids between $[0, max(x^l)]$ to determine the final result of $$V_{th}^l$$.

The previous method does not rely on real data statistics. It is made by a strong assumption that activation is uniformly 
distributed. In fact, we can get a better bias increment by analyzing the distribution of some training samples. 
Here, we propose two methods to calibrate the SNN's bias and weight, respectively layer-by-layer.  

**Bias correction (BC).** In order to calibrate the bias, we first define a reduced mean function:  

$$\mu_c(x)=\frac{1}{wh}\sum_{i=1}^{w}\sum_{j=1}^{h}x_{c,i,j}$$  

where w,h are the width and height of the feature-map, so $$\mu_c(x)$$ computes the spatial mean of the feature-map 
in each channel c. The spatial mean of conversion error can be written by:  

$$\mu_c (e^l) = \mu_c (x^l)-\mu_c(\bar{s}^l)$$

Then we can add the expected conversion error into the bias term as $$b^l<-b^l+\mu_c(e^l)$$. 

**Potential correction (PC).** Potential is similar to bias correction. In this method we can directly 
set $$v^l (0)$$ to $$T \times e^l$$ to calibrate the initial potential.


{% highlight ruby %}
for l=1 to L do
  SNN.layer[l].frequency = SNN.layer[l].output.sum(4) / T
  Layer[l].Error = ANN.layer[l].output - SNN.layer[l].frequency
  SNN.layer[l].bias <- SNN.layer[l].bias + Layer[l].Error.mean(0).mean(2).mean(3) # bias correct
  SNN.layer[l].mem <- SNN.layer[l].mean +  Layer[l].Error.mean(0) # potential correct  
{% endhighlight %}

**Weight calibration (WC).** The layer-wise conversion can be written as $$e^l = x^l - \bar{s}^l$$. Then we need to minimize the formulation below:  

$$\min_{w^l} \left \| e^l \right \|^2$$  

via stochastic gradient descent.

### Surrogate gradient
Surrogate gradient methods use a soft relaxed function to replace the hard step function and train SNN by BPTT. 
There are many shapes of surrogate gradients, like rectangles, exponential, and triangles. 
But during the training process, the surrogate gradient is always optimal? 

#### [Dspike]() 
Here we propose a new family of Differentiable Spike (Dspike) functions that can adaptively evolve during training to 
find the optimal shape and smoothness for gradient estimation. Mathmatically, Dspike function is like:

$$Dspike(x,b)=\frac{tanh(b(x-V_{th}))+tanh(b/2)}{2(tanh(b/2))}$$  

where b is the temperature to adjust the shape of the surrogate gradient function. Specifically we have 

$$\lim_{b\rightarrow 0_+} Dspike(x)\rightarrow x$$ and $$\lim_{b\rightarrow \infty_+} Dspike(x)\rightarrow sign(x-V_{th})$$.

Then we calculate the finite difference gradient (FDG) of the first layer's weight with a small constant step $$\varepsilon$$. 
For each weight value of first layer, we have:

$$\bigtriangledown_{\varepsilon,w_i}L = \frac{L(w_i+\varepsilon e_i)-L(w_i - \varepsilon e_i)}{2\varepsilon}$$

At each epoch, we use a batchsize training sample for our Dspike method to compute their FDG of first layer's weight. 
Then we calculate the three true gradients by BPTT under the situation: $$b$$, $$b+\delta b$$, and $$b-\delta b$$, 
respectively. Next, we will compare the cosie similarity between the true gradient and the FDG, and adjust 
the temperature of Dspike as the best $$b$$.

