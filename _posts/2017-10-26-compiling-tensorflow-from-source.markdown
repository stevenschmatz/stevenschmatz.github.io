---
layout: post
title: Compiling TensorFlow From Source
date: 2017-10-26 17:24
comments: true
external-url: compiling-tensorflow-from-source
categories:
- Machine Learning
comments: true
---

If you are new to TensorFlow, you may have seen this before:

```
The TensorFlow library wasn't compiled to use SSE4.2 instructions, but these are available on your machine and could speed up CPU computations.
The TensorFlow library wasn't compiled to use AVX instructions, but these are available on your machine and could speed up CPU computations.
The TensorFlow library wasn't compiled to use AVX2 instructions, but these are available on your machine and could speed up CPU computations.
The TensorFlow library wasn't compiled to use FMA instructions, but these are available on your machine and could speed up CPU computations.
```

The TensorFlow authors wanted to build a binary that would support _as many machines as possible_, which also means that the code runs sub-optimally on individual machines like mine.
Many machines support instruction sets like SSE, AVX, and FMA, which provide floating-point operations, vector operations, and fused multiply-add operations, all of which are relevant for computation graph frameworks like TensorFlow.
They need to be applied by building TensorFlow from source, and by setting some compiler options.

Using these instruction sets is actually hugely valuable for beginners, if you run lots of experiments on your machine, and don't want to fork out hundreds of dollars for an EC2 or Google Cloud instance.
Even though I only experienced a roughly 2x speedup on my benchmarks, **some report 4x to 8x speedups**. Even though this is clearly hugely valuable, all of the tutorials I could find were way more complex than they needed to be.

*For reference, I'm using a 2016 Macbook Pro on macOS Sierra.*

### Prerequisites

You will need to install [Bazel](https://www.bazel.build/), an open-source build and test tool.
Install instructions are [here](https://docs.bazel.build/versions/master/install-os-x.html).

Also, you will need to know what instruction sets your machine supports. To find this, open up a Python interpreter and run:

```
>>> import tensorflow as tf
>>> tf.Session()
The TensorFlow library wasn't compiled to use SSE4.2 instructions, but these are available on your machine and could speed up CPU computations.
The TensorFlow library wasn't compiled to use AVX instructions, but these are available on your machine and could speed up CPU computations.
The TensorFlow library wasn't compiled to use AVX2 instructions, but these are available on your machine and could speed up CPU computations.
The TensorFlow library wasn't compiled to use FMA instructions, but these are available on your machine and could speed up CPU computations.
```

Note these for later.

## Installing TensorFlow from source

1. Clone the TensorFlow repository.

```bash
git clone https://github.com/tensorflow/tensorflow
cd tensorflow
```

2. Earlier, we found which architectures your machine supports.
Construct a space-separated string, like the following:

```bash
--copt=-mavx --copt=-mavx2 --copt=-mfma --copt=-msse4.2
```

* `--copt=-mavx` corresponds to AVX instructions
* `--copt=-mavx2` corresponds to AVX2 instructions
* `--copt=-mfma` corresponds to FMA instructions
* `--copt=-msse4.2` corresponds to SSE4.2 instructions
* Also, if you want to have GPU support, you will need to include CUDA. Add `--config=cuda` here.

If your system does not support one of these instruction types, **do not include it here**.

Now, run `./configure`. You can leave all of the fields blank, except for the last one.

```
Please specify the location of python. [Default is /anaconda/bin/python]:
Please input the desired Python library path to use.  Default is [/anaconda/lib/python3.6/site-packages]
Do you wish to build TensorFlow with Google Cloud Platform support? [Y/n]:
Do you wish to build TensorFlow with Hadoop File System support? [Y/n]:
Do you wish to build TensorFlow with Amazon S3 File System support? [Y/n]:
Do you wish to build TensorFlow with XLA JIT support? [y/N]:
Do you wish to build TensorFlow with GDR support? [y/N]:
Do you wish to build TensorFlow with VERBS support? [y/N]:
Do you wish to build TensorFlow with OpenCL support? [y/N]:
Do you wish to build TensorFlow with CUDA support? [y/N]: <SET THIS TO YES IF YOU WANT GPU SUPPORT>
Do you wish to build TensorFlow with MPI support? [y/N]:
Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]: <ENTER THE COMMAND ABOVE>
```

Once this is done, we're ready to build TensorFlow.

```bash
bazel build -c opt <YOUR OPTIONS STRING HERE> -k //tensorflow/tools/pip_package:build_pip_package
```

This process will take a long time. On my machine, it took 32 minutes.
Hold tight; the time saved during runtime will make it worth it.
My compilation ran without errors, and with a few warnings.
If you have any errors, resolve them before wasting half an hour, because TensorFlow won't compile.

3. Once the compilation is done, we must build the wheel so `pip` can correctly install it.
To do that, run the following command in the `tensorflow` directory:

```bash
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg\n
```

And finally, to install `tensorflow` using `pip` so your Python interpreter has access to it, run:

```bash
sudo pip install --upgrade /tmp/tensorflow_pkg/tensorflow-*.whl
```

Now, you should be all set! To test your installation, run our test in your Python interpreter again:

```python
>>> import tensorflow as tf
>>> tf.Session()
>>>
```

If you see no warnings after the `tf.Session()` line, you're all set!

## Benchmarks

| Benchmark | Time with default installation (s) | Time with compilation from source (s) | Speedup |
| --------- | ---------------------------------- | ------------------------------------- | ------- |
| [Simple net](https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/3_NeuralNetworks/neural_network_raw.py) | 2.59 sec | 1.34 sec | 1.93x |
| [Convolutional net](https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/3_NeuralNetworks/convolutional_network_raw.py) | 52.94 sec | 36.41 sec | 1.45x |
| [LSTM](https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/3_NeuralNetworks/recurrent_network.py) | 532.20 sec | 337.80 sec | 1.57x |
| [Bi-directional LSTM](https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/3_NeuralNetworks/bidirectional_rnn.py) | 575.32 sec | 335.95 sec | 1.71x |
| [Dynamic LSTM](https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/3_NeuralNetworks/dynamic_rnn.py) | 369.1 sec | 260.4 sec | 1.41x |

Although most of the speedups are in the 1.5x-2x range, which may seem just marginal, for large jobs this can save hours.
I'm of the belief that speeding up your iteration times can be one of the most valuable things you can do for your workflow and learning.
By tightening up the feedback loop, it's much easier to gain intuition on neural network design and debugging.
