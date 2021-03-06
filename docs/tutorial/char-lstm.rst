Generating Random Sentence with LSTM RNN
========================================

This tutorial shows how to train a LSTM (Long short-term memory) RNN (recurrent
neural network) to perform character-level sequence training and prediction. The
original model, usually called ``char-rnn`` is described in `Andrej Karpathy's
blog <http://karpathy.github.io/2015/05/21/rnn-effectiveness/>`_, with
a reference implementation in Torch available `here
<https://github.com/karpathy/char-rnn>`_.

Because MXNet.jl does not have a specialized model for recurrent neural networks
yet, the example shown here is an implementation of LSTM by using the default
:class:`FeedForward` model via explicitly unfolding over time. We will be using
fixed-length input sequence for training. The code is adapted from the `char-rnn
example for MXNet's Python binding
<https://github.com/dmlc/mxnet/blob/master/example/rnn/char_lstm.ipynb>`_, which
demonstrates how to use low-level :doc:`symbolic APIs </api/symbolic-node>` to
build customized neural network models directly.

The most important code snippets of this example is shown and explained here.
To see and run the complete code, please refer to the `examples/char-lstm
<https://github.com/dmlc/MXNet.jl/tree/master/examples/char-lstm>`_ directory.
You will need to install `Iterators.jl
<https://github.com/JuliaLang/Iterators.jl>`_ and `StatsBase.jl
<https://github.com/JuliaStats/StatsBase.jl>`_ to run this example.

LSTM Cells
----------

Christopher Olah has a `great blog post about LSTM
<http://colah.github.io/posts/2015-08-Understanding-LSTMs/>`_ with beautiful and
clear illustrations. So we will not repeat the definition and explanation of
what an LSTM cell is here. Basically, an LSTM cell takes input ``x``, as well as
previous states (including ``c`` and ``h``), and produce the next states.
We define a helper type to bundle the two state variables together:

.. literalinclude:: ../../examples/char-lstm/lstm.jl
   :language: julia
   :start-after: #--LSTMState
   :end-before: #--/LSTMState

Because LSTM weights are shared at every time when we do explicit unfolding, so
we also define a helper type to hold all the weights (and bias) for an LSTM cell
for convenience.

.. literalinclude:: ../../examples/char-lstm/lstm.jl
   :language: julia
   :start-after: #--LSTMParam
   :end-before: #--/LSTMParam

Note all the variables are of type :class:`SymbolicNode`. We will construct the
LSTM network as a symbolic computation graph, which is then instantiated with
:class:`NDArray` for actual computation.

.. literalinclude:: ../../examples/char-lstm/lstm.jl
   :language: julia
   :start-after: #--lstm_cell
   :end-before: #--/lstm_cell

The following figure is stolen (permission requested) from
`Christopher Olah's blog
<http://colah.github.io/posts/2015-08-Understanding-LSTMs/>`_, which illustrate
exactly what the code snippet above is doing.

.. image:: images/LSTM3-chain.png

In particular, instead of defining the four gates independently, we do the
computation together and then use :class:`SliceChannel` to split them into four
outputs. The computation of gates are all done with the symbolic API. The return
value is a LSTM state containing the output of a LSTM cell.

Unfolding LSTM
--------------
Using the LSTM cell defined above, we are now ready to define a function to
unfold a LSTM network with L layers and T time steps. The first part of the
function is just defining all the symbolic variables for the shared weights and
states.

The ``embed_W`` is the weights used for character embedding --- i.e. mapping the
one-hot encoded characters into real vectors. The ``pred_W`` and ``pred_b`` are
weights and bias for the final prediction at each time step.

Then we define the weights for each LSTM cell. Note there is one cell for each
layer, and it will be replicated (unrolled) over time. The states are, however,
*not* shared over time. Instead, here we define the initial states here at the
beginning of a sequence, and we will update them with the output states at each
time step as we explicitly unroll the LSTM.

.. literalinclude:: ../../examples/char-lstm/lstm.jl
   :language: julia
   :start-after: #--LSTM-part1
   :end-before: #--/LSTM-part1

Unrolling over time is a straightforward procedure of stacking the embedding
layer, and then LSTM cells, on top of which the prediction layer. During
unrolling, we update the states and collect all the outputs. Note each time step
takes data and label as inputs. If the LSTM is named as ``:ptb``, the data and
label at step ``t`` will be named ``:ptb_data_$t`` and ``:ptb_label_$t``. Late
on when we prepare the data, we will define the data provider to match those
names.

.. literalinclude:: ../../examples/char-lstm/lstm.jl
   :language: julia
   :start-after: #--LSTM-part2
   :end-before: #--/LSTM-part2

Note at each time step, the prediction is connected to a :class:`SoftmaxOutput`
operator, which could back propagate when corresponding labels are provided. The
states are then connected to the next time step, which allows back propagate
through time. However, at the end of the sequence, the final states are not
connected to anything. This dangling outputs is problematic, so we explicitly
connect each of them to a :class:`BlockGrad` operator, which simply back
propagates 0-gradient and closes the computation graph.

In the end, we just group all the prediction outputs at each time step as
a single :class:`SymbolicNode` and return. Optionally we will also group the
final states, this is used when we use the trained LSTM to sample sentences.

.. literalinclude:: ../../examples/char-lstm/lstm.jl
   :language: julia
   :start-after: #--LSTM-part3
   :end-before: #--/LSTM-part3

Data Provider for Text Sequences
--------------------------------

Now we need to construct a data provider that takes a text file, divide the text
into mini-batches of fixed-length character-sequences, and provide them as
one-hot encoded vectors.

Note the is no fancy feature extraction at all. Each character is simply encoded
as a one-hot vector: a 0-1 vector of the size given by the vocabulary. Here we
just construct the vocabulary by collecting all the unique characters in the
training text -- there are not too many of them (including punctuations and
whitespace) for English text. Each input character is then encoded as a vector
of 0s on all coordinates, and 1 on the coordinate corresponding to that
character. The character-to-coordinate mapping is giving by the vocabulary.

The text sequence data provider implement the :doc:`data provider API
</api/io>`. We define the ``CharSeqProvider`` as below:

.. literalinclude:: ../../examples/char-lstm/seq-data.jl
   :language: julia
   :start-after: #--CharSeqProvider
   :end-before: #--/CharSeqProvider

The provided data and labels follow the naming convention of inputs used when
unrolling the LSTM. Note in the code below, apart from ``$name_data_$t`` and
``$name_label_$t``, we also provides the initial ``c`` and ``h`` states for each
layer. This is because we are using the high-level :class:`FeedForward` API,
which has no idea about time and states. So we will feed the initial states for
each sequence from the data provider. Since the initial states is always zero,
we just need to always provide constant zero blobs.

.. literalinclude:: ../../examples/char-lstm/seq-data.jl
   :language: julia
   :start-after: #--provide
   :end-before: #--/provide

Next we implement the :func:`AbstractDataProvider.eachbatch` interface for the provider.
We start by defining the data and label arrays, and the ``DataBatch`` object we
will provide in each iteration.

.. literalinclude:: ../../examples/char-lstm/seq-data.jl
   :language: julia
   :start-after: #--eachbatch-part1
   :end-before: #--/eachbatch-part1

The actual data providing iteration is implemented as a Julia **coroutine**. In this
way, we can write the data loading logic as a simple coherent ``for`` loop, and
do not need to implement the interface functions like :func:`Base.start`,
:func:`Base.next`, etc.

Basically, we partition the text into
batches, each batch containing several contiguous text sequences. Note at each
time step, the LSTM is trained to predict the next character, so the label is
the same as the data, but shifted ahead by one index.

.. literalinclude:: ../../examples/char-lstm/seq-data.jl
   :language: julia
   :start-after: #--eachbatch-part2
   :end-before: #--/eachbatch-part2


Training the LSTM
-----------------

Now we have implemented all the supporting infrastructures for our char-lstm.
To train the model, we just follow the standard high-level API. Firstly, we
construct a LSTM symbolic architecture:

.. literalinclude:: ../../examples/char-lstm/train.jl
   :language: julia
   :start-after: #--LSTM
   :end-before: #--/LSTM

Note all the parameters are defined in `examples/char-lstm/config.jl
<https://github.com/dmlc/MXNet.jl/blob/master/examples/char-lstm/config.jl>`_.
Now we load the text file and define the data provider. The data ``input.txt``
we used in this example is `a tiny Shakespeare dataset
<https://github.com/dmlc/web-data/tree/master/mxnet/tinyshakespeare>`_. But you
can try with other text files.

.. literalinclude:: ../../examples/char-lstm/train.jl
   :language: julia
   :start-after: #--data
   :end-before: #--/data

The last step is to construct a model, an optimizer and fit the mode to the
data. We are using the :class:`ADAM` optimizer [Adam]_ in this example.

.. literalinclude:: ../../examples/char-lstm/train.jl
   :language: julia
   :start-after: #--train
   :end-before: #--/train

Note we are also using a customized ``NLL`` evaluation metric, which calculate
the negative log-likelihood during training. Here is an output sample at the end of
the training process.

.. code-block:: text

   ...
   INFO: Speed: 357.72 samples/sec
   INFO: == Epoch 020 ==========
   INFO: ## Training summary
   INFO:                NLL = 1.4672
   INFO:         perplexity = 4.3373
   INFO:               time = 87.2631 seconds
   INFO: ## Validation summary
   INFO:                NLL = 1.6374
   INFO:         perplexity = 5.1418
   INFO: Saved checkpoint to 'char-lstm/checkpoints/ptb-0020.params'
   INFO: Speed: 368.74 samples/sec
   INFO: Speed: 361.04 samples/sec
   INFO: Speed: 360.02 samples/sec
   INFO: Speed: 362.34 samples/sec
   INFO: Speed: 360.80 samples/sec
   INFO: Speed: 362.77 samples/sec
   INFO: Speed: 357.18 samples/sec
   INFO: Speed: 355.30 samples/sec
   INFO: Speed: 362.33 samples/sec
   INFO: Speed: 359.23 samples/sec
   INFO: Speed: 358.09 samples/sec
   INFO: Speed: 356.89 samples/sec
   INFO: Speed: 371.91 samples/sec
   INFO: Speed: 372.24 samples/sec
   INFO: Speed: 356.59 samples/sec
   INFO: Speed: 356.64 samples/sec
   INFO: Speed: 360.24 samples/sec
   INFO: Speed: 360.32 samples/sec
   INFO: Speed: 362.38 samples/sec
   INFO: == Epoch 021 ==========
   INFO: ## Training summary
   INFO:                NLL = 1.4655
   INFO:         perplexity = 4.3297
   INFO:               time = 86.9243 seconds
   INFO: ## Validation summary
   INFO:                NLL = 1.6366
   INFO:         perplexity = 5.1378
   INFO: Saved checkpoint to 'examples/char-lstm/checkpoints/ptb-0021.params'


.. [Adam] Diederik Kingma and Jimmy Ba: *Adam: A Method for Stochastic
          Optimization*. `arXiv:1412.6980 <http://arxiv.org/abs/1412.6980>`_
          [cs.LG].


Sampling Random Sentences
-------------------------

After training the LSTM, we can now sample random sentences from the trained
model. The sampler works in the following way:

- Starting from some fixed character, take ``a`` for example, and feed it as input to the LSTM.
- The LSTM will produce an output distribution over the vocabulary and a state
  in the first time step. We sample a character from the output distribution,
  fix it as the second character.
- In the next time step, we feed the previously sampled character as input and
  continue running the LSTM by also taking the previous states (instead of the
  0 initial states).
- Continue running until we sampled enough characters.

Note we are running with mini-batches, so several sentences could be sampled
simultaneously. Here are some sampled outputs from a network I trained for
around half an hour on the Shakespeare dataset. Note all the line-breaks,
punctuations and upper-lower case letters are produced by the sampler itself.
I did not do any post-processing.

.. code-block:: text

   ## Sample 1
   all have sir,
   Away will fill'd in His time, I'll keep her, do not madam, if they here? Some more ha?

   ## Sample 2
   am.

   CLAUDIO:
   Hone here, let her, the remedge, and I know not slept a likely, thou some soully free?

   ## Sample 3
   arrel which noble thing
   The exchnachsureding worns: I ne'er drunken Biancas, fairer, than the lawfu?

   ## Sample 4
   augh assalu, you'ld tell me corn;
   Farew. First, for me of a loved. Has thereat I knock you presents?

   ## Sample 5
   ame the first answer.

   MARIZARINIO:
   Door of Angelo as her lord, shrield liken Here fellow the fool ?

   ## Sample 6
   ad well.

   CLAUDIO:
   Soon him a fellows here; for her fine edge in a bogms' lord's wife.

   LUCENTIO:
   I?

   ## Sample 7
   adrezilian measure.

   LUCENTIO:
   So, help'd you hath nes have a than dream's corn, beautio, I perchas?

   ## Sample 8
   as eatter me;
   The girlly: and no other conciolation!

   BISTRUMIO:
   I have be rest girl. O, that I a h?

   ## Sample 9
   and is intend you sort:
   What held her all 'clama's for maffice. Some servant.' what I say me the cu?

   ## Sample 10
   an thoughts will said in our pleasue,
   Not scanin on him that you live; believaries she.

   ISABELLLLL?

See `Andrej Karpathy's blog post
<http://karpathy.github.io/2015/05/21/rnn-effectiveness/>`_ on more examples and
links including Linux source codes, Algebraic Geometry Theorems, and even
cooking recipes. The code for sampling can be found in
`examples/char-lstm/sampler.jl
<https://github.com/dmlc/MXNet.jl/blob/master/examples/char-lstm/sampler.jl>`_.

Visualizing the LSTM
--------------------

Finally, you could visualize the LSTM by calling :func:`to_graphviz` on the
constructed LSTM symbolic architecture. We only show an example of 1-layer and
2-time-step LSTM below. The automatic layout produced by GraphViz is definitely
much less clear than `Christopher Olah's illustrations
<http://colah.github.io/posts/2015-08-Understanding-LSTMs/>`_, but could
otherwise be very useful for debugging. As we can see, the LSTM unfolded over
time is just a (very) deep neural network. The complete code for producing this
visualization can be found in `examples/char-lstm/visualize.jl
<https://github.com/dmlc/MXNet.jl/blob/master/examples/char-lstm/visualize.jl>`_.

.. image:: images/char-lstm-vis.svg
