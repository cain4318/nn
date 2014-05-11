<a name="nn.dok"/>
# Neural Network Package #

This package provides an easy way to build and train simple or complex
neural networks.

Each module of a network is composed of [Modules](doc/module.md#nn.Modules) and there
are several sub-classes of `Module` available: container classes like
[Sequential](doc/containers.md#nn.Sequential), [Parallel](doc/containers.md#nn.Parallel) and
[Concat](doc/containers.md#nn.Concat) , which can contain simple layers like
[Linear](doc/simple.md#nn.Linear), [Mean](doc/simple.md#nn.Mean), [Max](doc/simple.md#nn.Max) and
[Reshape](doc/simple.md#nn.Reshape), as well as [convolutional layers](doc/convolution.md), and [transfer
functions](doc/transfer.md) like [Tanh](doc/transfer.md#nn.Tanh).

Loss functions are implemented as sub-classes of
[Criterion](doc/criterion.md#nn.Criterions). They are helpful to train neural network on
classical tasks.  Common criterions are the Mean Squared Error
criterion implemented in [MSECriterion](doc/criterion.md#nn.MSECriterion) and the
cross-entropy criterion implemented in
[ClassNLLCriterion](doc/criterion.md#nn.ClassNLLCriterion).

Finally, the [StochasticGradient](doc/training.md#nn.StochasticGradient) class provides a
high level way to train the neural network of choice, even though it is
easy with a simple for loop to [train a neural network yourself](doc/training.md#nn.DoItYourself).

For those who want to implement their own modules, we suggest using
the `nn.Jacobian` class for testing the derivatives of their class,
together with the [torch.Tester](https://github.com/torch/torch7/blob/master/doc/tester.md) class. The sources
of `nn` package contains sufficiently many examples of such tests.


<a name="nn.overview.dok"/>
# Detailed Overview of the Neural Network Package #

__Module__

A neural network is called a [Module](doc/module.md#nn.Module) (or simply
_module_ in this documentation) in Torch. `Module` is an abstract
class which defines four main methods:
  * [forward(input)](doc/module.md#nn.Module.forward) which computes the output of the module given the `input` [Tensor](https://github.com/torch/torch7/blob/master/doc/tensor.md).
  * [backward(input, gradOutput)](doc/module.md#nn.Module.backward) which computes the gradients of the module with respect to its own parameters, and its own inputs.
  * [zeroGradParameters()](doc/module.md#nn.Module.zeroGradParameters) which zeroes the gradient with respect to the parameters of the module.
  * [updateParameters(learningRate)](doc/module.md#nn.Module.updateParameters) which updates the parameters after one has computed the gradients with `backward()`

It also declares two members:
  * [output](doc/module.md#nn.Module.output) which is the output returned by `forward()`.
  * [gradInput](doc/module.md#nn.Module.gradInput) which contains the gradients with respect to the input of the module, computed in a `backward()`.

Two other perhaps less used but handy methods are also defined:
  * [share(mlp,s1,s2,...,sn)](doc/module.md#nn.Module.share) which makes this module share the parameters s1,..sn of the module `mlp`. This is useful if you want to have modules that share the same weights.
  * [clone(...)](doc/module.md#nn.Module.clone) which produces a deep copy of (i.e. not just a pointer to) this Module, including the current state of its parameters (if any).

Some important remarks:
  * `output` contains only valid values after a [forward(input)](doc/module.md#nn.Module.forward).
  * `gradInput` contains only valid values after a [backward(input, gradOutput)](doc/module.md#nn.Module.backward).
  * [backward(input, gradOutput)](doc/module.md#nn.Module.backward) uses certain computations obtained during [forward(input)](doc/module.md#nn.Module.forward). You _must_ call `forward()` before calling a `backward()`, on the _same_ `input`, or your gradients are going to be incorrect!


__Plug and play__

Building a simple neural network can be achieved by constructing an available layer.
A linear neural network (perceptron!) is built only in one line:
```lua
mlp = nn.Linear(10,1) -- perceptron with 10 inputs
```

More complex neural networks are easily built using container classes
[Sequential](doc/containers.md#nn.Sequential) and [Concat](doc/containers.md#nn.Concat). `Sequential` plugs
layer in a feed-forward fully connected manner. `Concat` concatenates in
one layer several modules: they take the same inputs, and their output is
concatenated.

Creating a one hidden-layer multi-layer perceptron is thus just as easy as:
```lua
mlp = nn.Sequential()
mlp:add( nn.Linear(10, 25) ) -- 10 input, 25 hidden units
mlp:add( nn.Tanh() ) -- some hyperbolic tangent transfer function
mlp:add( nn.Linear(25, 1) ) -- 1 output
```

Of course, `Sequential` and `Concat` can contains other
`Sequential` or `Concat`, allowing you to try the craziest neural
networks you ever dreamt of! See the [[#nn.Modules|complete list of
available modules]].

__Training a neural network__

Once you built your neural network, you have to choose a particular
[Criterion](doc/criterion.md#nn.Criterions) to train it. A criterion is a class which
describes the cost to be minimized during training.

You can then train the neural network by using the
[StochasticGradient](doc/training.md#nn.StochasticGradient) class.

```lua
 criterion = nn.MSECriterion() -- Mean Squared Error criterion
 trainer = nn.StochasticGradient(mlp, criterion)
 trainer:train(dataset) -- train using some examples
```

StochasticGradient expect as a `dataset` an object which implements
the operator `dataset[index]` and implements the method
`dataset:size()`. The `size()` methods returns the number of
examples and `dataset[i]` has to return the i-th example.

An `example` has to be an object which implements the operator
`example[field]`, where `field` might take the value `1` (input
features) or `2` (corresponding label which will be given to the
criterion).  The input is usually a Tensor (except if you use special
kind of gradient modules, like [table layers](doc/table.md#nn.TableLayers)). The
label type depends of the criterion.  For example, the
[MSECriterion](doc/criterion.md#nn.MSECriterion) expect a Tensor, but the
[ClassNLLCriterion](doc/criterion#nn.ClassNLLCriterion) except a integer number (the
class).

Such a dataset is easily constructed by using Lua tables, but it could
any `C` object for example, as long as required operators/methods
are implemented.  [See an example](doc/containers.md#nn.DoItStochasticGradient).

`StochasticGradient` being written in `Lua`, it is extremely easy
to cut-and-paste it and create a variant to it adapted to your needs
(if the constraints of `StochasticGradient` do not satisfy you).

__Low Level Training Of a Neural Network__

If you want to program the `StochasticGradient` by hand, you
essentially need to control the use of forwards and backwards through
the network yourself.  For example, here is the code fragment one
would need to make a gradient step given an input `x`, a desired
output `y`, a network `mlp` and a given criterion `criterion`
and learning rate `learningRate`:

```lua
function gradUpdate(mlp, x, y, criterion, learningRate) 
  local pred = mlp:forward(x)
  local err = criterion:forward(pred, y)
  local gradCriterion = criterion:backward(pred, y)
  mlp:zeroGradParameters()
  mlp:backward(x, gradCriterion)
  mlp:updateParameters(learningRate)
end
```
For example, if you wish to use your own criterion you can simple replace 
`gradCriterion` with the gradient vector of your criterion of choice.








