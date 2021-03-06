API - Layers
=========================

To make TensorLayer simple, we minimize the number of layer classes as much as
we can. So we encourage you to use TensorFlow's function.
For example, we do not provide layer for local response normalization, we suggest
you to apply ``tf.nn.lrn`` on ``network.outputs``.
More functions can be found in `TensorFlow API <https://www.tensorflow.org/versions/master/api_docs/index.html>`_


Understand layer
-----------------

All TensorLayer layers have a number of properties in common:

 - ``layer.outputs`` : Tensor, the outputs of current layer.
 - ``layer.all_params`` : a list of Tensor, all network variables in order.
 - ``layer.all_layers`` : a list of Tensor, all network outputs in order.
 - ``layer.all_drop`` : a dictionary of {placeholder : float}, all keeping probabilities of noise layer.

All TensorLayer layers have a number of methods in common:

 - ``layer.print_params()`` : print the network variables information in order (after ``sess.run(tf.initialize_all_variables())``). alternatively, print all variables by ``tl.layers.print_all_variables()``.
 - ``layer.print_layers()`` : print the network layers information in order.
 - ``layer.count_params()`` : print the number of parameters in the network.



The initialization of a network is done by input layer, then we can stacked layers
as follow, a network is a ``Layer`` class.
The most important properties of a network are ``network.all_params``, ``network.all_layers`` and ``network.all_drop``.
The ``all_params`` is a list which store all pointers of all network parameters in order,
the following script define a 3 layer network, then:

``all_params`` = [W1, b1, W2, b2, W_out, b_out]

The ``all_layers`` is a list which store all pointers of the outputs of all layers,
in the following network:

``all_layers`` = [drop(?,784), relu(?,800), drop(?,800), relu(?,800), drop(?,800)], identity(?,10)]

where ``?`` reflects any batch size. You can print the layer information and parameters information by
using ``network.print_layers()`` and ``network.print_params()``.
To count the number of parameters in a network, run ``network.count_params()``.



.. code-block:: python

  sess = tf.InteractiveSession()

  x = tf.placeholder(tf.float32, shape=[None, 784], name='x')
  y_ = tf.placeholder(tf.int64, shape=[None, ], name='y_')

  network = tl.layers.InputLayer(x, name='input_layer')
  network = tl.layers.DropoutLayer(network, keep=0.8, name='drop1')
  network = tl.layers.DenseLayer(network, n_units=800,
                                  act = tf.nn.relu, name='relu1')
  network = tl.layers.DropoutLayer(network, keep=0.5, name='drop2')
  network = tl.layers.DenseLayer(network, n_units=800,
                                  act = tf.nn.relu, name='relu2')
  network = tl.layers.DropoutLayer(network, keep=0.5, name='drop3')
  network = tl.layers.DenseLayer(network, n_units=10,
                                  act = tl.activation.identity,
                                  name='output_layer')

  y = network.outputs
  y_op = tf.argmax(tf.nn.softmax(y), 1)

  cost = tl.cost.cross_entropy(y, y_)

  train_params = network.all_params

  train_op = tf.train.AdamOptimizer(learning_rate, beta1=0.9, beta2=0.999,
                              epsilon=1e-08, use_locking=False).minimize(cost, var_list = train_params)

  sess.run(tf.initialize_all_variables())

  network.print_params()
  network.print_layers()

In addition, ``network.all_drop`` is a dictionary which stores the keeping probabilities of all
noise layer. In the above network, they are the keeping probabilities of dropout layers.

So for training, enable all dropout layers as follow.

.. code-block:: python

  feed_dict = {x: X_train_a, y_: y_train_a}
  feed_dict.update( network.all_drop )
  loss, _ = sess.run([cost, train_op], feed_dict=feed_dict)
  feed_dict.update( network.all_drop )

For evaluating and testing, disable all dropout layers as follow.

.. code-block:: python

  feed_dict = {x: X_val, y_: y_val}
  feed_dict.update(dp_dict)
  print("   val loss: %f" % sess.run(cost, feed_dict=feed_dict))
  print("   val acc: %f" % np.mean(y_val ==
                          sess.run(y_op, feed_dict=feed_dict)))

For more details, please read the MNIST examples.

Creating custom layers
------------------------

Understand Dense layer
^^^^^^^^^^^^^^^^^^^^^^^^^

Before creating your own TensorLayer layer, let's have a look at Dense layer.
It creates a weights matrix and biases vector if not exists, then implement
the output expression.
At the end, as a layer with parameter, we also need to append the parameters into ``all_params``.


.. code-block:: python

  class DenseLayer(Layer):
      """
      The :class:`DenseLayer` class is a fully connected layer.

      Parameters
      ----------
      layer : a :class:`Layer` instance
          The `Layer` class feeding into this layer.
      n_units : int
          The number of units of the layer.
      act : activation function
          The function that is applied to the layer activations.
      W_init : weights initializer
          The initializer for initializing the weight matrix.
      b_init : biases initializer
          The initializer for initializing the bias vector. If None, skip biases.
      W_init_args : dictionary
          The arguments for the weights tf.get_variable.
      b_init_args : dictionary
          The arguments for the biases tf.get_variable.
      name : a string or None
          An optional name to attach to this layer.
      """
      def __init__(
          self,
          layer = None,
          n_units = 100,
          act = tf.nn.relu,
          W_init = tf.truncated_normal_initializer(stddev=0.1),
          b_init = tf.constant_initializer(value=0.0),
          W_init_args = {},
          b_init_args = {},
          name ='dense_layer',
      ):
          Layer.__init__(self, name=name)
          self.inputs = layer.outputs
          if self.inputs.get_shape().ndims != 2:
              raise Exception("The input dimension must be rank 2")
          n_in = int(self.inputs._shape[-1])
          self.n_units = n_units
          print("  tensorlayer:Instantiate DenseLayer %s: %d, %s" % (self.name, self.n_units, act))
          with tf.variable_scope(name) as vs:
              W = tf.get_variable(name='W', shape=(n_in, n_units), initializer=W_init, **W_init_args )
              if b_init:
                  b = tf.get_variable(name='b', shape=(n_units), initializer=b_init, **b_init_args )
                  self.outputs = act(tf.matmul(self.inputs, W) + b)#, name=name)
              else:
                  self.outputs = act(tf.matmul(self.inputs, W))

          # Hint : list(), dict() is pass by value (shallow).
          self.all_layers = list(layer.all_layers)
          self.all_params = list(layer.all_params)
          self.all_drop = dict(layer.all_drop)
          self.all_layers.extend( [self.outputs] )
          if b_init:
             self.all_params.extend( [W, b] )
          else:
             self.all_params.extend( [W] )

A simple layer
^^^^^^^^^^^^^^^

To implement a custom layer in TensorLayer, you will have to write a Python class
that subclasses Layer and implement the ``outputs`` expression.

The following is an example implementation of a layer that multiplies its input by 2:

.. code-block:: python

  class DoubleLayer(Layer):
      def __init__(
          self,
          layer = None,
          name ='double_layer',
      ):
          Layer.__init__(self, name=name)
          self.inputs = layer.outputs
          self.outputs = self.inputs * 2

          self.all_layers = list(layer.all_layers)
          self.all_params = list(layer.all_params)
          self.all_drop = dict(layer.all_drop)
          self.all_layers.extend( [self.outputs] )


Modifying Pre-train Behaviour
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Greedy layer-wise pretrain is an important task for deep neural network
initialization, while there are many kinds of pre-train methods according
to different network architectures and applications.

For example, the pre-train process of `Vanilla Sparse Autoencoder <http://deeplearning.stanford.edu/wiki/index.php/Autoencoders_and_Sparsity>`_
can be implemented by using KL divergence (for sigmoid) as the following code,
but for `Deep Rectifier Network <http://www.jmlr.org/proceedings/papers/v15/glorot11a/glorot11a.pdf>`_,
the sparsity can be implemented by using the L1 regularization of activation output.

.. code-block:: python

  # Vanilla Sparse Autoencoder
  beta = 4
  rho = 0.15
  p_hat = tf.reduce_mean(activation_out, reduction_indices = 0)
  KLD = beta * tf.reduce_sum( rho * tf.log(tf.div(rho, p_hat))
          + (1- rho) * tf.log((1- rho)/ (tf.sub(float(1), p_hat))) )


There are many pre-train methods, for this reason, TensorLayer provides a simple way to modify or design your
own pre-train method. For Autoencoder, TensorLayer uses ``ReconLayer.__init__()``
to define the reconstruction layer and cost function, to define your own cost
function, just simply modify the ``self.cost`` in ``ReconLayer.__init__()``.
To creat your own cost expression please read `Tensorflow Math <https://www.tensorflow.org/versions/master/api_docs/python/math_ops.html>`_.
By default, ``ReconLayer`` only updates the weights and biases of previous 1
layer by using ``self.train_params = self.all _params[-4:]``, where the 4
parameters are ``[W_encoder, b_encoder, W_decoder, b_decoder]``, where
``W_encoder, b_encoder`` belong to previous DenseLayer, ``W_decoder, b_decoder``
belong to this ReconLayer.
In addition, if you want to update the parameters of previous 2 layers at the same time, simply modify ``[-4:]`` to ``[-6:]``.


.. code-block:: python

  ReconLayer.__init__(...):
      ...
      self.train_params = self.all_params[-4:]
      ...
  	self.cost = mse + L1_a + L2_w











.. automodule:: tensorlayer.layers

.. autosummary::

   Layer
   InputLayer
   Word2vecEmbeddingInputlayer
   EmbeddingInputlayer
   DenseLayer
   ReconLayer
   DropoutLayer
   DropconnectDenseLayer
   Conv2dLayer
   DeConv2dLayer
   Conv3dLayer
   DeConv3dLayer
   PoolLayer
   BatchNormLayer
   RNNLayer
   DynamicRNNLayer
   FlattenLayer
   ConcatLayer
   ReshapeLayer
   SlimNetsLayer
   MultiplexerLayer
   EmbeddingAttentionSeq2seqWrapper
   flatten_reshape
   clear_layers_name
   set_name_reuse
   print_all_variables
   initialize_rnn_state


Basic layer
-----------

.. autoclass:: Layer

Input layer
------------

.. autoclass:: InputLayer
  :members:

Word Embedding Input layer
-----------------------------

Word2vec layer for training
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: Word2vecEmbeddingInputlayer

Embedding Input layer
^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: EmbeddingInputlayer

Dense layer
------------

Dense layer
^^^^^^^^^^^^^

.. autoclass:: DenseLayer

Reconstruction layer for Autoencoder
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: ReconLayer
   :members:

Noise layer
------------

Dropout layer
^^^^^^^^^^^^^^^^

.. autoclass:: DropoutLayer

Dropconnect + Dense layer
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: DropconnectDenseLayer

Convolutional layer
----------------------

1D Convolutional layer
^^^^^^^^^^^^^^^^^^^^^^^^

We don't provide 1D CNN layer, actually TensorFlow only provides `tf.nn.conv2d` and `tf.nn.conv3d`,
so to implement 1D CNN, you can use Reshape layer as follow.

.. code-block:: python

  x = tf.placeholder(tf.float32, shape=[None, 500], name='x')
  network = tl.layers.ReshapeLayer(x, shape=[-1, 500, 1, 1], name='reshape')
  network = tl.layers.Conv2dLayer(network,
                      act = tf.nn.relu,
                      shape = [10, 1, 1, 16], # 16 features
                      strides=[1, 2, 1, 1],   # stride of 2
                      padding='SAME',
                      name = 'cnn')



2D Convolutional layer
^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: Conv2dLayer

2D Deconvolutional layer
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: DeConv2dLayer


3D Convolutional layer
^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: Conv3dLayer

3D Deconvolutional layer
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: DeConv3dLayer

Pooling layer
----------------

Pooling layer for any dimensions and any pooling functions

.. autoclass:: PoolLayer

Normalization layer
--------------------

We do not provide layer for local response normalization as it does not have any weights and arguments, we suggest
you to apply ``tf.nn.lrn`` on ``network.outputs``.

.. autoclass:: BatchNormLayer

Recurrent layer
------------------

All recurrent layers can implement any type of RNN cell by feeding different cell function (LSTM, GRU etc).

Fixed Length RNN layer
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: RNNLayer

Dynamic RNN layer
^^^^^^^^^^^^^^^^^^^^^^^^^^
.. autoclass:: DynamicRNNLayer

Shape layer
------------

Flatten layer
^^^^^^^^^^^^^^^

.. autoclass:: FlattenLayer

Concat layer
^^^^^^^^^^^^^^

.. autoclass:: ConcatLayer

Reshape layer
^^^^^^^^^^^^^^^

.. autoclass:: ReshapeLayer

Merge TF-Slim
---------------

Yes ! TF-Slim models can be merged into TensorLayer, all Google's Pre-trained model can be used easily ,
see `Slim-model <https://github.com/tensorflow/models/tree/master/slim#Install>`_ .

.. autoclass:: SlimNetsLayer

Flow control layer
----------------------

.. autoclass:: MultiplexerLayer

Wrapper
---------

Embedding + Attention + Seq2seq
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: EmbeddingAttentionSeq2seqWrapper
  :members:



Helper functions
------------------------

.. autofunction:: flatten_reshape
.. autofunction:: clear_layers_name
.. autofunction:: set_name_reuse
.. autofunction:: print_all_variables
.. autofunction:: initialize_rnn_state
