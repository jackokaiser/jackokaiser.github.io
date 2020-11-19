---
title: PhD on Synaptic Learning for Neuromorphic Vision'
date: 2020-08-18
permalink: /posts/2020/08/synaptic-learning
tags:
  - Spiking Neural Networks
  - Humain Brain Project
  - Dynamic Vision Sensor (DVS)
---

After over five years of researching in the field of biological learning for robotics, I have successfully completed my PhD.
The full dissertation is published on the [KIT bibliothek](https://publikationen.bibliothek.kit.edu/1000122690).
In this blog post, I introduce the field of neuromorphic computing in layman's terms, and present some of the results.

The Dynamic Vision Sensor is a very special type of camera.
Conventional cameras take images at a given rate, around 30 times per seconds.
Standard computer vision algorithms have to read and process these images, one by one.
Modern smartphones are capable to perform such processing, but it consumes a lot of energy.
On the other hand, the Dynamic Vision Sensor will only emit an individual event, when the light received by one pixel has changed.
An event consists of the address of the pixel which changed, and whether light increased or decreased for this pixel.
In other words, when the scene does not change, and the sensor does not move, no events are emitted.
A computer vision algorithm can now run more efficiently: instead of processing full images regurlarly, it only has to process individual events.


{:.wrapper-md}
![Dynamic Vision Sensor](/images/PhD/dvs_vs_camera.png "Dynamic Vision Sensor")

Until now, most computer vision algorihtms were tailored to process images, not events.
This also includes deep neural networks, which are used today in almost every computer vision problems.
A deep neural network consists of neurons connected to each other in layers.
Every neuron has a value, which is computed from the neurons in the previous layers and communicated to the neurons in the next layer.
The actual computations depends on some values, called weights, which are learned.
To process an image, the neurons of the first layer simply take the value of the pixels in the image.
The values of the second layer of neurons is computed from the first layer and so on, until the last layer of neuron which is the output.
Therefore, for every procssed image, all neurons of a layer have to communicate their values synchronously.
This is another expensive process (although it fits very well to a particular hardware: the Graphical Processing Unit (GPU)).

<div class="wrapper-md">
    <img src="/images/PhD/analog-neuron.png" alt="Conventional neuron" title="Conventional neuron"/>
</div>

Spiking neurons are a variation of conventional neurons.
They are particularly suited to processing events rather than images.
Unlike a conventional neuron, a spiking neuron maintains a dynamical state which is not communicated to other neurons.
Communication between neurons only consists of spikes, emitted when the state reaches a threshold.
Spikes are spontaneous, stereotypical events: the information resides in the time at which the spike was emitted.
Spikes and address events from the Dynamic Vision Sensors are very similar.
In fact, to process address events with a spiking network, the first layer of neurons simply emit spikes for every address event.

<div class="wrapper-md">
    <img src="/images/PhD/spiking-neuron.png" alt="Spiking neuron" title="Spiking neuron"/>
</div>

Intuitively, processing a stream of events with a spiking network requires far less computations/communications than processing a sequence of images with conventional neural networks.
The reason is, instead of processing the full information about the visual scene at every timestep again and again, we only process and communicate  differences.
Less computations implies potentially saving energy while running faster.
However, this style of event-based computations do not fit conventional hardware very well: it is a paradigm shift.
Taking advantage of these saved computations requires dedicated hardware, called neuromorphic hardware.
Designing and manufacturing neuromorphic hardware was initiated in academia (Caltech, Zürich, Manchester, Dresden, ...).
Since then, both [Intel](https://www.intel.fr/content/www/fr/fr/research/neuromorphic-computing.html) and [IBM](https://www.research.ibm.com/articles/brain-chip.shtml) started developing neuromorphic chips.

Deep learning networks are used everyhere today, because they can be easily trained with an algorithm called backpropagation.
But how can deep networks of spiking neurons be trained?
Are such networks capable of learning directly from the events of the Dynamic Vision Sensor?
These are the research questions I have investigated as part of my PhD.

Over all the methods I have evaluated during my research, approximations of backpropagation turned out to provide the best performance.
Not only was it learning more accurately, these approximations also solve some of the biologically implausible aspects of backpropagation, yielding a new theory of learning in the brain.

To go further, let's formalize what is learning.
Machine learning is a neat technique to solve complex problems, whenever data is available.
At its core, learning requires:
- a parametrized model: some mathematical operations, which is a function of some parameters. In deep learning, the neural network is the model, which is parametrized the synaptic weights $$W$$.
- a loss function $$L(W)$$: a metric assessing the performance of the neural network on the task, with respect to the available data.
The goal of learning is to find the weights which minimize this loss function: $$W^\star=argmin_W L(W)$$.

To find the best weights, a naive approach would be to try random weights, evaluate the loss function, and simply keep the ones where the loss function is the smallest.
In fact, this is what evolutionary algorithms are doing.
However, evaluating the loss function is expensive, and we would like to do it as few times as possible.
To this end, the best method to date is gradient descent.
Aside from evaluating the loss function $$L(W)$$, gradient descent computes the gradient of the loss by the weights $$\frac{\partial L}{\partial w_{ij}}$$, for all weights $$w_{ij} \in W$$.
By definition, the gradient of the loss by a weight $$w_{ij}$$ means **how an infinitesimal change in $$w_{ij}$$ affects the loss**.
Therefore, by taking an (arbitrary small) step in the opposite direction of the gradient, we have good chances that the loss function will decrease.
Gradient descent repeats this operation for all weights $$w_{ij} \in W$$ until the loss function stop decreasing:

$$ w_{ij} \leftarrow w_{ij} - \alpha \times \frac{\partial L}{\partial w_{ij}},$$

with $$\alpha \in \mathbb{R}$$ the learning rate: an arbitrary small step.

Now back to neural networks.
Backpropagation is the method used to compute the gradient $$\frac{\partial L}{\partial w_{ij}}$$ in deep learning.
To explain how it works, we need to dive in the equations of artificial neurons.
A conventional deep learning neuron starts by summing up the values of its pre-synaptic neurons, weighted by the synaptic weights.
It then applies a non-linear function -- usually simply nulling-out negative values (ReLU).
These are the $$u_i$$ and the $$s_i$$ terms in the following neural equations:

$$
\begin{aligned}
    u_i &= \sum_{j\in \textit{pre}} w_{ij} \times s_j \\
    s_i &= u_i \: \text{if} \: u_i > 0 \: \text{else} \: 0
\end{aligned}
$$

Using this definition, the step from a conventional neuron to a spiking neuron is small.
In fact, there are only two important differences with spiking neurons:
- $$ u_i $$ carries a trace of the previous activity (it is a state which needs to be stored and updated)
- $$ s_i $$ is a threshold function (0 or 1, denoting spike and absence of spike, unlike conventional neuron which have output in $$\mathbb{R}$$)
Mathematically, a spiking neuron can be expressed as:

$$
\begin{aligned}
    u_i &= \sum_{j\in \textit{pre}} w_{ij} (\epsilon \ast s_j + \eta \ast s_i \\
    s_i (t) &= 1 \: \text{if} \: u_i > 0 \: \text{else} \: 0
\end{aligned}
$$

The $$\ast$$ operation denotes a temporal convolution, with kernel $$\epsilon$$ and $$\eta$$ respectively.
In words, this means that $$u_i$$ also depends on the previous values of $$s_j$$ and $$s_i$$.

{:.wrapper-md}
![Example dynamics of a spiking neuron](/images/PhD/all_dynamics.png "Example dynamics of a spiking neuron")

The resulting learning rule developed with Prof. Emre Neftci at the University of California, Irvine,  [DECOLLE](https://github.com/nmi-lab/decolle-public) - the learning rule approximating backpropagation -
