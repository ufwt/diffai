DiffAI v2.0 <a href="https://www.sri.inf.ethz.ch/"><img width="100" alt="portfolio_view" align="right" src="http://safeai.ethz.ch/img/sri-logo.svg"></a>
=============================================================================================================

![High Level](https://raw.githubusercontent.com/eth-sri/diffai/v2/media/overview.png)

DiffAI is a system for training neural networks to be provably robust and for proving that they are robust.

Background
----------

By now, it is well known that otherwise working networks can be tricked by clever attacks. For example [Goodfellow et al.](https://arxiv.org/abs/1412.6572) demonstrated a network with high classification accuracy which classified one image of a panda correctly, and a seemingly identical attack picture 
incorrectly.  Many defenses against this type of attack have been produced, but very few produce networks for which *provably* verifying the safety of a prediction is feasible.

Abstract Interpretation is a technique for verifying properties of programs by soundly overapproximating their behavior.  When applied to neural networks, an infinite set (a ball) of possible inputs is passed to an approximating "abstract" network
to produce a superset of the possible outputs from the actual network.  Provided an appropreate representation for these sets, demonstrating that the network classifies everything in the ball correctly becomes a simple task. The method used to represent these sets is the abstract domain, and the specific approximations are the abstract transformers.  

In DiffAI, the entire abstract interpretation process is programmed using PyTorch so that it is differentiable and can be run on the GPU, 
and a loss function is crafted so that low values correspond to inputs which can be proved safe (robust).

Requirements 
------------

python 3.5 or higher, pip3, and virtualenv

Recommended Setup 
-----------------

```
$ git clone https://github.com/eth-sri/DiffAI.git
$ cd DiffAI
$ virtualenv pytorch --python python3.6 # or whatever version you are using
$ source pytorch/bin/activate
(pytorch) $ pip install -r requirements.txt
```

Note: you need to activate your virtualenv every time you start a new shell.

Getting Started
---------------

DiffAI can be run as a standalone program.  To see a list of arguments, type 

```
(pytorch) $ python . --help
```

At the minimum, DiffAI expects at least one domain to train with and one domain to test with, and a network with which to test.  For example, to train with the Box domain, baseline training (Point) and test against the FGSM attack and the ZSwitch domain with a simple feed forward network on the MNIST dataset (default, if none provided), you would type:

```
(pytorch) $ python . -d "Point()" -d "Box()" -t "PGD()" -t "ZSwitch()" -n ffnn
```

Unless otherwise specified by "--out", the output is logged to the folder "out/".  
In the folder corresponding to the experiment that has been run, one can find the saved configuration options in 
"config.txt", and a pickled net which is saved every 10 epochs (provided that testing is set to happen every 10th epoch).

To load a saved model, use "--test" as per the example:

```
(pytorch) $ alias test-diffai="python . -d Point --epochs 1 --dont-write --test-freq 1"
(pytorch) $ test-diffai -t Box --update-test-net-name convBig --test PATHTOSAVED_CONVBIG.pynet --width 0.1 --test-size 500 --test-batch-size 500
```

Note that "--update-test-net-name" will create a new model based on convBig and try to use the weights in the pickled PATHTOSAVED_CONVBIG.pynet to initialize that models weights.  This is not always necessary, but is useful when the code for a model changes (in components) but does not effect the number or usage of weight, or when loading a model pickled by a cuda process into a cpu process.

The default specification type is the L_infinity Ball specified explicitly by "--spec boxSpec",
which uses an epsilon specified by "--width"

The default specification type is the L_infinity Ball specified explicitly by "--spec boxSpec", 
which uses an epsilon specified by "--width"




Training Domain DSL
-------------------

In DiffAI version 2.0, a dsl has been provided to specify arbitrary training domains. In particular, it is now possible to train on combinations of attacks and abstract domains on specifications defined by attacks. Specifying training domains is possible in the command line using ```-d "DOMAIN_INITIALIZATION"```.  The possible combinations are the classes listed in domains.py. The same syntax is also supported for testing domains, to allow for testing robustness with different epsilon-sized attacks and specifications.

Listed below are a few examples:

* ```-t "PGD(k=4, w=0.1)" -t "ZNIPS(w=0.3)" ``` Will first test with the PGD attack with an epsilon=w=0.1 and, the number of iterations k=4 and step size set to w/k.  It will also test with the zonotope domain using the transformer specified in our [NIPS 2018 paper](https://www.sri.inf.ethz.ch/publications/singh2018effective) with an epsilon=w=0.3.

* ```-t "PGDK(r=3,k=16,restart=2, w=0.1)"``` tests on points found using PGD with a step size of r*w/k and two restarts, and an attack-generated specification.

* ```-d Point()``` is standard non-defensive training.

* ```-d "Mix(a=PGD(), b=Box(), aw=1, bw=0.1)"``` trains on points produced by pgd with the default parameters listed in domains.py, and points produced using the box domain.  The loss is combined linearly using the weights aw and bw and scaled by 1/(aw + bw). The epsilon used for both is the ambient epsilon specified with "--width".

* ```-d "DList((PGD(w=0.1),1), (Box(w=0.01),0.1), (Box(w=0.1),0.01))"``` is a generalization of the Mix domain allowing for training with arbitrarily many domains at once weighted by the given values (the resulting loss is scaled by the inverse of the sum of weights).

* ```-d "AdvDom(a=PGD(), b=Box())"``` trains using the Box domain, but constructs specifications as L∞ balls containing the PGD attack image and the original image "o".  

* ```-d "BiAdv(a=PGD(), b=Box())"``` is similar, but creates specifications between the pgd attack image "a" and "o - (a - o)".

One domain we have found particularly useful for training is ```Mix(a=PGDK(r=3,k=16,restart=2, w=0.1), b=BiAdv(a=PGD(k=5, w=0.05)), bw=0.1)```.

While the above domains are all deterministic (up to gpu error and shuffling orders), we have also implemented nondeterministic training domains:

* ```-d "Coin(a=PGD(), b=Box(), aw=1, bw=0.1)"``` is like Mix, but chooses which domain to train a batch with by the probabilities determined by aw / (aw + bw) and bw / (aw + bw).

* ```-d "DProb((PGD(w=0.1),1), (Box(w=0.01),0.1), (Box(w=0.1),0.01))"``` is to Coin what DList is to Mix.

* ```-d AdvDom(a=PGD(), b=DList((PointB(),1), (PointA(), 1), (Box(), 0.2)))``` can be used to share attack images between multiple training types.  Here an attack image "m" is found using PGD, then both the original image "o" and the attack image "m" are passed to DList which trains using three different ways:  PointA trains with "o", PointB trains with "m", and Box trains on the box produced between them.  This can also be used with Mix.

* ```-d Normal(w=0.3)``` trains using images sampled from a normal distribution around the provided image using standard deviation w.

* ```-d NormalAdv(a=PGD(), w=0.3)``` trains using PGD (but this could be an abstract domain) where perturbations are constrained to a box determined by a normal distribution around the original image with standard deviation w.

There are more domains implemented than listed here, and of course more interesting combinations are possible.  Please look carefully at domains.py for default values and further options.

Contents
--------

* components.py: A high level neural network library for composable layers and operations
* domains.py: abstract domains and attacks which can be used as a drop in replacement for pytorch tensors in any model built with components from components.py
* losses.py: high level loss functions for training which can take advantage of abstract domains.
* models.py: A repository of models to train with which are used in the paper.
* \_\_main\_\_.py: The entry point to run the experiments.
* helpers.py: Assorted helper functions.  Does some monkeypatching, so you might want to be careful importing our library into your project.

Notes
-----

Not all of the datasets listed in the help message are supported.  Supported datasets are:

* CIFAR10
* CIFAR100
* MNIST
* SVHN
* FashionMNIST

Unsupported datasets will not necessarily throw errors.

About
-----

* DiffAI is now on Version 2.0.  
* This repository contains the code used for the experiments in the 2018 ICML paper, [Differentiable Abstract Interpretation for Provably Robust Neural Networks](https://files.sri.inf.ethz.ch/website/papers/icml18-diffai.pdf).  To reproduce these experiments exactly, one must download the source from download the [source code for Version 1.0](https://github.com/eth-sri/diffai/releases/tag/v1.0)
* Further information and related projects can be found at [the SafeAI Project](http://safeai.ethz.ch/)
* [High level slides](https://files.sri.inf.ethz.ch/website/slides/mirman2018differentiable.pdf)

Citing This Framework
---------------------

```
@inproceedings{
  title={Differentiable Abstract Interpretation for Provably Robust Neural Networks},
  author={Mirman, Matthew and Gehr, Timon and Vechev, Martin},
  booktitle={International Conference on Machine Learning (ICML)},
  year={2018},
  url={https://www.icml.cc/Conferences/2018/Schedule?showEvent=2477},
}
```

Contributors
------------

* [Matthew Mirman](https://www.mirman.com) - matt@mirman.com
* [Timon Gehr](https://www.sri.inf.ethz.ch/tg.php) - timon.gehr@inf.ethz.ch
* Marc Fischer - marcfisc@student.ethz.ch
* [Gagandeep Singh](https://www.sri.inf.ethz.ch/people/gagandeep) - gsingh@inf.ethz.ch
* [Martin Vechev](https://www.sri.inf.ethz.ch/vechev.php) - martin.vechev@inf.ethz.ch

License and Copyright
---------------------

* Copyright (c) 2018 [Secure, Reliable, and Intelligent Systems Lab (SRI), ETH Zurich](https://www.sri.inf.ethz.ch/)
* Licensed under the [MIT License](https://opensource.org/licenses/MIT)
